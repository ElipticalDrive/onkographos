# AdR-Gaussian

Also called: Adaptive Radius Gaussian Splatting

## Overview
AdR-Gaussian is a rendering acceleration method for 3D Gaussian Splatting that optimizes the rasterization pipeline by moving serial culling operations to a parallel preprocessing stage and balancing computational loads across pixel threads [S1]. It achieves significant speedups without reducing the total number of Gaussians, thereby avoiding the quality degradation associated with pruning-based acceleration methods [S1].

## How it works
AdR-Gaussian accelerates rendering by addressing two bottlenecks in the standard 3DGS rasterization pipeline: the serial nature of culling in the Render stage and the uneven computational load across pixel threads [S1]. The method operates in two phases: Early Culling and Load Balancing [S1].

### Inputs and Outputs
*   **Inputs:** A set of 3D Gaussians (defined by position, covariance, opacity $\sigma$, and spherical harmonics), camera parameters, and a target image resolution.
*   **Outputs:** A rendered image with reduced computational cost compared to standard 3DGS rasterization [S1].

### Data Structures
The method relies on the standard 3DGS rasterization pipeline stages [S1]:
1.  **Preprocess:** Parallel Gaussian-level processing.
2.  **InclusiveSum:** Parallel Gaussian-level processing.
3.  **DuplicateWithKeys:** Parallel Gaussian-level processing.
4.  **SortPairs:** Gaussian-Tile pair parallel sorting.
5.  **IdentifyTileRanges:** Gaussian-Tile pair parallel range identification.
6.  **Render:** Pixel-level parallel rendering with serial alpha blending within tiles [S1].

The screen is divided into $16 \times 16$ tiles [S1].

### Phase 1: Early Culling
This phase moves culling from the serial Render stage to the parallel Preprocess stage [S1]. It uses two techniques: Adaptive Radius and Axis-Aligned Bounding Box (AABB) [S1].

#### 1.1 Adaptive Radius Culling
For each Gaussian processed in parallel during the Preprocess stage:
1.  Compute the projected 2D covariance matrix $\Sigma'$ [S1].
2.  Define a minimum splatting opacity threshold $\alpha_{low}$, a predefined constant from the original 3DGS model [S1].
3.  Determine the region where the splatting opacity $\alpha$ satisfies $\alpha \ge \alpha_{low}$ [S1]. The splatting opacity is defined as:
    $$ \alpha = \sigma \exp\left(-\frac{1}{2} \mathbf{x}^T \Sigma'^{-1} \mathbf{x}\right) $$
    where $\sigma$ is the Gaussian opacity, $\mathbf{x}$ is the 2D distance vector from the Gaussian center to the pixel, and $\Sigma'$ is the projected covariance matrix [S1].
4.  The condition $\alpha \ge \alpha_{low}$ transforms into an ellipse inequality:
    $$ \mathbf{x}^T \Sigma'^{-1} \mathbf{x} \le 2 \ln\left(\frac{\sigma}{\alpha_{low}}\right) $$
    Expanding this into 2D coordinates $(x, y)$ yields the ellipse equation:
    $$ Ax^2 + By^2 + Cxy + D \le 0 $$
    where the coefficients are:
    $$ A = \Sigma'_{YY}, \quad B = \Sigma'_{XX}, \quad C = -2\Sigma'_{XY} $$
    $$ D = -2 \sqrt{\Sigma'_{XX} \Sigma'_{YY} - \Sigma'_{XY}^2} \ln\left(\frac{\sigma}{\alpha_{low}}\right) $$
    [S1]
5.  Calculate the **adaptive radius** $r_{ad}$ as the bounding circle radius of this ellipse [S1].
6.  Calculate the **original radius** $r_o$ based on the 99% confidence interval of the Gaussian [S1].
7.  Determine the final culling radius $r$:
    $$ r = \min(r_{ad}, r_o) $$
    This clamping ensures the rendering range does not exceed the standard confidence interval, which is critical when $\sigma$ is high or $\alpha_{low}$ is low [S1].
8.  Use $r$ to cull Gaussian-Tile pairs that do not overlap with the circle of radius $r$ centered at the projected Gaussian mean [S1].

#### 1.2 Axis-Aligned Bounding Box (AABB) Culling
To improve culling efficiency, particularly in the minor axis direction where circular culling is less effective, an AABB is computed for the ellipse defined in Step 1.4 [S1].
1.  Define the ellipse function $F(x, y) = Ax^2 + By^2 + Cxy + D$ [S1].
2.  Calculate the maximum x-coordinate ($x_{max}$) and maximum y-coordinate ($y_{max}$) of the ellipse [S1]. This involves finding the extrema of the ellipse along the coordinate axes [S1].
3.  Set the half-width $w = x_{max}$ and half-height $h = y_{max}$ [S1].
4.  Define the AABB centered at the projected Gaussian mean with dimensions $2w \times 2h$ [S1].
5.  Cull Gaussian-Tile pairs that do not overlap with this AABB [S1]. This provides more accurate culling in the minor axis direction than the circular adaptive radius alone [S1].

### Phase 2: Load Balancing
After early culling, the number of Gaussians to be rendered varies significantly across pixels, causing uneven thread loads in the Render stage [S1]. This phase balances the computational cost [S1].
1.  Quantify the load of each pixel thread, defined as the number of Gaussians required to be blended for that pixel [S1].
2.  Identify **heavy-load pixels** (those with a high number of Gaussians) and **light-load pixels** (those with a low number) [S1].
3.  For heavy-load pixels, reduce the number of rendered Gaussians to minimize thread waiting time [S1]. This is typically achieved by truncating the alpha blending process earlier or culling Gaussians with very low opacity contributions that would have negligible impact on the final color [G].
4.  For light-load pixels, increase the number of rendered Gaussians (if possible, e.g., by relaxing culling thresholds or including slightly further Gaussians) to hedge against rendering quality loss [S1].
5.  Render the pixels in parallel using the balanced set of Gaussians [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ad_r_gaussian_rasterize(gaussians, camera, image_width, image_height, alpha_low):
    """
    AdR-Gaussian rendering pipeline.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - pos: 3D position
            - cov: 3D covariance matrix
            - sigma: Opacity
            - sh: Spherical Harmonics coefficients
        camera: Camera object with projection and view matrices
        image_width: int
        image_height: int
        alpha_low: float, minimum splatting opacity threshold for culling
    Returns:
        image: Rendered image buffer
    """
    
    # Tile configuration
    TILE_SIZE = 16
    num_tiles_x = (image_width + TILE_SIZE - 1) // TILE_SIZE
    num_tiles_y = (image_height + TILE_SIZE - 1) // TILE_SIZE
    
    # Stage 1: Preprocess (Parallel per Gaussian)
    # Compute projected covariance and determine culling bounds
    culling_data = []
    for g in gaussians:
        # Project Gaussian to 2D
        # 1. Transform center to camera space
        center_cam = camera.world_to_camera(g.pos)
        
        # 2. Project covariance matrix to 2D (Jacobian projection)
        # J is the Jacobian of the perspective projection
        # Sigma_2d = J * R * Sigma_3d * R^T * J^T
        # (Assuming standard 3DGS projection logic)
        cov_2d = project_covariance(g.cov, center_cam, camera)
        
        # 3. Compute ellipse coefficients for alpha >= alpha_low
        # Ellipse: Ax^2 + By^2 + Cxy + D <= 0
        # Sigma_2d elements
        sxx = cov_2d[0, 0]
        syy = cov_2d[1, 1]
        sxy = cov_2d[0, 1]
        
        A = syy
        B = sxx
        C = -2 * sxy
        det = sxx * syy - sxy * sxy
        if det <= 0:
            # Degenerate Gaussian, skip or handle as point
            continue
            
        D = -2 * sqrt(det) * log(g.sigma / alpha_low)
        
        # 4. Compute Adaptive Radius (Bounding Circle of Ellipse)
        # The bounding circle radius of an ellipse Ax^2 + By^2 + Cxy + D <= 0
        # is determined by the maximum extent from the center.
        # For an ellipse, the semi-major axis length is the radius.
        # Eigenvalues of [[A, C/2], [C/2, B]] give 1/semi-axis^2
        # However, the source defines it as the bounding circle radius.
        # Standard approach: max(sqrt(lambda_max_inv)) where lambda are eigenvalues of the quadratic form matrix.
        # Matrix M = [[A, C/2], [C/2, B]]
        # Eigenvalues of M are lambda1, lambda2.
        # Semi-axes lengths are 1/sqrt(lambda1), 1/sqrt(lambda2).
        # Radius r_ad = max(1/sqrt(lambda1), 1/sqrt(lambda2))
        # Note: The source says "bounding circle radius of this ellipse".
        
        M = [[A, C/2], [C/2, B]]
        eigenvalues = compute_eigenvalues(M)
        # Eigenvalues should be positive for valid ellipse
        r_ad = 0
        if eigenvalues[0] > 0 and eigenvalues[1] > 0:
            r_ad = max(1.0 / sqrt(eigenvalues[0]), 1.0 / sqrt(eigenvalues[1]))
        else:
            # Fallback if eigenvalues are not positive (degenerate)
            r_ad = 0
            
        # 5. Compute Original Radius (99% confidence interval)
        # For 2D Gaussian, 99% confidence ellipse corresponds to chi-squared value ~5.991
        # Radius r_o is typically derived from the major axis of the 99% ellipse.
        # r_o = sqrt(5.991) * semi_major_axis
        semi_major = max(1.0 / sqrt(eigenvalues[0]), 1.0 / sqrt(eigenvalues[1]))
        r_o = sqrt(5.991) * semi_major
        
        # 6. Final Radius
        r = min(r_ad, r_o)
        
        # 7. Compute AABB for the ellipse
        # Max x and y on the ellipse Ax^2 + By^2 + Cxy + D = 0
        # To find max x, set dy/dx = 0 or use Lagrange multipliers.
        # Alternatively, find extrema of x on the ellipse.
        # The max x coordinate of the ellipse centered at origin is:
        # x_max = sqrt( -D / (A - C^2/(4B)) ) ? No, simpler to use eigen-decomposition or rotation.
        # Let's rotate the ellipse to align with axes.
        # Angle theta = 0.5 * atan2(C, A - B)
        # Rotated coefficients A', B'
        # x_max = sqrt(-D/A'), y_max = sqrt(-D/B')
        
        theta = 0.5 * atan2(C, A - B)
        cos_t = cos(theta)
        sin_t = sin(theta)
        
        # Rotate quadratic form
        # A' = A cos^2 + B sin^2 + C sin cos
        # B' = A sin^2 + B cos^2 - C sin cos
        A_rot = A * cos_t**2 + B * sin_t**2 + C * sin_t * cos_t
        B_rot = A * sin_t**2 + B * cos_t**2 - C * sin_t * cos_t
        
        # Check if A_rot and B_rot are positive
        if A_rot > 0 and B_rot > 0:
            x_max = sqrt(-D / A_rot)
            y_max = sqrt(-D / B_rot)
        else:
            # Fallback to r
            x_max = r
            y_max = r
            
        w = x_max
        h = y_max
        
        # Store culling data
        culling_data.append({
            'center_2d': project_point(center_cam, camera),
            'r': r,
            'w': w,
            'h': h,
            'sigma': g.sigma,
            'sh': g.sh,
            'cov_2d': cov_2d,
            'pos_cam': center_cam
        })
        
    # Stage 2: InclusiveSum & DuplicateWithKeys (Parallel per Gaussian)
    # Assign Gaussians to tiles based on AABB overlap
    # Create list of (GaussianIndex, TileIndex) pairs
    gaussian_tile_pairs = []
    for idx, data in enumerate(culling_data):
        cx, cy = data['center_2d']
        w, h = data['w'], data['h']
        
        # Determine tile range
        tile_min_x = max(0, floor((cx - w) / TILE_SIZE))
        tile_max_x = min(num_tiles_x - 1, ceil((cx + w) / TILE_SIZE))
        tile_min_y = max(0, floor((cy - h) / TILE_SIZE))
        tile_max_y = min(num_tiles_y - 1, ceil((cy + h) / TILE_SIZE))
        
        for ty in range(tile_min_y, tile_max_y + 1):
            for tx in range(tile_min_x, tile_max_x + 1):
                gaussian_tile_pairs.append((idx, ty * num_tiles_x + tx))
                
    # Stage 3: SortPairs & IdentifyTileRanges
    # Sort pairs by TileIndex, then by depth (z-coordinate of Gaussian in camera space)
    # Depth is used for alpha blending order (back-to-front)
    # Note: 3DGS typically sorts by depth.
    gaussian_tile_pairs.sort(key=lambda x: (x[1], culling_data[x[0]]['pos_cam'].z))
    
    # Identify ranges for each tile
    tile_ranges = {}
    for tile_idx in range(num_tiles_x * num_tiles_y):
        # Find start and end indices in the sorted list
        # (Implementation detail: binary search or linear scan)
        pass
        
    # Stage 4: Render (Pixel-level parallel) with Load Balancing
    image = zeros((image_height, image_width, 3))
    
    # Pre-calculate loads per pixel to balance
    # Count Gaussians per pixel
    pixel_loads = zeros((image_height, image_width), dtype=int)
    for idx, tile_idx in gaussian_tile_pairs:
        data = culling_data[idx]
        cx, cy = data['center_2d']
        # Determine which pixels this Gaussian affects
        # This is complex; typically done during rasterization.
        # For load balancing, we estimate based on tile counts or actual pixel coverage.
        # Simplified: Assume load is proportional to number of Gaussians in tile.
        pass
        
    # Load Balancing Strategy:
    # 1. Quantify variance of Gaussian counts per pixel.
    # 2. For heavy-load pixels, reduce number of Gaussians.
    # 3. For light-load pixels, increase number of Gaussians (if possible).
    
    # Implementation of Load Balancing:
    # Since the source describes reducing Gaussians for heavy-load pixels,
    # we can implement a threshold on the number of Gaussians to blend per pixel.
    # Or, more accurately, the source implies adjusting the set of Gaussians.
    # A practical implementation:
    # - Compute initial render.
    # - Identify pixels with high computational cost (many Gaussians).
    # - For these pixels, truncate the blending loop earlier (e.g., stop when alpha contribution < epsilon).
    # - For light pixels, ensure all Gaussians are processed.
    
    # Parallel Render Kernel
    def render_pixel(px, py, tile_idx):
        # Get Gaussians for this tile
        # Sort by depth (already done)
        # Alpha blend
        r, g, b = 0.0, 0.0, 0.0
        alpha_acc = 0.0
        
        # Get list of Gaussians for this pixel
        # This requires mapping Gaussian-Tile pairs to pixels
        # Standard 3DGS rasterizer does this efficiently.
        # Here we simulate the logic.
        
        gaussians_for_pixel = get_gaussians_for_pixel(px, py, gaussian_tile_pairs, culling_data)
        
        # Load Balancing Adjustment
        # If pixel is heavy load, limit number of Gaussians
        if is_heavy_load_pixel(px, py):
            gaussians_for_pixel = gaussians_for_pixel[:MAX_HEAVY_LOAD]
        else:
            # Light load, process all
            pass
            
        for g_data in gaussians_for_pixel:
            # Calculate splatting opacity alpha
            dx = px - g_data['center_2d'].x
            dy = py - g_data['center_2d'].y
            x_vec = [dx, dy]
            
            # Alpha = sigma * exp(-0.5 * x^T Sigma'^-1 x)
            inv_cov = inverse(g_data['cov_2d'])
            exponent = -0.5 * dot(x_vec, dot(inv_cov, x_vec))
            alpha = g_data['sigma'] * exp(exponent)
            
            if alpha < 1e-4: # Skip negligible contributions
                continue
                
            # Get color from SH
            color = sample_sh(g_data['sh'], dx, dy)
            
            # Alpha blending
            # C_out = C_in * (1 - alpha) + C_gauss * alpha
            # A_out = A_in + alpha * (1 - A_in)
            
            r = r * (1 - alpha) + color.r * alpha
            g = g * (1 - alpha) + color.g * alpha
            b = b * (1 - alpha) + color.b * alpha
            alpha_acc = alpha_acc + alpha * (1 - alpha_acc)
            
            if alpha_acc > 0.999: # Early exit if fully opaque
                break
                
        image[py, px] = [r, g, b]
        
    # Launch parallel render
    for ty in range(num_tiles_y):
        for tx in range(num_tiles_x):
            tile_idx = ty * num_tiles_x + tx
            # Process pixels in tile
            for py in range(ty * TILE_SIZE, min((ty + 1) * TILE_SIZE, image_height)):
                for px in range(tx * TILE_SIZE, min((tx + 1) * TILE_SIZE, image_width)):
                    render_pixel(px, py, tile_idx)
                    
    return image
```

### Parameters
*   **$\alpha_{low}$**: Predefined minimum splatting opacity constant used to define the lower bound for culling. Set in the original 3DGS model [S1].
*   **Tile Size**: $16 \times 16$ pixels [S1].

### Edge Cases and Pitfalls
*   **Adaptive Radius Clamping**: If the Gaussian opacity $\sigma$ is high or $\alpha_{low}$ is low, the adaptive radius $r_{ad}$ may exceed the original radius $r_o$ (based on the 99% confidence interval). In this case, the final radius $r$ must be clamped to $r_o$ to ensure the rendering range does not exceed the standard confidence interval [S1].
*   **Degenerate Gaussians**: If the projected covariance matrix is singular or near-singular, the ellipse coefficients may be invalid. The pseudocode includes a check for positive eigenvalues and a fallback to the radius $r$ [S1, G].

## Operating characteristics
*   **Time Complexity**: The Render stage ($E_p$) takes the largest proportion of computational cost (over 50%) in standard 3DGS due to serial alpha blending and complex splatting color calculation [S1]. AdR-Gaussian reduces this by moving culling to the parallel Preprocess stage and balancing loads, significantly reducing $E_p$ [S1].
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: On the Mip-NeRF360 dataset, AdR-Gaussian attains an average 590 FPS rendering speed on a single NVIDIA RTX3090 GPU [S1]. Experiments on three datasets demonstrate an average 310% acceleration in rendering speed [S1].
*   **Strengths**: Accelerates rendering without reducing the total number of Gaussians, thus avoiding quality loss associated with pruning methods [S1].
*   **Limitations**: Requires modification of the rasterization pipeline to include the Preprocess and Load Balancing stages [S1].

## Background
AdR-Gaussian was proposed to address the rendering speed bottlenecks of 3D Gaussian Splatting, specifically the serial culling in the Render stage and uneven load across pixel threads [S1]. It was published in 2024 [S1].

## Sources
[S1] AdR-Gaussian: Accelerating Gaussian Splatting with Adaptive Radius, https://doi.org/10.1145/3680528.3687675, fetched 2026-10-26
