# Single-view NeRF Training with Depth Map

Also called: Depth-guided Single-view NeRF, Single-view NeRF with Depth Initialization

## Overview
This method addresses the under-constrained optimization problem of single-view NeRF reconstruction by initializing the geometry of visible regions using a 360-degree depth estimation prior [S1]. It sets itself apart from sibling methods by explicitly fusing multiple perspective depth estimates into a canonical panorama depth map via deformable alignment before integrating it into the NeRF training loop, rather than relying solely on implicit geometric regularization or multi-view consistency [S1].

## How it works
**Inputs and Outputs:**
- Input: A single input RGB panorama $I_{\text{rgb}}$.
- Output: A trained NeRF model $f_{\text{nerf}}$ parameterized by $\theta$, valid for visible regions.

**Data Structures:**
- $I_{\text{rgb}}$: The input panoramic image.
- $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$: Set of $K$ perspective projection images derived from $I_{\text{rgb}}$.
- $\{\bar{I}_d^k\}_{k=1}^K$: Set of estimated depth maps for each perspective image.
- $I_d$: The canonical learnable panorama depth map.
- $w_k$: Global scale factor for the $k$-th perspective depth map.
- $\{b_{k,i,j}\}$: Local bias field for the $k$-th perspective depth map, where $i,j$ are pixel coordinates.
- $M$: A projective mapping function that transforms perspective image coordinates to panorama image coordinates.

**Numbered Steps:**

1.  **Perspective Projection:** Project the single input panorama $I_{\text{rgb}}$ into $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$ using different viewing directions. The number of projections is set to $K=20$ [S1]. This is necessary because existing monocular depth estimators are trained on limited field-of-view images and do not handle 360-degree panoramas directly [S1].

2.  **Depth Estimation:** Use a monocular depth estimation model [52] to estimate depth values for each perspective image $\bar{I}_{\text{rgb}}^k$, obtaining the set of depth maps $\{\bar{I}_d^k\}_{k=1}^K$ [S1]. Note that due to scale ambiguity in single-view depth estimation, estimated depth values in overlapped regions may differ significantly [S1].

3.  **Deformable Alignment:** Perform deformable alignment to jointly align the perspective depth maps $\{\bar{I}_d^k\}$ to a canonical learnable panorama depth map $I_d$. This step optimizes global scale factors $w_k$ and local bias fields $\{b_{k,i,j}\}$ for each perspective map [S1]. The optimization objective is:
    $$ \sum_{k,i,j} \left\| \bar{v}_{i,j}^k w_k + b_{k,i,j} - v_{M(i,j)} \right\| + \mathcal{L}_s(b_{i,j+1}, b_{i,j}) + \mathcal{L}_s(b_{i+1,j}, b_{i,j}) $$
    where:
    - $\bar{v}_{i,j}^k$ is the pixel value of the perspective depth map $\bar{I}_d^k$ at coordinates $(i,j)$.
    - $v_{M(i,j)}$ is the pixel value of the panorama depth map $I_d$ at the corresponding mapped coordinate.
    - $\mathcal{L}_s$ is the smooth L1 loss.
    - The last two terms constitute a total variation regularization term [S1].
    
    *Pseudocode for Alignment Optimization:*
    ```python
    # Initialize learnable parameters
    # W: array of size K, global scales
    # B: array of size (K, H, W), local biases
    # I_d: array of size (H_p, W_p), canonical panorama depth map
    
    def smooth_l1_loss(x, y):
        diff = x - y
        abs_diff = abs(diff)
        return sum(min(abs_diff, 1.0) - 0.5) # Simplified representation of smooth L1

    def optimize_alignment(I_d, W, B, perspective_depths, M_map):
        for iteration in range(max_iterations):
            total_loss = 0
            for k in range(K):
                w_k = W[k]
                b_k = B[k]
                v_bar_k = perspective_depths[k]
                
                # Data alignment term
                for i, j in pixel_coordinates:
                    v_pano = I_d[M_map(i, j)]
                    v_est = v_bar_k[i, j] * w_k + b_k[i, j]
                    total_loss += smooth_l1_loss(v_est, v_pano)
                
                # Total variation regularization on biases
                for i, j in pixel_coordinates:
                    if j < H - 1:
                        total_loss += smooth_l1_loss(b_k[i, j+1], b_k[i, j])
                    if i < W - 1:
                        total_loss += smooth_l1_loss(b_k[i+1, j], b_k[i, j])
            
            # Update W, B, I_d using gradient descent on total_loss
            update_parameters(W, B, I_d, total_loss)
    ```

4.  **NeRF Training:** Train the single-view NeRF using the RGB panorama $I_{\text{rgb}}$ and the aligned depth map $I_d$. The NeRF is optimized by minimizing the combined color and depth reconstruction loss $\mathcal{L}_{\text{nerf}}'$ [S1]:
    $$ \mathcal{L}_{\text{nerf}}' = \sum_{\mathbf{r} \in \mathcal{R}} \left( \|\hat{C}(\mathbf{r}) - C(\mathbf{r})\|^2 + \|\hat{D}(\mathbf{r}) - \bar{D}(\mathbf{r})\|^2 \right) $$
    where:
    - $\mathcal{R}$ is the set of rays sampled from the panorama.
    - $\hat{C}(\mathbf{r})$ is the expected pixel color rendered by the NeRF.
    - $C(\mathbf{r})$ is the ground-truth pixel color from the panorama.
    - $\hat{D}(\mathbf{r})$ is the expected depth values rendered by the NeRF (computed as $\sum w_i t_i$ as per standard volume rendering).
    - $\bar{D}(\mathbf{r})$ is the corresponding depth pixel value from the aligned depth map $I_d$ [S1].

**Parameters:**
- $K$: Number of perspective projections. Set to 20 [S1].

**Edge Cases and Pitfalls:**
- **Scale Ambiguity:** Monocular depth estimation suffers from scale ambiguity, leading to inconsistent depth values in overlapped regions between perspective projections. The deformable alignment step with learnable scales $w_k$ and biases $b_{k,i,j}$ is required to resolve this [S1].
- **Visible Regions Only:** Since only a single panorama is available, the trained NeRF $f_{\text{nerf}}$ is only valid for visible regions. Invisible regions require separate completion methods (e.g., collaborative RGBD inpainting) [S1].

## Operating characteristics
- **Time Complexity:** The method adds an initial optimization step for deformable alignment (Eq. 2) before the main NeRF training loop. The complexity depends on the number of iterations for alignment and the standard NeRF training iterations.
- **Memory Complexity:** Requires storing $K$ perspective depth maps, the canonical depth map $I_d$, and the learnable parameters $W$ and $B$.
- **Strengths:** Provides accurate geometry reconstruction for visible regions in single-view scenarios by leveraging explicit depth priors, addressing the under-constrained nature of single-view NeRF [S1].
- **Limitations:** The resulting NeRF is only valid for visible regions; invisible regions are not reconstructed by this specific method [S1]. Relies on the quality of the external monocular depth estimator [52].

## Background
This method is part of the PERF (Panoramic Neural Radiance Field From a Single Panorama) framework [S1]. It was introduced to address the lack of geometry consistency in single-view NeRF methods, which often rely on inpainting or view-consistency refinement without explicit depth initialization [S1]. It builds on the principle that initializing geometry with a 360-degree depth estimation method helps constrain the optimization [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
