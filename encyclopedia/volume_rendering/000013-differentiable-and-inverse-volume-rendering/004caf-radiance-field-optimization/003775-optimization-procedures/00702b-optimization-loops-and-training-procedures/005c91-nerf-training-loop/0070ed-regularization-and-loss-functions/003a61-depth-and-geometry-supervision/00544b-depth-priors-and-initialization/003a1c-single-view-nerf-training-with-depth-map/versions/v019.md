# Single-view NeRF Training with Depth Map

Also called: Depth-guided Single-view NeRF, Single-view NeRF with Depth Initialization

## Overview
This method addresses the under-constrained optimization problem of single-view NeRF reconstruction by initializing the geometry of visible regions using a 360-degree depth estimation [S1]. It sets itself apart from sibling methods like [Depth Supervision](14220-depth-supervision.md) by explicitly constructing a canonical panorama depth map from multiple perspective projections via deformable alignment before integrating it into the NeRF training loop, rather than applying depth loss directly to raw perspective estimates or using multi-view consistency [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A single input panorama $I_{\text{rgb}}$.
- **Output:** A trained NeRF network $f_{\text{nerf}}$ parameterized by $\theta$, representing the geometry and appearance of visible regions.

**Data Structures:**
- **Perspective Projections:** A set of $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$ and their corresponding estimated depth maps $\{\bar{I}_d^k\}_{k=1}^K$.
- **Canonical Panorama Depth Map:** A learnable depth map $I_d$ defined on the same coordinate system as $I_{\text{rgb}}$.
- **Alignment Parameters:** For each perspective view $k$, a global scale factor $w_k$ and a local bias field $\{b_{k,i,j}\}$ for each pixel $(i,j)$.

**Numbered Steps:**

1.  **Perspective Projection:** Project the single input panorama $I_{\text{rgb}}$ into $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$ using different viewing directions. The number of projections is set to $K=20$ [S1]. This is necessary because existing monocular depth estimators are trained on limited field-of-view images and do not handle 360-degree panoramas directly [S1].

2.  **Monocular Depth Estimation:** Pass each perspective image $\bar{I}_{\text{rgb}}^k$ through a pre-trained monocular depth estimation model (referenced as [52] in the source) to obtain depth maps $\{\bar{I}_d^k\}_{k=1}^K$ [S1]. Note that due to scale ambiguity in single-view depth estimation, depth values in overlapped regions between different perspectives may differ significantly [S1].

3.  **Deformable Alignment:** Align the perspective depth maps $\{\bar{I}_d^k\}$ to a canonical learnable panorama depth map $I_d$ by optimizing the alignment objective. This step resolves scale and bias inconsistencies across views [S1].
    -   For each perspective map $k$, assign a scale $w_k$ and local bias $\{b_{k,i,j}\}$.
    -   Optimize the following objective function (Eq. 2 in [S1]):
        $$ \sum_{k,i,j} \left\| \bar{v}_{i,j}^k w_k + b_{k,i,j} - v_{M(i,j)} \right\| + \lambda_s \left( L_s(b_{i,j+1}, b_{i,j}) + L_s(b_{i+1,j}, b_{i,j}) \right) $$
        where:
        -   $\bar{v}_{i,j}^k$ is the pixel value of depth map $\bar{I}_d^k$ at coordinates $(i,j)$.
        -   $v_{M(i,j)}$ is the pixel value of the canonical panorama depth map $I_d$ at the coordinate mapped from $(i,j)$ via the projective operation $M$ (mapping perspective image coordinates to panorama image coordinates).
        -   $L_s$ is the smooth L1 loss.
        -   The second term is a total variation regularization term on the bias field to ensure spatial smoothness [S1].
        -   $\lambda_s$ is a weighting factor for the regularization term (Not given in the sources).

4.  **NeRF Training:** Train the single-view NeRF using the RGB panorama $I_{\text{rgb}}$ and the aligned canonical depth map $I_d$. The NeRF is optimized to minimize the combined color and depth reconstruction loss (Eq. 3 in [S1]):
    $$ \mathcal{L}_{\text{nerf}}' = \sum_{\mathbf{r} \in \mathcal{R}} \left( \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|^2 + \left\| \hat{D}(\mathbf{r}) - \bar{D}(\mathbf{r}) \right\|^2 \right) $$
    where:
    -   $\mathcal{R}$ is the set of rays sampled from the panorama.
    -   $\hat{C}(\mathbf{r})$ is the expected pixel color rendered by the NeRF.
    -   $C(\mathbf{r})$ is the ground-truth pixel color from $I_{\text{rgb}}$.
    -   $\hat{D}(\mathbf{r})$ is the expected depth value rendered by the NeRF, computed as $\hat{D}(\mathbf{r}) = \sum_{i=1}^N w_i t_i$ (as per [Parent](15188-depth-priors-and-initialization.md)).
    -   $\bar{D}(\mathbf{r})$ is the corresponding depth pixel value from the canonical depth map $I_d$ at the ray's intersection point [S1].

**Parameters:**
-   $K$: Number of perspective projections. Set to 20 [S1].

**Edge Cases and Pitfalls:**
-   **Scale Ambiguity:** Monocular depth estimation suffers from scale ambiguity, leading to inconsistent depth values in overlapping regions of different perspective projections. The deformable alignment step is critical to resolve this [S1].
-   **Visible Regions Only:** Since only a single panorama is available, the trained NeRF $f_{\text{nerf}}$ is only accurate for visible regions. Invisible regions require separate completion methods (e.g., collaborative RGBD inpainting) [S1].

## Operating characteristics
-   **Time Complexity:** The method requires an initial depth estimation pass for $K$ views and an optimization step for deformable alignment before the main NeRF training loop. The alignment optimization involves solving for scale and bias parameters for all pixels across $K$ views.
-   **Memory Complexity:** Requires storing $K$ perspective depth maps and the canonical panorama depth map $I_d$. The alignment parameters ($w_k, b_{k,i,j}$) add memory overhead proportional to the resolution of the depth maps.
-   **Strengths:** Provides accurate geometry initialization for visible regions, addressing the ill-posed nature of single-view reconstruction. The deformable alignment effectively handles scale inconsistencies from monocular estimators.
-   **Limitations:** The method is limited to visible regions; invisible regions are not reconstructed by this step alone. Performance depends on the accuracy of the underlying monocular depth estimator [52] and the quality of the perspective projections.
-   **When to use:** When reconstructing a scene from a single panoramic image and accurate geometry of visible parts is required.
-   **When not to use:** When the scene has significant occlusions or invisible regions that need to be filled, or when multi-view inputs are available.

## Background
This method was introduced in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1]. It addresses the limitation of previous single-view NeRF methods that did not fully ensure geometry consistency. By leveraging 360-degree depth estimation and deformable alignment, it enables photo-realistic panoramic NeRF reconstruction from a single input [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
