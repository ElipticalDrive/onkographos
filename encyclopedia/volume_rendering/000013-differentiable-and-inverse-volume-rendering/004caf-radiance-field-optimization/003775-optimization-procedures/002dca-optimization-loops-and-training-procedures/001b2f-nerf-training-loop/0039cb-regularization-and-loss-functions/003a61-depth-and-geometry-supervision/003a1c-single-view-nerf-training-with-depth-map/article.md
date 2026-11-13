# Single-view NeRF Training with Depth Map

Also called: Depth-guided Single-view NeRF, Single-view NeRF with Depth Initialization

## Overview
This method addresses the under-constrained optimization problem of single-view NeRF reconstruction by initializing the geometry of visible regions using a 360-degree depth estimation [S1]. It sets itself apart from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by specifically handling the scale ambiguity and field-of-view limitations inherent in projecting a single panorama into perspective views for depth estimation, using a deformable alignment step to create a canonical depth map before NeRF training [S1].

## How it works
**Inputs:**
- A single input RGB panorama $I_{\text{rgb}}$.
- A pre-trained monocular depth estimation model [S1].

**Outputs:**
- A trained NeRF model $f_{\text{nerf}}$ representing the visible regions of the scene.
- A canonical learnable panorama depth map $I_d$.

**Data Structures:**
- **Perspective Projections:** A set of $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$ and corresponding depth maps $\{\bar{I}_d^k\}_{k=1}^K$.
- **Canonical Depth Map:** $I_d$, a panorama-resolution depth map.
- **Alignment Parameters:** For each perspective view $k$, a global scale factor $w_k$ and a local bias field $\{b_{k,i,j}\}$ corresponding to pixel coordinates $(i,j)$.
- **Coordinate Mapping:** A function $M$ that maps perspective image coordinates to panorama image coordinates.

**Numbered Steps:**

1.  **Perspective Projection:** Project the single input panorama $I_{\text{rgb}}$ into $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}$ using different viewing directions. The standard value for $K$ is 20 [S1]. This is necessary because existing monocular depth estimators are trained on limited field-of-view images and do not work directly on 360-degree panoramas warped into rectangular matrices [S1].

2.  **Depth Estimation:** Use a depth estimation model to estimate depth values for each perspective image $\bar{I}_{\text{rgb}}^k$, obtaining the set of perspective depth maps $\{\bar{I}_d^k\}$ [S1]. Note that due to scale ambiguity in single-view depth estimation, estimated depth values in overlapped regions between different perspectives may differ significantly [S1].

3.  **Deformable Alignment:** Perform a deformable alignment to jointly align the perspective depth maps $\{\bar{I}_d^k\}$ to a canonical learnable panorama depth map $I_d$. This step resolves scale and bias inconsistencies. The alignment is achieved by optimizing the following objective function [S1]:

    $$ \sum_{k,i,j} \left\| \bar{v}_{i,j}^k w_k + b_{k,i,j} - v_{M(i,j)} \right\| + \lambda_s \left( L_s(b_{i,j+1}, b_{i,j}) + L_s(b_{i+1,j}, b_{i,j}) \right) $$

    Where:
    - $\bar{v}_{i,j}^k$ is the pixel value of the perspective depth map $\bar{I}_d^k$ at $(i,j)$.
    - $v_{M(i,j)}$ is the pixel value of the canonical panorama depth map $I_d$ at the mapped coordinate.
    - $w_k$ is the scale value for perspective view $k$.
    - $b_{k,i,j}$ is the local bias value for pixel $(i,j)$ in view $k$.
    - $L_s$ is the smooth L1 loss.
    - The second term is a total variation regularization term [S1].

4.  **NeRF Training:** Train the single-view NeRF using the RGB panorama $I_{\text{rgb}}$ and the aligned depth map $I_d$. The NeRF is optimized by minimizing the combined color and depth reconstruction loss $\mathcal{L}_{\text{nerf}}'$ [S1]:

    $$ \mathcal{L}_{\text{nerf}}' = \sum_{r \in \mathcal{R}} \left( \|\hat{C}(r) - C(r)\|^2 + \|\hat{D}(r) - \bar{D}(r)\|^2 \right) $$

    Where:
    - $\mathcal{R}$ is the set of rays sampled from the panorama.
    - $\hat{C}(r)$ is the expected pixel color rendered by the NeRF for ray $r$.
    - $C(r)$ is the ground-truth pixel color from the panorama.
    - $\hat{D}(r)$ is the expected depth value rendered by the NeRF for ray $r$.
    - $\bar{D}(r)$ is the corresponding depth pixel computed by the depth estimator (derived from $I_d$) for ray $r$.

    The trained NeRF $f_{\text{nerf}}$ is valid only for visible regions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_view_nerf_with_depth(panorama_rgb, depth_estimator, k=20):
    """
    Trains a single-view NeRF using depth-initialized geometry.
    
    Args:
        panorama_rgb: Input RGB panorama image.
        depth_estimator: Pre-trained monocular depth estimation model.
        k: Number of perspective projections (default 20).
        
    Returns:
        trained_nerf: The trained NeRF model.
    """
    
    # Step 1: Project panorama to K perspective views
    perspective_images = []
    for view_idx in range(k):
        # Project panorama to perspective view using specific direction
        # Implementation of projection M depends on camera intrinsics/extrinsics
        img = project_panorama_to_perspective(panorama_rgb, view_idx)
        perspective_images.append(img)
        
    # Step 2: Estimate depth for each perspective view
    perspective_depth_maps = []
    for img in perspective_images:
        # Get depth map from estimator
        d_map = depth_estimator.predict(img)
        perspective_depth_maps.append(d_map)
        
    # Step 3: Deformable Alignment to canonical panorama depth map
    # Initialize canonical depth map I_d (e.g., zeros or mean of estimates)
    canonical_depth_map = initialize_canonical_depth(panorama_rgb.shape)
    
    # Initialize alignment parameters
    scales = [1.0 for _ in range(k)]
    biases = [[[[0.0 for _ in range(width)] for _ in range(height)] for _ in range(k)] for _ in range(k)] # Placeholder for structure
    
    # Optimize alignment parameters to minimize Eq. (2)
    # This is an optimization loop over w_k, b_k,i,j
    for iteration in range(num_alignment_iterations):
        loss = 0.0
        for k_idx in range(k):
            for i in range(height):
                for j in range(width):
                    # Get pixel value from perspective depth map
                    v_bar = perspective_depth_maps[k_idx][i, j]
                    
                    # Map coordinates to panorama coordinates
                    pano_i, pano_j = map_perspective_to_pano(i, j, k_idx)
                    
                    # Get pixel value from canonical depth map
                    v_pano = canonical_depth_map[pano_i, pano_j]
                    
                    # Data alignment term: || v_bar * w_k + b_k,i,j - v_pano ||
                    # Using L1 or Smooth L1 as per source
                    data_term = smooth_l1_loss(v_bar * scales[k_idx] + biases[k_idx][i][j], v_pano)
                    loss += data_term
                    
                    # Total variation regularization on biases
                    if i < height - 1:
                        loss += smooth_l1_loss(biases[k_idx][i+1][j], biases[k_idx][i][j])
                    if j < width - 1:
                        loss += smooth_l1_loss(biases[k_idx][i][j+1], biases[k_idx][i][j])
                        
        # Backpropagate to update scales, biases, and canonical_depth_map
        update_parameters(loss)
        
    # Step 4: Train NeRF with RGB and Aligned Depth
    nerf_model = initialize_nerf()
    
    for training_step in range(num_nerf_iterations):
        # Sample rays from panorama
        rays = sample_rays_from_panorama(panorama_rgb)
        
        # Render color and depth from NeRF
        rendered_colors, rendered_depths = nerf_model.render(rays)
        
        # Get ground truth color and depth
        gt_colors = get_pixel_colors(rays, panorama_rgb)
        gt_depths = get_depth_values(rays, canonical_depth_map)
        
        # Compute Loss: L_nef' = sum(||C_hat - C||^2 + ||D_hat - D_bar||^2)
        color_loss = mean_squared_error(rendered_colors, gt_colors)
        depth_loss = mean_squared_error(rendered_depths, gt_depths)
        total_loss = color_loss + depth_loss
        
        # Update NeRF weights
        update_nerf_weights(total_loss)
        
    return nerf_model
```

**Parameters:**
- $K$: Number of perspective projections. Typical value is 20 [S1].

**Edge Cases and Pitfalls:**
- **Scale Ambiguity:** Monocular depth estimation suffers from scale ambiguity, leading to inconsistent depth values in overlapped regions between perspective views. The deformable alignment step is critical to resolve this [S1].
- **Field of View:** Standard depth estimators fail on 360-degree panoramas. Projection to limited FOV perspective views is required [S1].
- **Visible Regions Only:** The trained NeRF is only valid for visible regions in the single panorama. Invisible regions require separate inpainting/completion methods (e.g., using Stable Diffusion) [S1].

## Subtopics

### Depth-Supervised NeRF
Uses depth supervision but typically assumes multi-view consistency or direct depth input without the specific single-view panorama projection and deformable alignment pipeline [8687-depth-supervised-nerf.md].

### Geometric Baseline Regularization
Focuses on regularization terms for geometry without necessarily using explicit depth map initialization from a single view [7475-geometric-baseline-regularization.md].

### 3D Point Supervision Loss
Uses 3D points for supervision rather than dense depth maps [12693-3d-point-supervision-loss.md].

## Operating characteristics
- **Time Complexity:** The method involves an initial optimization step for deformable alignment over $K$ views and $N$ pixels, followed by standard NeRF training. The alignment step adds significant overhead compared to standard single-view NeRF.
- **Memory Complexity:** Requires storing $K$ perspective depth maps and the canonical depth map. The alignment parameters (scales and biases) add memory overhead proportional to $K \times \text{resolution}$.
- **Strengths:** Provides accurate geometry initialization for visible regions in single-view scenarios, addressing the under-constrained nature of the problem [S1].
- **Limitations:** Only reconstructs visible regions. Invisible regions require additional inpainting steps. Performance depends on the quality of the monocular depth estimator [S1].
- **When to Use:** When only a single panoramic image is available and accurate geometry for visible regions is required.
- **When Not to Use:** When multi-view images are available (use multi-view NeRF methods) or when invisible regions are critical and cannot be inpainted [S1].

## Background
This method was introduced in the paper "PERF: Panoramic Neural Radiance Field From a Single Panorama" [S1]. It addresses the limitations of previous single-view methods that did not fully address geometry consistency. The approach builds on 360MonoDepth [51] for depth estimation and integrates it with NeRF training. It is part of the broader effort to enable photo-realistic panoramic NeRF from single-view inputs [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
