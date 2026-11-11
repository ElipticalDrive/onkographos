# Single-view NeRF Training with Depth Map

Also called: Depth-guided Single-view NeRF, Single-view NeRF with Depth Initialization

## Overview
This method addresses the under-constrained optimization problem of reconstructing a 360-degree scene from a single panoramic input by initializing the geometry of visible regions using a 360-degree depth estimation prior [S1]. It sets itself apart from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by explicitly handling the scale ambiguity and misalignment inherent in projecting a single panorama into multiple perspective views for depth estimation, rather than relying on multi-view consistency or direct monocular depth supervision on the final rendered rays [S1].

## How it works
**Inputs:** A single input RGB panorama $I_{\text{rgb}}$.
**Outputs:** A trained NeRF model $f_{\text{nerf}}$ representing the visible regions of the scene, initialized with a consistent depth map $I_d$.

**Data Structures:**
1.  **Perspective Projections:** A set of $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$ and their corresponding estimated depth maps $\{\bar{I}_d^k\}_{k=1}^K$.
2.  **Canonical Panorama Depth Map:** A learnable depth map $I_d$ in panorama coordinates.
3.  **Alignment Parameters:** For each perspective view $k$, a global scale factor $w_k$ and a local bias field $\{b_{k,i,j}\}$, where $i,j$ are pixel coordinates.

**Numbered Steps:**

1.  **Perspective Projection:** Project the single input panorama $I_{\text{rgb}}$ into $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$ using different viewing directions. The standard value for $K$ is 20 [S1]. This is necessary because existing monocular depth estimators are trained on limited field-of-view images and fail on 360-degree panoramas warped into rectangular matrices [S1].

2.  **Monocular Depth Estimation:** Use a pre-trained depth estimation model (e.g., [52] in [S1]) to estimate depth values for each perspective image $\bar{I}_{\text{rgb}}^k$, obtaining the set of depth maps $\{\bar{I}_d^k\}_{k=1}^K$. Note that due to scale ambiguity in single-view depth estimation, depth values in overlapped regions between different perspective views may differ significantly [S1].

3.  **Deformable Alignment:** Perform a deformable alignment to jointly align the perspective depth maps $\{\bar{I}_d^k\}$ to a canonical learnable panorama depth map $I_d$. This step resolves scale and bias inconsistencies. The alignment is achieved by optimizing the following objective function [S1]:

    $$
    \sum_{k,i,j} \left\| \bar{v}_{i,j}^k w_k + b_{k,i,j} - v_{M(i,j)} \right\| + L_s(b_{i,j+1}, b_{i,j}) + L_s(b_{i+1,j}, b_{i,j})
    $$

    Where:
    *   $\bar{v}_{i,j}^k$ is the pixel value of the depth map $\bar{I}_d^k$ at coordinates $(i,j)$.
    *   $v_{M(i,j)}$ is the pixel value of the canonical panorama depth map $I_d$ at the coordinate mapped from $(i,j)$ via the projective operation $M$.
    *   $w_k$ is the scale value for view $k$.
    *   $b_{k,i,j}$ is the local bias value for view $k$ at pixel $(i,j)$.
    *   $L_s$ is the smooth L1 loss.
    *   The last two terms constitute a total variation regularization term [S1, S53].

4.  **NeRF Training with Depth Supervision:** Train the single-view NeRF using the RGB panorama $I_{\text{rgb}}$ and the aligned depth map $I_d$. The NeRF is optimized by minimizing the combined color and depth reconstruction loss $\mathcal{L}_{\text{nerf}}'$ [S1]:

    $$
    \mathcal{L}_{\text{nerf}}' = \sum_{r \in R} \left( \left\| \hat{C}(r) - C(r) \right\|^2 + \left\| \hat{D}(r) - \bar{D}(r) \right\|^2 \right)
    $$

    Where:
    *   $R$ is the set of sampled rays.
    *   $\hat{C}(r)$ is the expected pixel color rendered by the NeRF for ray $r$.
    *   $C(r)$ is the ground-truth pixel color from the panorama.
    *   $\hat{D}(r)$ is the expected depth value rendered by the NeRF for ray $r$.
    *   $\bar{D}(r)$ is the corresponding depth pixel value computed by the depth estimator (derived from $I_d$) for ray $r$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_view_nerf_with_depth(panorama_rgb, depth_estimator, k=20):
    """
    Trains a single-view NeRF using depth-initialized geometry.
    
    Args:
        panorama_rgb: Input 360-degree RGB panorama.
        depth_estimator: Pre-trained monocular depth estimation model.
        k: Number of perspective projections (default 20).
        
    Returns:
        trained_nerf: The optimized NeRF model.
    """
    
    # Step 1: Project panorama to K perspective views
    perspective_images = []
    for view_idx in range(k):
        # Project panorama to perspective view using specific direction
        img = project_panorama_to_perspective(panorama_rgb, view_idx)
        perspective_images.append(img)
        
    # Step 2: Estimate depth for each perspective view
    perspective_depth_maps = []
    for img in perspective_images:
        # Use external depth estimator [52]
        depth_map = depth_estimator.predict(img)
        perspective_depth_maps.append(depth_map)
        
    # Step 3: Deformable Alignment to canonical panorama depth map
    # Initialize learnable canonical depth map I_d
    canonical_depth_map = initialize_depth_map(panorama_rgb.shape)
    
    # Initialize alignment parameters
    scales = [1.0] * k  # w_k
    biases = [[0.0] * width * height for _ in range(k)] # b_k,i,j
    
    # Optimize alignment objective (Eq. 2)
    # This requires differentiable rendering of depth map or direct pixel-wise optimization
    # minimizing: sum_k || v_bar * w + b - v_M || + TV(b)
    optimize_alignment(perspective_depth_maps, canonical_depth_map, scales, biases)
    
    # Step 4: Train NeRF with combined loss (Eq. 3)
    nerf = initialize_nerf()
    
    for iteration in range(num_iterations):
        # Sample rays from the panorama
        rays = sample_rays_from_panorama(panorama_rgb)
        
        # Render color and depth from NeRF
        rendered_colors, rendered_depths = nerf.render(rays)
        
        # Get ground truth color and corresponding depth from aligned map
        gt_colors = get_pixel_colors(rays, panorama_rgb)
        gt_depths = get_depth_from_canonical_map(rays, canonical_depth_map)
        
        # Calculate Loss
        color_loss = mean_squared_error(rendered_colors, gt_colors)
        depth_loss = mean_squared_error(rendered_depths, gt_depths)
        total_loss = color_loss + depth_loss
        
        # Backpropagate and update NeRF parameters
        update_nerf(nerf, total_loss)
        
    return nerf
```

**Edge Cases and Pitfalls:**
*   **Scale Ambiguity:** Monocular depth estimation suffers from scale ambiguity. The deformable alignment step is critical to resolve this by learning scale factors $w_k$ and biases $b_{k,i,j}$ [S1].
*   **Panorama Warping:** Standard depth estimators fail on 360-degree panoramas due to the rectangular warping artifact. Projection to limited FOV perspective views is required [S1].
*   **Visible Regions Only:** The trained NeRF $f_{\text{nerf}}$ only accurately represents visible regions. Invisible regions require separate completion methods (e.g., inpainting) [S1].

## Operating characteristics
*   **Time Complexity:** The method adds an initial alignment phase involving optimization of scale and bias parameters for $K$ views, followed by standard NeRF training. The alignment step adds overhead proportional to $K \times \text{pixels} \times \text{iterations}$.
*   **Memory Complexity:** Requires storing $K$ perspective depth maps and the canonical depth map. The NeRF model size is standard.
*   **Strengths:** Provides geometric initialization for single-view reconstruction, addressing the under-constrained nature of the problem. Handles the specific challenge of 360-degree depth estimation by leveraging perspective projections.
*   **Limitations:** Only reconstructs visible regions. Performance depends on the accuracy of the external depth estimator [52]. The alignment optimization adds complexity.
*   **When to use:** When only a single panoramic image is available and geometric consistency is desired for visible regions.
*   **When not to use:** When multi-view images are available (use multi-view NeRF methods) or when invisible regions need to be reconstructed without additional inpainting steps.

## Background
This method originates from the work on Panoramic Neural Radiance Field (PERF) from a single panorama [S1]. It was developed to address the limitations of previous single-view methods that did not fully ensure geometry consistency. The approach builds on 360MonoDepth [51] for depth estimation and integrates it with NeRF training.

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
