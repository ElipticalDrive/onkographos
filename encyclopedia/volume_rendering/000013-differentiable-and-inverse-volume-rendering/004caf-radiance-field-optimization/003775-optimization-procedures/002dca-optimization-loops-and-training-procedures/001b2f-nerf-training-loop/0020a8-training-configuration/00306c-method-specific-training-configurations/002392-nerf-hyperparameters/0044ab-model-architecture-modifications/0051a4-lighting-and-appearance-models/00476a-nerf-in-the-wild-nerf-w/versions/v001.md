# NeRF in the Wild (NeRF-W)

Also called: NeRF-W

## Overview
NeRF in the Wild (NeRF-W) addresses the failure of standard NeRF models when applied to uncontrolled, real-life photographs ("in-the-wild" scenes) by handling two primary sources of variation: per-image appearance changes due to lighting and transient objects that appear in only some images [S1]. It sets itself apart from standard NeRF by decoupling the scene's static geometry from its dynamic appearance and transient occlusions, allowing for robust novel view synthesis in environments where camera poses may be inaccurate and content is inconsistent across views [S1].

## How it works
NeRF-W modifies the standard radiance field function $F_\Theta$ by introducing conditional inputs and auxiliary branches to separate static scene properties from image-specific variations [S1].

**Inputs and Outputs:**
*   **Input:** A set of images $\{I_i\}$ with associated camera poses $\{P_i\}$ (often approximate or uncalibrated).
*   **Output:** A rendered image $\hat{I}$ for a target viewpoint, decomposed into static and transient components.

**Data Structures:**
1.  **Static Density MLP ($M_\sigma$):** A standard MLP that takes positional encoding of 3D coordinates $\gamma(\mathbf{x})$ as input. It is shared across all images in the scene.
    *   Output: Volume density $\sigma$.
2.  **Static Color MLP ($M_c$):** Takes $\gamma(\mathbf{x})$ and view direction $\gamma(\mathbf{d})$ as input.
    *   Output: Static color $\mathbf{c}_{static}$.
3.  **Appearance Embedding ($\mathbf{z}_i$):** A learnable vector (latent code) specific to each image $i$.
    *   Used to condition the color prediction.
4.  **Transient MLP ($M_\tau$):** Takes $\gamma(\mathbf{x})$, $\gamma(\mathbf{d})$, and a transient embedding $\mathbf{t}_i$ (specific to image $i$) as input.
    *   Output: Transient density $\sigma_\tau$ and transient color $\mathbf{c}_\tau$.

**Numbered Steps:**
1.  **Initialize:** For each image $i$ in the training set, initialize a unique appearance embedding $\mathbf{z}_i$ and a transient embedding $\mathbf{t}_i$. Initialize the shared static density MLP $M_\sigma$ and static color MLP $M_c$. Initialize the transient MLP $M_\tau$.
2.  **Sample Rays:** For each training image $i$, cast rays through each pixel. Sample $N$ points $\mathbf{x}_n$ along each ray.
3.  **Encode Inputs:** Apply positional encoding $\gamma$ to positions $\mathbf{x}_n$ and directions $\mathbf{d}$.
4.  **Predict Static Properties:**
    *   Compute static density: $\sigma_n = M_\sigma(\gamma(\mathbf{x}_n))$.
    *   Compute static color: $\mathbf{c}_{n, static} = M_c(\gamma(\mathbf{x}_n), \gamma(\mathbf{d}), \mathbf{z}_i)$. Note: The appearance embedding $\mathbf{z}_i$ is concatenated or used as a condition in $M_c$ [S1].
5.  **Predict Transient Properties:**
    *   Compute transient density: $\sigma_{n, \tau} = M_\tau(\gamma(\mathbf{x}_n), \gamma(\mathbf{d}), \mathbf{t}_i)$.
    *   Compute transient color: $\mathbf{c}_{n, \tau} = M_\tau(\gamma(\mathbf{x}_n), \gamma(\mathbf{d}), \mathbf{t}_i)$ (or a separate head if implemented, but NeRF-W uses one MLP for both transient density and color conditioned on $\mathbf{t}_i$) [S1].
6.  **Composite:** Render the final pixel color $\hat{C}_i(r)$ using the standard volume rendering integral, summing contributions from both static and transient components:
    $$
    \hat{C}_i(r) = \sum_{n=1}^{N} T_{n, static} \alpha_{n, static} \mathbf{c}_{n, static} + \sum_{n=1}^{N} T_{n, total} \alpha_{n, \tau} \mathbf{c}_{n, \tau}
    $$
    Where:
    *   $\alpha_{n, static} = (1 - \exp(-\sigma_n \delta_n))$
    *   $T_{n, static} = \exp(-\sum_{j=1}^{n-1} \sigma_j \delta_j)$
    *   $\alpha_{n, \tau} = (1 - \exp(-\sigma_{n, \tau} \delta_n))$
    *   $T_{n, total} = \exp(-\sum_{j=1}^{n-1} (\sigma_j + \sigma_{j, \tau}) \delta_j)$
    *   $\delta_n$ is the distance between sample points.
7.  **Optimize:** Minimize the reconstruction loss (e.g., MSE) between $\hat{C}_i(r)$ and the ground truth image $I_i$. Update weights of $M_\sigma, M_c, M_\tau$ and embeddings $\mathbf{z}_i, \mathbf{t}_i$.

**Parameters:**
*   Number of samples per ray $N$: Not given in the sources.
*   Embedding dimension: Not given in the sources.
*   MLP hidden layers/units: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Transient Overfitting:** If the transient MLP is too powerful, it may absorb static scene details, leading to artifacts. The separation of static and transient branches helps mitigate this [S1].
*   **Pose Sensitivity:** NeRF-W assumes approximate poses are available. If poses are severely incorrect, the model may struggle to distinguish between transient objects and geometric inconsistencies [S1].

## Subtopics

### Transient Filtering via Candidate Head
This variant, proposed in UP-NeRF, isolates the transient network to improve robustness in pose optimization [S2, unofficial]. It uses a candidate head to predict transient properties (color, density, beta) alongside static properties, enabling robust pose optimization by filtering out transient occluders [S2, unofficial]. It differs from NeRF-W by isolating the transient network and using deep features (DINO) for candidate filtering, whereas NeRF-W does not isolate the transient network and relies on different architecture assumptions [S2, unofficial].

**Steps:**
1.  Predict transient color $c_i^{(\tau)}$, transient sigma $\sigma_i^{(\tau)}$, and beta $\beta_i$ using predicted candidate feature $f_i$ and candidate embedding $\ell_i^{(c)}$ [S2, unofficial].
2.  Render pixel color $\hat{C}_i(r)$ using static and transient parameters ($c_i, c_i^{(\tau)}, \sigma_i, \sigma_i^{(\tau)}$) following the NeRF-W compositing scheme [S2, unofficial].

**Pitfalls:**
*   Merging transient and candidate heads degrades performance: Separating the transient head is necessary; failing to separate it causes the model to fail in estimating correct poses and reduces rendering quality [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time increases compared to standard NeRF due to the additional MLPs and learnable embeddings per image. Inference time is similar to standard NeRF, with a slight overhead from the transient MLP evaluation.
*   **Memory Complexity:** Memory usage increases linearly with the number of images due to the storage of per-image appearance and transient embeddings.
*   **Strengths:** Handles lighting variations and transient objects effectively; robust to approximate camera poses.
*   **Limitations:** Requires per-image embeddings, which can be memory-intensive for large datasets; performance depends on the quality of initial camera poses.
*   **When to use:** Uncontrolled "in-the-wild" scenes with varying lighting and transient objects.
*   **When not to use:** Controlled studio environments with consistent lighting and no transient objects, where standard NeRF is sufficient and more efficient.

## Background
NeRF in the Wild (NeRF-W) was introduced by Brualla et al. to address the limitations of early NeRF models when applied to real-life photographs [S1]. It builds on the standard NeRF framework by conditioning the color MLP on per-image appearance embeddings and using a separate MLP for transient objects [S1]. The method is particularly relevant for applications involving large-scale, unstructured image collections, such as phototourism and urban reconstruction.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
