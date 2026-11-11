# Single-view NeRF Training with Depth Map

Also called: Depth-guided Single-view NeRF, Single-view NeRF with Depth Initialization

## Overview
This method addresses the under-constrained optimization problem of single-view NeRF reconstruction by initializing the geometry of visible regions using a 360-degree depth estimation [S1]. It sets itself apart from sibling methods by explicitly fusing multiple perspective depth maps into a canonical panorama depth map via deformable alignment before NeRF training, rather than relying solely on photometric loss or geometric baselines [S1].

## How it works
**Inputs and Outputs:**
- Input: A single input panorama $I_{\text{rgb}}$.
- Output: A trained NeRF model $f_{\text{nerf}}$ representing the visible regions of the scene.

**Data Structures:**
- $I_{\text{rgb}}$: The input panoramic RGB image.
- $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$: Set of $K$ perspective-projected RGB images.
- $\{\bar{I}_d^k\}_{k=1}^K$: Set of $K$ estimated perspective depth maps.
- $I_d$: The canonical learnable panorama depth map.
- $w_k$: Scale parameter for the $k$-th perspective depth map.
- $\{b_{k,i,j}\}$: Local bias values for the $k$-th perspective depth map at pixel coordinates $(i,j)$.
- $M$: Projective operation mapping perspective image coordinates to panorama image coordinates.

**Steps:**

1.  **Perspective Projection:** Project the single input panorama $I_{\text{rgb}}$ into $K$ perspective images $\{\bar{I}_{\text{rgb}}^k\}_{k=1}^K$ using different directions. The number of projections is set to $K=20$ [S1]. This is necessary because existing monocular depth estimators are trained on limited field-of-view images and do not work directly on 360-degree panoramas warped into rectangular matrices [S1].

2.  **Depth Estimation:** Use a depth estimation model [52] to estimate depth values for each perspective image $\bar{I}_{\text{rgb}}^k$, obtaining the set of depth maps $\{\bar{I}_d^k\}_{k=1}^K$ [S1]. Note that due to scale ambiguity in single-view depth estimation, estimated depth values in overlapped regions may differ significantly [S1].

3.  **Deformable Alignment:** Perform deformable alignment to jointly align the perspective depth maps $\{\bar{I}_d^k\}$ to a canonical learnable panorama depth map $I_d$. This is achieved by optimizing the following objective function [S1]:

    $$ \sum_{k,i,j} \left\| \bar{v}_{i,j}^k w_k + b_{k,i,j} - v_{M(i,j)} \right\| + L_s(b_{i,j+1}, b_{i,j}) + L_s(b_{i+1,j}, b_{i,j}) $$

    where:
    - $\bar{v}_{i,j}^k$ is the pixel value of the perspective depth map $\bar{I}_d^k$.
    - $v$ is the pixel value of the panorama depth map $I_d$.
    - $L_s$ is the smooth L1 loss.
    - $M$ is the projective operation mapping perspective image coordinates to panorama image coordinates.
    - The second and third terms constitute a total variation term [S1].

4.  **NeRF Training:** Train the single-view NeRF using the RGB panorama $I_{\text{rgb}}$ and the aligned depth map $I_d$. The NeRF is optimized by minimizing the combined color and depth reconstruction loss $\mathcal{L}_{\text{nerf}}'$ [S1]:

    $$ \mathcal{L}_{\text{nerf}}' = \sum_{r \in R} \left( \|\hat{C}(r) - C(r)\|^2 + \|\hat{D}(r) - \bar{D}(r)\|^2 \right) $$

    where:
    - $r \in R$ represents rays.
    - $\hat{C}(r)$ is the expected pixel color rendered by the NeRF.
    - $C(r)$ is the ground-truth pixel color.
    - $\hat{D}(r)$ is the expected depth values rendered by the NeRF.
    - $\bar{D}(r)$ is the corresponding depth pixel computed by the depth estimator [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_view_nerf_training_with_depth_map(I_rgb, K=20):
    # Step 1: Project panorama to K perspective images
    perspective_rgb_images = project_panorama_to_perspective(I_rgb, K)
    
    # Step 2: Estimate depth for each perspective image
    perspective_depth_maps = []
    for img in perspective_rgb_images:
        depth_map = depth_estimation_model(img) # Reference [52]
        perspective_depth_maps.append(depth_map)
        
    # Step 3: Deformable Alignment to canonical panorama depth map
    # Initialize learnable panorama depth map I_d, scales w_k, and biases b_k
    I_d = initialize_panorama_depth()
    w = [1.0 for _ in range(K)]
    b = [[0.0 for _ in range(height)] for _ in range(width)] # Local biases
    
    # Optimize alignment objective (Eq. 2)
    # Note: In implementation, this requires differentiable projection M
    # and optimization of w, b, and I_d jointly.
    for iteration in range(align_iterations):
        loss_align = 0.0
        for k in range(K):
            for i, j in pixels:
                # Map perspective coordinates to panorama coordinates
                pano_coords = M(perspective_coords(k, i, j))
                pano_u, pano_v = pano_coords
                
                # Get values
                v_bar = perspective_depth_maps[k][i, j]
                v_pano = I_d[pano_u, pano_v]
                
                # Data alignment term
                aligned_val = v_bar * w[k] + b[k][i, j]
                loss_align += abs(aligned_val - v_pano)
                
                # Total variation terms for biases
                if j < width - 1:
                    loss_align += smooth_l1(b[k][i, j+1], b[k][i, j])
                if i < height - 1:
                    loss_align += smooth_l1(b[k][i+1, j], b[k][i, j])
                    
        # Backpropagate to update w, b, and I_d
        update_parameters(loss_align)
        
    # Step 4: Train NeRF with combined loss
    nerf_model = initialize_nerf()
    for iteration in range(nerf_iterations):
        loss_nerf = 0.0
        for ray in sample_rays(I_rgb):
            # Render color and depth from NeRF
            C_hat = nerf_model.render_color(ray)
            D_hat = nerf_model.render_depth(ray)
            
            # Get ground truth color and aligned depth
            C_gt = ray.get_color()
            D_bar = get_aligned_depth(ray, I_d) # Corresponding depth from I_d
            
            # Compute loss (Eq. 3)
            color_loss = mse(C_hat, C_gt)
            depth_loss = mse(D_hat, D_bar)
            loss_nerf += color_loss + depth_loss
            
        # Backpropagate to update NeRF parameters
        update_nerf_parameters(loss_nerf)
        
    return nerf_model
```

**Parameters:**
- $K$: Number of perspective projections. Set to 20 [S1].

**Edge Cases and Pitfalls:**
- Scale Ambiguity: Single-view depth estimation suffers from scale ambiguity, leading to large differences in overlapped regions between perspective depth maps [S1]. The deformable alignment step is critical to resolve this.
- Field of View: Standard monocular depth estimators do not work on 360-degree panoramas directly; projection to limited FOV perspective images is required [S1].
- Visible Regions Only: Since only a single panorama is available, the trained NeRF $f_{\text{nerf}}$ only works for visible regions. Invisible regions require separate completion methods (e.g., collaborative RGBD inpainting) [S1].

## Operating characteristics
- **Time Complexity:** The method involves an initial alignment phase optimizing $K$ depth maps to a canonical map, followed by standard NeRF training. The alignment adds overhead proportional to $K$ and the number of pixels.
- **Memory Complexity:** Requires storing $K$ perspective depth maps and the canonical panorama depth map during alignment.
- **Strengths:** Provides accurate reconstruction for visible regions by leveraging geometric priors from depth estimation, addressing the under-constrained nature of single-view NeRF [S1].
- **Limitations:** The resulting NeRF is limited to visible regions; invisible regions are not reconstructed by this specific method [S1].

## Background
This method is part of the PERF (Panoramic Neural Radiance Field From a Single Panorama) framework [S1]. It was introduced to address geometry consistency issues in single-view panoramic NeRF reconstruction, where previous methods like SPInNeRF focused on inpainting and perceptual loss without explicit geometry consistency mechanisms [S1]. The approach builds on 360MonoDepth [51] for depth estimation [S1].

## Sources
[S1] PERF: Panoramic Neural Radiance Field From a Single Panorama, https://doi.org/10.1109/tpami.2024.3387307, fetched 2026-10-26
