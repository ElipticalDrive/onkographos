# NeRF Editing Procedure

Also called: Feature Field Distillation Editing, DFF-based Editing

## Overview
This method enables direct 3D editing of a trained NeRF by decoupling semantic or geometric queries from the radiance field representation using a Distilled Feature Field (DFF) [S1]. Unlike methods that rely on iterative optimization loops to minimize a loss against edited images, this approach modifies the rendering pipeline itself: it uses a pre-trained teacher network to determine whether a 3D point belongs to a target object, allowing for immediate application of transforms (deletion, colorization, translation) during ray marching [S1]. It is distinct from optimization-based inpainting [12372-nerf-in-inpainting-optimization.md] because it does not require re-training the NeRF weights for every edit, and from local editing via depth propagation [8658-progressive-inpainting-and-erasing.md] because it operates in 3D space rather than projecting 2D masks across views.

## How it works
**Inputs:**
1.  A trained NeRF $f_\theta$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ [S1].
2.  A Distilled Feature Field (DFF) $f_{\phi}$, which is a lightweight network trained to predict the probability $p$ that a 3D coordinate $\mathbf{x}$ matches a set of user-defined queries (e.g., semantic classes or object instances) [S1].
3.  A user-defined threshold $\tau$ for selection [S1].
4.  An editing transformation $g$ (e.g., translation vector, colorization function $b$) [S1].

**Outputs:**
*   Edited rendered images $\hat{C}_{edit}(\mathbf{r})$ for given camera rays [S1].

**Data Structures:**
*   **NeRF $f_\theta$:** Standard MLP or grid-based representation. Inputs: 3D coordinate $\mathbf{x} \in \mathbb{R}^3$, viewing direction $\mathbf{d} \in \mathbb{R}^3$. Outputs: Density $\sigma \in \mathbb{R}$, Color $\mathbf{c} \in \mathbb{R}^3$ [S1].
*   **DFF $f_{\phi}$:** Lightweight network. Input: 3D coordinate $\mathbf{x} \in \mathbb{R}^3$. Output: Probability $p \in [0, 1]$ indicating the likelihood that $\mathbf{x}$ belongs to the queried object [S1].

**Steps:**
1.  **Ray Sampling:** For each camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, sample $N$ points $\{t_1, \dots, t_N\}$ along the ray, yielding 3D coordinates $\{\mathbf{x}_1, \dots, \mathbf{x}_N\}$ [S1].
2.  **Query Probability Calculation:** For each point $\mathbf{x}_i$, query the DFF to obtain the probability $p_i = f_{\phi}(\mathbf{x}_i)$ that the coordinate matches the user's query [S1].
3.  **Selection Decision:** Define a binary selection mask $m_i$:
    $$
    m_i = \begin{cases} 
    1 & \text{if } p_i \ge \tau \\
    0 & \text{otherwise}
    \end{cases}
    $$
    Alternatively, the sources note that one can use the mixture of selected and not-selected results in the proportion of $p$ without a hard threshold [S1].
4.  **Point Processing:** For each point $\mathbf{x}_i$, determine the effective density $\sigma'_i$ and color $\mathbf{c}'_i$ based on $m_i$:
    *   **If not selected ($m_i = 0$):**
        *   Query the vanilla NeRF: $(\sigma_i, \mathbf{c}_i) = f_\theta(\mathbf{x}_i, \mathbf{d})$ [S1].
        *   Set $\sigma'_i = \sigma_i, \mathbf{c}'_i = \mathbf{c}_i$ [S1].
    *   **If selected ($m_i = 1$):** Apply the specific editing transform:
        *   **Deletion:** Set $\sigma'_i = 0$. The color $\mathbf{c}'_i$ is irrelevant as density is zero [S1].
        *   **Color Editing:** Query the NeRF for $(\sigma_i, \mathbf{c}_i) = f_\theta(\mathbf{x}_i, \mathbf{d})$. Apply the colorization function $b$ to the color: $\mathbf{c}'_i = b(\mathbf{c}_i)$. Keep $\sigma'_i = \sigma_i$ [S1].
        *   **Translation / Rescaling:**
            1.  Compute the inverse transformation $\mathbf{x}'_i = g^{-1}(\mathbf{x}_i)$ [S1].
            2.  Query the DFF for $\mathbf{x}'_i$ to get probability $p'_i$.
            3.  If $p'_i \ge \tau$ (i.e., the deformed point is also part of the object):
                *   Query the NeRF for the original point: $(\sigma_i, \mathbf{c}_i) = f_\theta(\mathbf{x}_i, \mathbf{d})$ [S1].
                *   Query the NeRF for the deformed point: $(\sigma'_i, \mathbf{c}'_i) = f_\theta(\mathbf{x}'_i, \mathbf{d})$ [S1].
                *   Mix the colors based on their alphas. Let $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $\alpha'_i = 1 - \exp(-\sigma'_i \delta_i)$. The mixed color is:
                    $$
                    \mathbf{c}_{mixed} = \frac{\alpha_i \mathbf{c}_i + \alpha'_i \mathbf{c}'_i}{\alpha_i + \alpha'_i}
                    $$
                    (Note: The source states "mix their colors $c(x)$ and $c(x_0)$ in the ratio of their alphas at the point for simplicity" [S1]. The exact weighting formula is not explicitly defined beyond "ratio of their alphas", but standard alpha compositing or weighted average is implied. For implementation, a weighted average by alpha is the standard interpretation of "ratio of alphas" in this context [G]).
                *   Set $\sigma_{final} = \sigma_i + \sigma'_i$ (or similar accumulation depending on how the boundary is handled, but the source emphasizes color mixing) [S1].
            4.  If $p'_i < \tau$ (deformed point is not in object):
                *   Use the original point's properties: $\sigma'_i = \sigma_i, \mathbf{c}'_i = \mathbf{c}_i$ [S1].
5.  **Volume Rendering:** Perform standard volume rendering using the modified tuples $(\sigma'_i, \mathbf{c}'_i)$:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T(t_i) (1 - \exp(-\sigma'_i \delta_i)) \mathbf{c}'_i
    $$
    where $T(t_i) = \exp\left(-\sum_{j=1}^{i-1} \sigma'_j \delta_j\right)$ [S1].

**Parameters:**
*   $\tau$: User-defined threshold for DFF probability. Typical values are not specified in the sources, but it acts as a binary switch [S1].
*   $b$: Colorization function. Implementation depends on the specific edit (e.g., a learned mapping or a simple color space transformation) [S1].
*   $g$: Geometric transformation. For translation, $g(\mathbf{x}) = \mathbf{x} + \mathbf{v}$, so $g^{-1}(\mathbf{x}) = \mathbf{x} - \mathbf{v}$ [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Mixing:** When both $\mathbf{x}$ and $\mathbf{x}'$ are selected (e.g., at the boundary of a translated object), the color mixing logic must handle the case where both densities are non-zero. The source suggests mixing in the ratio of their alphas [S1].
*   **DFF Accuracy:** The quality of editing depends heavily on the DFF's ability to accurately segment the queried object in 3D. Errors in DFF will lead to incorrect selection masks [S1].
*   **Threshold Sensitivity:** The choice of $\tau$ affects the smoothness of the edit. A hard threshold may cause aliasing, while a soft mixture (using $p$ directly) may blur edges [S1].

## Subtopics

### Local Editing via SAM and Depth Propagation
A technique to edit specific objects within a NeRF without altering the background, using 2D segmentation masks propagated across views via depth information [S2]. It employs the Segment Anything Model (SAM) to derive masks for each image, allowing users to isolate a target instance with a single point [S2]. The identified mask is then projected across different views using depth information, continuing as long as the projected mask sufficiently overlaps with the corresponding mask in the current view [S2]. This integrated mask is used to restrict editing to the foreground object, leaving the background intact [S2]. This method is chosen when precise 2D segmentation is available and multi-view consistency can be enforced via depth, but it may fail if depth estimation is incorrect or if the object has glossy surfaces that confuse depth [S2].

### ViCA-NeRF Warm-Up
A strategy to efficiently warm up the editing process by blending edits directly before the entire editing process, accelerating editing while preserving consistency [S2]. It randomly selects a view to edit and projects the edited view onto all other views [S2]. The update for a view $i$ given an edited view $j$ is calculated as:
$$
I'_i = \lambda I_i \odot m_w + (1 - \lambda) I_{i,j} \odot m_w + I_i \odot (1 - m_w)
$$
where $m_w$ is the binary mask for the projected area and $\lambda$ is a hyperparameter controlling the ratio preserved from the original pixel value [S2]. This is used to quickly propagate style or content changes across views before iterative refinement [S2].

### NeRF-RPN Application
A method for scene editing using 3D object detection proposals [S3]. It edits the scene by setting the density inside a proposal bounding box to zero when rendering [S3]. This effectively removes the detected object from the scene. It is used when the goal is object removal based on 3D detection rather than semantic segmentation [S3].

## Operating characteristics
*   **Time Complexity:** The editing process is fast during rendering because it does not require gradient descent or weight updates. The cost is dominated by querying the DFF and the NeRF for each sample point [S1].
*   **Memory Complexity:** Requires storing the NeRF weights and the DFF weights. The DFF is described as "lightweight" [S1].
*   **Strengths:** Immediate editing without re-training; supports various transforms (deletion, color, translation) [S1].
*   **Limitations:** Relies on the accuracy of the DFF; hard to edit objects not well-represented in the DFF's training data; may produce artifacts at boundaries if the DFF segmentation is noisy [S1].
*   **When to use:** When real-time or near-real-time editing is required, and the objects to be edited can be segmented by a pre-trained feature field [S1].
*   **When not to use:** When the edit requires complex geometric changes that cannot be described by a simple transformation $g$, or when the DFF does not cover the semantic space of the target object [S1].

## Background
The NeRF Editing Procedure via Feature Field Distillation was introduced in "Decomposing NeRF for Editing Via Feature Field Distillation" [S1]. It builds on the idea of decomposing the NeRF into a radiance field and a feature field, allowing for editing without modifying the radiance field weights directly. The DFF is trained to distill features from a teacher network (e.g., LSeg or DINO) [S1]. This approach contrasts with optimization-based methods like Instruct-NeRF2NeRF [2] which require iterative updates [S2].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S3] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
