# Two-stage Gaussian-Surfel (GES) Optimization

Also called: 2D-GES optimization, 3D-GES optimization, Gaussian-Surfel Joint Optimization

## Overview
Two-stage Gaussian-Surfel (GES) optimization is a training pipeline for differentiable rendering that separates the initialization of opaque surface geometry from the joint optimization of appearance and fine geometry. It solves the problem of color leakage and order-independent blending artifacts that occur when translucent surfels and 3D Gaussians are optimized simultaneously [S1]. Unlike single-stage approaches where Gaussians dominate image fitting at the expense of surfel coverage, GES first establishes a coarse, opaque surfel backbone, then enriches it with 3D Gaussians for high-frequency details [S1].

## How it works
The method consists of two distinct phases: Surfel Optimization (Stage 1) and Gaussian-Surfel Joint Optimization (Stage 2).

### Inputs and Outputs
**Inputs:**
*   Training images from $J$ views.
*   Camera parameters for each view.
*   Initial surfel primitives (position, opacity, spherical harmonics or RGB color, normal).

**Outputs:**
*   Optimized surfel primitives (position, opacity, color, normal).
*   Optimized 3D Gaussian primitives (position, covariance, opacity, spherical harmonics).

### Data Structures
The implementation relies on two sets of primitives:
1.  **Surfels:** Each surfel $i$ stores position $\mathbf{p}_i$, opacity $\alpha_i$, color $c_i$ (RGB or SH coefficients), and normal $\mathbf{n}_i$.
2.  **3D Gaussians:** Each Gaussian $k$ stores position $\mathbf{p}_k$, covariance matrix $\Sigma_k$ (or scale/rotation), opacity $\alpha_k$, and spherical harmonics coefficients.

### Step 1: Surfel Optimization (Stage 1)
The goal is to establish a coarse, opaque geometry.

1.  **Initialization:** Initialize surfel positions and properties. Set initial opacity values to be translucent (e.g., low initial weights).
2.  **Rendering:** Render color, depth, and normal maps using alpha blending. The alpha-blended depth $D(\hat{\mathbf{x}})$ and normal $N(\hat{\mathbf{x}})$ for a pixel $\hat{\mathbf{x}}$ are computed as [S1]:
    $$
    D(\hat{\mathbf{x}}) = \sum_{i=1}^N d_i \alpha_i(\hat{\mathbf{x}}) \prod_{j=1}^{i-1} (1 - \alpha_j(\hat{\mathbf{x}}))
    $$
    $$
    N(\hat{\mathbf{x}}) = \sum_{i=1}^N \mathbf{n}_i \alpha_i(\hat{\mathbf{x}}) \prod_{j=1}^{i-1} (1 - \alpha_j(\hat{\mathbf{x}}))
    $$
    where $d_i$ is the depth of surfel $i$, $\mathbf{n}_i$ is its normal, and $\alpha_i(\hat{\mathbf{x}})$ is its projected opacity. $N$ is the total number of surfels.
3.  **Supervision Target Generation:** At the midpoint of the surfel optimization iterations, render the alpha-blended normal maps $N^{supv}$ for all training views. These serve as static supervision targets for normal consistency in later iterations [S1].
4.  **Loss Computation:**
    *   **RGB Loss:** Standard photometric loss between rendered and ground truth images.
    *   **Normal Consistency Loss ($\mathcal{L}_n$):** Aligns the rendered normal map $N(\hat{\mathbf{x}})$ with the normals derived from the rendered depth map $D(\hat{\mathbf{x}})$ or the static target $N^{supv}$. The loss is defined as [S1]:
        $$
        \mathcal{L}_{sn} = \sum_{\hat{\mathbf{x}}} (1 - N^{supv}(\hat{\mathbf{x}}) \cdot N(\hat{\mathbf{x}}))
        $$
        Note: $\mathcal{L}_n$ is effective primarily in early iterations for translucent surfels. In later iterations, opaque regions naturally maintain consistent gradients, making this loss redundant for high-weight surfels [S1].
5.  **Surfel Densification/Pruning:** Periodically densify surfels based on gradient magnitude or visibility. Prune surfels with opacity/weight below a threshold (e.g., $w_{thresh} = 16$ for real scenes, $4$ for synthetic) [S1].
6.  **Sorting Order Adjustment:** During optimization, adjust the per-pixel sorting order of surfels to prevent protrusion and interleaving artifacts, particularly in textureless scenes [S1].

### Step 2: Gaussian-Surfel Joint Optimization (Stage 2)
3D Gaussians are added to refine geometry and appearance.

1.  **Gaussian Initialization:**
    *   Identify surfels with opacity/weight $w_i < 0.8$ at the 10,000-th iteration of Stage 1.
    *   Initialize 3D Gaussian positions using the positions of these selected surfels. Discard surfels with $w_i \ge 0.8$ [S1].
    *   Initialize other Gaussian properties (scale, rotation, opacity, SH coefficients) following the standard 3DGS initialization protocol [Kerbl et al. 2023] [S1].
2.  **Joint Optimization Loop:**
    *   **Rendering:** Render the combined scene (surfels + Gaussians).
    *   **Loss Function:** The total loss is [S1]:
        $$
        \mathcal{L} = \lambda_n \mathcal{L}_n + \lambda_{gd} \mathcal{L}_{gd} + \mathcal{L}_{rgb}
        $$
        where $\lambda_n = 0.05$ and $\lambda_{gd} = 0.1$ [S1].
        *   $\mathcal{L}_{rgb}$: Photometric loss.
        *   $\mathcal{L}_n$: Normal consistency loss (using the static target $N^{supv}$ from Stage 1).
        *   $\mathcal{L}_{gd}$: Geometry depth loss, constraining the weighted depth of Gaussians to match surfel depth:
            $$
            \mathcal{L}_{gd} = \text{MAE}\left(\frac{D_G}{W_G} - D_s\right)
            $$
            where $D_G$ and $W_G$ are the weighted depth and weight of Gaussians, and $D_s$ is the surfel depth [S1].
    *   **Gaussian Densification:** Every 1000 iterations:
        1.  Compute the squared error map between ground truth and rendered image across all training views.
        2.  Normalize the error map and treat normalized error as sampling probability.
        3.  Sample a fixed number of pixels per image based on these probabilities.
        4.  Use the corresponding surfel depth map and camera parameters to obtain a point cloud for the initial positions of new 3D Gaussians.
        5.  Initialize other Gaussian properties as in [Kerbl et al. 2023] [S1].
    *   **Gaussian Pruning:** Compute the contribution score $s_{i,j}$ for each Gaussian $i$ in view $j$ [S1]:
        $$
        s_{i,j} = \max_{\hat{\mathbf{x}}} \frac{c_{i,j} \alpha_{i,j}(\hat{\mathbf{x}})}{W_s + W_G}
        $$
        where $\hat{\mathbf{x}}$ is the pixel covered by the Gaussian, $\alpha_{i,j}(\hat{\mathbf{x}})$ is the Gaussian's opacity, $c_{i,j} = \max_k c_{i,j}^k$ is the maximum channel value of the Gaussian's color, and $W_s, W_G$ are the total weights of surfels and Gaussians respectively. Prune Gaussians where $\max_j s_{i,j} < 0.02$ across all $J$ views [S1].

### Parameters
*   **$\epsilon$ (Truncation/Weighting Parameter):** Controls the weighting of Gaussians near surfel depth. Strategies include:
    *   $\epsilon = 0$: Causes color discontinuities by wrongly truncating Gaussians close to surfel depth [S1].
    *   $\epsilon = 0.1$ (Fixed): May cause color leakage in fine structures [S1].
    *   Adaptive $\epsilon$: Adjusted based on geometry granularity for high-quality rendering [S1].
*   **Surfel Weight Threshold ($w_{thresh}$):** 16 for real scenes, 4 for synthetic scenes [S1].
*   **Densification Interval:** Every 1000 iterations [S1].
*   **Surfel Discard Threshold:** $w_i < 0.8$ at iteration 10,000 [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_ges_optimization(training_images, camera_params, initial_surfels):
    """
    Implements the Two-stage Gaussian-Surfel Optimization.
    """
    # Stage 1: Surfel Optimization
    surfels = initial_surfels
    mid_point_iter = 10000  # Assumed midpoint for supervision target generation
    
    # Render supervision targets at midpoint
    # Note: In practice, this is done during the loop at the specific iteration
    supervision_targets = None 
    
    for iter in range(20000): # Example total iterations
        # --- Stage 1 Logic (First half) ---
        if iter < mid_point_iter:
            # Render surfels
            rendered_color, rendered_depth, rendered_normal = render_surfels(surfels, camera_params)
            
            # Compute Losses
            rgb_loss = compute_rgb_loss(rendered_color, training_images[iter])
            normal_loss = compute_normal_loss(rendered_normal, rendered_depth, surfels)
            
            # Generate supervision targets at midpoint
            if iter == mid_point_iter:
                supervision_targets = render_alpha_blended_normals(surfels, camera_params)
            
            # Backprop and update surfels
            update_surfels(surfels, rgb_loss, normal_loss)
            
            # Periodic Densification/Pruning for Surfels
            if iter % 1000 == 0:
                surfels = densify_surfels(surfels, training_images, camera_params)
                surfels = prune_surfels(surfels, threshold=16) # Real scene threshold
        
        # --- Stage 2 Logic (Second half) ---
        else:
            # Initialize Gaussians if first time entering Stage 2
            if not hasattr(globals(), 'gaussians'):
                gaussians = initialize_gaussians_from_surfels(surfels, threshold=0.8)
            
            # Render combined scene
            rendered_color, rendered_depth, rendered_normal = render_combined(surfels, gaussians, camera_params)
            
            # Compute Losses
            rgb_loss = compute_rgb_loss(rendered_color, training_images[iter])
            
            # Normal consistency loss using static supervision targets
            # Only effective for translucent parts or early joint phase
            normal_loss = compute_normal_loss_supervised(rendered_normal, supervision_targets)
            
            # Geometry depth loss
            gaussians_depth, gaussians_weight = compute_weighted_depth_weight(gaussians)
            surfels_depth = compute_surfel_depth(surfels)
            geo_loss = compute_mae_loss(gaussians_depth / gaussians_weight, surfels_depth)
            
            total_loss = 0.05 * normal_loss + 0.1 * geo_loss + rgb_loss
            
            # Backprop and update both
            update_surfels(surfels, total_loss)
            update_gaussians(gaussians, total_loss)
            
            # Periodic Densification/Pruning for Gaussians
            if iter % 1000 == 0:
                # Densification based on error map
                error_map = compute_squared_error_map(rendered_color, training_images[iter])
                sampled_pixels = sample_pixels_from_error_map(error_map)
                new_gaussians = initialize_gaussians_from_depth_map(sampled_pixels, surfels, camera_params)
                gaussians = merge_gaussians(gaussians, new_gaussians)
                
                # Pruning based on contribution score
                gaussians = prune_gaussians(gaussians, surfels, threshold=0.02)

    return surfels, gaussians

def render_alpha_blended_normals(surfels, camera_params):
    """
    Renders alpha-blended normal map using Eq. 14 from [S1].
    """
    # Sort surfels by depth for each pixel
    sorted_surfels = sort_surfels_by_depth(surfels, camera_params)
    
    normal_map = zeros_like(image_shape)
    accumulated_alpha = zeros_like(image_shape)
    
    for surfel in sorted_surfels:
        projected_normal = project_normal(surfel.normal, camera_params)
        projected_alpha = project_opacity(surfel.opacity, camera_params)
        
        # Alpha blending formula
        # N(x) = sum(n_i * alpha_i * prod(1 - alpha_j))
        weight = projected_alpha * (1 - accumulated_alpha)
        normal_map += projected_normal * weight
        accumulated_alpha += weight
        
    return normal_map

def initialize_gaussians_from_surfels(surfels, threshold=0.8):
    """
    Initializes Gaussians from surfels with weight < threshold.
    """
    gaussians = []
    for surfel in surfels:
        if surfel.weight < threshold:
            g = Gaussian()
            g.position = surfel.position
            g.scale, g.rotation, g.opacity, g.sh_coeffs = init_3dgs_properties() # From Kerbl et al. 2023
            gaussians.append(g)
    return gaussians

def prune_gaussians(gaussians, surfels, threshold=0.02):
    """
    Prunes Gaussians based on contribution score Eq. 9 from [S1].
    """
    valid_gaussians = []
    for g in gaussians:
        max_score = 0.0
        for view in training_views:
            # Compute opacity alpha_ij(x) and color c_ij
            # Maximize over pixels covered by Gaussian
            score = 0.0
            for pixel in g.covered_pixels(view):
                alpha = g.opacity_at_pixel(pixel, view)
                color_val = max(g.color_channels(pixel, view))
                W_s = total_weight_surfels(view)
                W_G = total_weight_gaussians(view)
                
                s_ij = (color_val * alpha) / (W_s + W_G)
                if s_ij > score:
                    score = s_ij
            
            if score > max_score:
                max_score = score
        
        if max_score >= threshold:
            valid_gaussians.append(g)
            
    return valid_gaussians
```

### Edge Cases and Pitfalls
*   **Single-Stage Optimization:** Optimizing translucent surfels and Gaussians together favors Gaussians for image fitting, leaving insufficient surfel coverage. This causes color leakage in novel views due to order-independent Gaussian blending [S1].
*   **Sorting Order:** Omitting per-pixel sorting order adjustment during surfel optimization causes surfels to protrude and interleave under z-buffer depth testing, especially in textureless indoor scenes [S1].
*   **$\epsilon$ Parameter:** Setting $\epsilon = 0$ causes Gaussians close to surfel depth to be wrongly truncated, resulting in color discontinuities. Setting $\epsilon$ to a fixed small value (e.g., 0.1) may cause color leakage in fine structures [S1].
*   **Normal Loss Redundancy:** The normal consistency loss $\mathcal{L}_n$ is only effective for translucent surfels in early iterations. In later iterations, opaque regions naturally maintain consistent depth gradients and normals, making $\mathcal{L}_n$ redundant for those areas [S1].

## Operating characteristics
*   **Performance:** 3D-GES achieves 675 fps at 1080p and 233 fps at 2160p on the Mip-NeRF360 dataset [S1]. This is approximately 3.6× faster than 3DGS at 1080p [S1].
*   **Storage:** Requires 366MB for Mip-NeRF360 (less than half of 3DGS's 734MB). After pruning, quantization, and hash-grid compression, storage drops to 47MB [S1].
*   **Quality:** Two-stage optimization achieves a PSNR of 27.38 on Mip-NeRF360, compared to 27.04 for single-stage optimization [S1].
*   **Strengths:** Ultra-fast rendering, low storage, view-consistent rendering without popping artifacts, high-fidelity geometry via surfels [S1].
*   **Limitations:** Requires careful tuning of $\epsilon$ and two-stage separation. Single-stage fails to maintain surface coverage [S1].
*   **When to use:** When high rendering speed and low storage are critical, and high-fidelity geometry is required. Suitable for real-time applications [S1].
*   **When not to use:** When scene geometry is extremely complex and surfels cannot adequately capture it, or if implementation complexity of two-stage pipeline is prohibitive [S1].

## Background
Two-stage Gaussian-Surfel (GES) optimization was introduced in "When Gaussian Meets Surfel: Ultra-fast High-fidelity Radiance Field Rendering" [S1]. The method builds on 3D Gaussian Splatting [Kerbl et al. 2023] and 2D Gaussian Splatting [Huang et al. 2024], integrating surfel-based rendering to provide robust surface geometry. It addresses the limitations of pure Gaussian splatting, which lacks explicit surface representation, and pure surfel rendering, which may lack high-frequency detail. The two-stage approach was validated to be necessary for preventing color leakage and ensuring sufficient surface coverage [S1].

## Sources
[S1] When Gaussian Meets Surfel: Ultra-fast High-fidelity Radiance Field Rendering, https://doi.org/10.1145/3730925, fetched 2026-10-26
