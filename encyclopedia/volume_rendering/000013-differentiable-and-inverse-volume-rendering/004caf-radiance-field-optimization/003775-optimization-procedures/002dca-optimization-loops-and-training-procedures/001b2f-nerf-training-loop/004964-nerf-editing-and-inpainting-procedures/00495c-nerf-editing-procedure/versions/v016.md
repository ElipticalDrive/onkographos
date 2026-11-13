# NeRF Editing Procedure

Also called: Feature Field Distillation Editing, DFF-based Editing

## Overview
The NeRF Editing Procedure enables direct manipulation of a trained Neural Radiance Field (NeRF) by integrating external semantic or geometric guidance into the volume rendering process, rather than retraining the network from scratch. It solves the problem of applying 3D-consistent edits (such as deletion, colorization, or translation) to a continuous radiance field using 2D or sparse 3D inputs. It differs from iterative optimization methods like [Instruct-NeRF2NeRF](12372-nerf-in-inpainting-optimization.md) by modifying the rendering query logic directly via a distillation feature field, allowing for immediate, differentiable editing without gradient descent over the NeRF weights.

## How it works
The method operates by querying a distillation feature field (DFF) alongside the standard NeRF to determine if a 3D point falls within an edited region.

**Inputs:**
*   Trained NeRF $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$.
*   Distillation Feature Field (DFF) $f_\phi: \mathbf{x} \to \mathbf{z}$, where $\mathbf{z}$ encodes semantic or geometric features distilled from teacher networks (e.g., LSeg, DINO) [S1].
*   Editing query $Q$ (e.g., text prompt, segmentation mask, or transformation parameters).
*   Camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$.

**Outputs:**
*   Edited pixel color $\hat{C}_{edit}(\mathbf{r})$.

**Data Structures:**
*   **DFF:** A neural network (typically a small MLP or grid) mapping 3D coordinates $\mathbf{x}$ to a feature vector $\mathbf{z}$. The output is processed to yield a selection probability $p \in [0, 1]$ [S1].
*   **Selection Probability $p$:** Calculated by comparing the DFF feature $\mathbf{z}$ against the query $Q$. If $p$ exceeds a user-defined threshold $\tau$, the point is "selected" [S1].

**Numbered Steps:**

1.  **Ray Sampling:** Sample $N$ points $\{\mathbf{x}_1, \dots, \mathbf{x}_N\}$ along the camera ray $\mathbf{r}$ at depths $\{t_1, \dots, t_N\}$, with spacing $\delta_i = t_{i+1} - t_i$ [S1].
2.  **Query DFF:** For each point $\mathbf{x}_i$, query the DFF to obtain feature $\mathbf{z}_i$. Compute the probability $p_i$ that $\mathbf{x}_i$ matches the editing query $Q$ [S1].
3.  **Determine Selection:** Define the selection status $s_i$:
    $$
    s_i = \begin{cases} 
    1 & \text{if } p_i \ge \tau \\
    0 & \text{otherwise}
    \end{cases}
    $$
    Alternatively, use the soft probability $p_i$ directly as a weight [S1].
4.  **Compute Density and Color:**
    *   **If $s_i = 0$ (Not Selected):** Query the vanilla NeRF to get density $\sigma_i = \sigma(\mathbf{x}_i)$ and color $\mathbf{c}_i = \mathbf{c}(\mathbf{x}_i)$ [S1].
    *   **If $s_i = 1$ (Selected):** Apply the specific editing transform:
        *   **Deletion:** Set density $\sigma_i = 0$. Color $\mathbf{c}_i$ is irrelevant as the point becomes transparent [S1].
        *   **Color Editing:** Query NeRF for $\sigma_i, \mathbf{c}_i$. Transform color using a colorization function $b(\cdot)$: $\mathbf{c}_i' = b(\mathbf{c}_i)$ [S1].
        *   **Translation/Rescaling:** Compute the inverse transformation $\mathbf{x}_0 = g^{-1}(\mathbf{x}_i)$, where $g$ is the editing transformation (e.g., translation vector addition) [S1].
            *   Query DFF at $\mathbf{x}_0$ to get $p_0$. If $\mathbf{x}_0$ is selected ($p_0 \ge \tau$), use $\mathbf{x}_0$ to query NeRF for $\sigma_0, \mathbf{c}_0$ [S1].
            *   If both $\mathbf{x}_i$ and $\mathbf{x}_0$ are selected and have non-zero density (e.g., at boundaries), mix colors based on alpha ratios:
                $$
                \mathbf{c}_i' = \frac{\alpha_i \mathbf{c}_i + \alpha_0 \mathbf{c}_0}{\alpha_i + \alpha_0}
                $$
                where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $\alpha_0 = 1 - \exp(-\sigma_0 \delta_i)$ [S1].
            *   If only one is selected, use its color. If neither, fall back to vanilla NeRF [S1].
5.  **Volume Rendering:** Render the ray using the modified tuples $(\sigma_i, \mathbf{c}_i')$:
    $$
    \hat{C}_{edit}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i'
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].

**Parameters:**
*   $\tau$: Selection threshold. Not given in the sources. [G: Typical values range from 0.5 to 0.9 depending on DFF confidence.]
*   $b(\cdot)$: Colorization function. Not given in the sources. [G: Often a learned MLP or a fixed transformation like HSL shift.]

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** When translating objects, the mixing of colors at the boundary between the original and deformed position can cause artifacts if the density estimates are noisy [S1].
*   **DFF Accuracy:** The quality of editing depends heavily on the DFF's ability to accurately segment the target object. Misalignment leads to incomplete edits or background corruption [S1].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive masks for each image, where users pinpoint a target instance with a single point [S2]. The identified mask is projected across different views using depth information, continuing as long as the projected mask sufficiently overlaps with the corresponding mask in the current view [S2]. This integrated mask is then used to restrict editing to the target object, preventing simultaneous changes to the background [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views [S2]. The update for a view $i$ given an edited view $j$ is calculated as:
$$
I_i' = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w)
$$
where $m_w$ is the binary mask for the projected area and $\lambda$ is a hyperparameter controlling the ratio preserved from the original pixel value [S2].

### NeRF-RPN Application
A method for scene editing using 3D object detection proposals [S3]. It edits the scene by setting the density inside a proposal bounding box to zero when rendering, effectively removing the detected object [S3].

## Operating characteristics
*   **Time Complexity:** Rendering time is comparable to standard NeRF rendering, with a small constant overhead for querying the DFF and computing selection probabilities [S1].
*   **Memory Complexity:** Requires storing the DFF parameters in addition to the NeRF weights [S1].
*   **Strengths:** Immediate editing without iterative optimization; supports various edit types (deletion, color, geometry) via a unified framework [S1].
*   **Limitations:** Relies on the accuracy of the distillation feature field; may struggle with complex semantic queries if the DFF is not well-trained [S1].
*   **When to use:** When real-time or near-real-time editing is required, or when iterative optimization is too slow [S1].
*   **When not to use:** When the editing requires significant structural changes that the DFF cannot capture, or when high-fidelity semantic consistency is critical and the DFF is unavailable [S1].

## Background
The NeRF Editing Procedure was introduced in "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It builds on the idea of decomposing the NeRF into a base radiance field and a distillation feature field, allowing for editing via feature manipulation rather than weight updates. The method leverages pre-trained teacher networks (LSeg, DINO) to distill semantic information into the DFF [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
