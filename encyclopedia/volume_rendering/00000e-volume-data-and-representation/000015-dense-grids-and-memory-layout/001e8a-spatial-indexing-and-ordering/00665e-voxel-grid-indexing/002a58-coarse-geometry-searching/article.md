# Coarse Geometry Searching

Also called: Two-stage optimization with coarse voxel grid

## Overview
Coarse geometry searching is a two-stage optimization process where the first stage efficiently identifies the 3D areas of interest (geometry) using a low-resolution voxel grid with view-invariant color, thereby reducing the number of queried points for the fine detail stage [S1, S2]. It distinguishes itself from sibling methods like [Spatial Hashing](5002-spatial-hashing.md) by explicitly optimizing a dense, low-resolution grid to generate an occupancy mask for pruning, rather than relying on static hashing or sparse chunking structures [S1, S2].

## How it works
**Inputs:** Training images with known camera poses; hyperparameters $M^{(c)}$ (expected total voxels), $\alpha^{(\text{init})(c)}$ (initial alpha), $\delta^{(c)}$ (step size), and coarse resolution ratio (typically 1/4 of fine resolution) [S1, S2].
**Outputs:** An optimized coarse density voxel grid $V^{(\text{density})(c)}$ used to derive an occupancy mask; the coarse color grid $V^{(\text{rgb})(c)}$ is discarded after mask extraction [S1, S2].

**Data Structures:**
1.  **Coarse Density Grid:** $V^{(\text{density})(c)} \in \mathbb{R}^{1 \times N_x^{(c)} \times N_y^{(c)} \times N_z^{(c)}}$. Stores scalar density values.
2.  **Coarse Color Grid:** $V^{(\text{rgb})(c)} \in \mathbb{R}^{3 \times N_x^{(c)} \times N_y^{(c)} \times N_z^{(c)}}$. Stores view-invariant RGB colors.
3.  **Bounding Box (BBox):** Axis-aligned box tightly enclosing training camera frustums. Dimensions $L_x^{(c)}, L_y^{(c)}, L_z^{(c)}$ [S1].

**Step-by-Step Method:**

1.  **Grid Initialization:**
    *   Calculate voxel size $s^{(c)}$ based on the BBox volume and hyperparameter $M^{(c)}$:
        $$ s^{(c)} = \sqrt[3]{\frac{L_x^{(c)} \cdot L_y^{(c)} \cdot L_z^{(c)}}{M^{(c)}}} $$
    *   Compute grid dimensions $N_x^{(c)}, N_y^{(c)}, N_z^{(c)}$ as $\lfloor L_x^{(c)} / s^{(c)} \rfloor$, etc. [S1].
    *   Initialize all values in $V^{(\text{density})(c)}$ to 0 [S1].
    *   Set the bias $b$ for the softplus activation function to ensure initial transmittance is near 1:
        $$ b = \log \left( (1 - \alpha^{(\text{init})(c)})^{-\frac{1}{s^{(c)}}} - 1 \right) $$
        where $\alpha^{(\text{init})(c)}$ is a hyperparameter [S1].

2.  **Prior 2 (Learning Rate Scaling):**
    *   For each grid point $j$, count the number of training views $n_j$ to which it is visible.
    *   Determine $n_{\max}$, the maximum view count across all grid points.
    *   Scale the base learning rate for grid point $j$ by factor $n_j / n_{\max}$ [S1].

3.  **Coarse Training:**
    *   Train the coarse model for a fixed number of epochs (e.g., 5 epochs) [S2].
    *   **Sampling:** For each ray, sample points $x_i$ using step size $\delta^{(c)}$:
        $$ x_0 = o + t^{(\text{near})} d $$
        $$ x_i = x_0 + i \cdot \delta^{(c)} \cdot \frac{d}{\|d\|^2} $$
        where $o$ is camera center, $d$ is ray direction, and $t^{(\text{near})}$ is the near bound [S1].
    *   **Query:** Interpolate density $\dot{\sigma}^{(c)}$ and color $\mathbf{c}^{(c)}$ from $V^{(\text{density})(c)}$ and $V^{(\text{rgb})(c)}$ at query points $x_i$.
    *   **Rendering:** Compute rendered color $\hat{C}(\mathbf{r})$ using volume rendering:
        $$ \alpha_i = 1 - \exp(-\dot{\sigma}^{(c)} \delta_i) $$
        $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^K T_i \alpha_i \mathbf{c}_i + T_{K+1} \mathbf{c}_{\text{bg}} $$
        where $\delta_i$ is the distance to adjacent points, and $\mathbf{c}_{\text{bg}}$ is background color [S1].
    *   **Loss:** Minimize photometric Mean Squared Error (MSE) between $\hat{C}(\mathbf{r})$ and observed colors, potentially with background entropy regularization [S1].

4.  **Mask Extraction and Discard:**
    *   Extract an occupancy mask from the density outputs of the trained coarse model in an unsupervised manner (e.g., thresholding density or alpha) [S2].
    *   Discard the weights and features of the coarse model ($V^{(\text{density})(c)}$ and $V^{(\text{rgb})(c)}$) [S2].
    *   Use the mask to prune empty space in subsequent fine-stage optimization [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_geometry_searching(training_images, camera_poses, M_c, alpha_init_c, coarse_res_ratio, num_epochs=5):
    # 1. Setup BBox and Grid Dimensions
    bbox = compute_tight_bbox(camera_poses)
    L_x, L_y, L_z = bbox.dimensions
    voxel_size = (L_x * L_y * L_z / M_c) ** (1/3)
    N_x = floor(L_x / voxel_size)
    N_y = floor(L_y / voxel_size)
    N_z = floor(L_z / voxel_size)
    
    # 2. Initialize Grids
    V_density = zeros((1, N_x, N_y, N_z))
    V_rgb = zeros((3, N_x, N_y, N_z))
    
    # Calculate bias for softplus to ensure initial transmittance ~ 1
    bias = log((1 - alpha_init_c)**(-1/voxel_size) - 1)
    
    # 3. Compute View Counts for Learning Rate Prior
    view_counts = compute_view_visibility(V_density, camera_poses)
    n_max = max(view_counts)
    lr_scales = view_counts / n_max
    
    # 4. Training Loop
    for epoch in range(num_epochs):
        for image, pose in zip(training_images, camera_poses):
            rays = generate_rays(image, pose)
            rendered_colors = []
            target_colors = []
            
            for ray in rays:
                # Sample points along ray
                points = sample_ray(ray, voxel_size, delta_c=voxel_size) # delta_c is hyperparameter
                
                # Query grid
                densities = interpolate_3d(V_density, points)
                colors = interpolate_3d(V_rgb, points)
                
                # Apply activation with bias
                # Assuming softplus activation: sigma = softplus(density + bias)
                sigmas = softplus(densities + bias)
                
                # Volume Render
                alphas = 1 - exp(-sigmas * ray.step_size)
                transmittances = cumulative_product(1 - alphas[:-1])
                # Handle first point transmittance
                T = [1.0] + list(transmittances)
                
                rendered = sum(T[i] * alphas[i] * colors[i] for i in range(len(points)))
                rendered_colors.append(rendered)
                target_colors.append(image.pixel_color)
            
            # Compute Loss (MSE)
            loss = mean_squared_error(rendered_colors, target_colors)
            
            # Backpropagate with scaled learning rates
            # Note: Implementation detail [G]: Apply lr_scales to gradients of V_density elements
            update_gradients(V_density, V_rgb, loss, lr_scales)
            
    # 5. Extract Mask and Discard
    occupancy_mask = extract_occupancy_mask(V_density) # e.g., threshold density > 0
    # Discard coarse grids
    V_density = None
    V_rgb = None
    
    return occupancy_mask
```

## Operating characteristics
*   **Time Complexity:** The coarse stage is significantly faster than the fine stage due to the low resolution (1/4 linear dimension, 1/64 volume) and view-invariant color [S2]. It typically runs for a fixed small number of epochs (e.g., 5) [S2].
*   **Memory Complexity:** Requires storing two dense grids ($V^{(\text{density})(c)}$ and $V^{(\text{rgb})(c)}$) at coarse resolution. Memory is discarded after mask extraction [S1, S2].
*   **Strengths:** Reduces the number of queried points in the fine stage by pruning empty space, leading to faster convergence and lower memory usage for the subsequent fine model [S1, S2]. The view-count-based learning rate prior prevents overfitting to sparse views [S1].
*   **Limitations:** The coarse grid is dense, which may be inefficient for very large scenes compared to sparse methods like [Bonxai](16862-bonxai.md) or [Sparse Voxel Storage](15595-sparse-voxel-storage-and-chunking.md). It assumes a uniform voxel size within the BBox [S1].
*   **When to use:** When rapid convergence is needed and the scene fits within a manageable bounding box. It is a prerequisite for methods that use explicit grid features in a two-stage pipeline [S2].

## Background
Coarse geometry searching was introduced in "Direct Voxel Grid Optimization" (DVGO) by Sun et al. (CVPR 2022) [S1]. It was subsequently adopted and adapted in "NeRFLight" for fast and light neural radiance fields [S2]. The method addresses the problem of slow convergence and overfitting in empty space common in early voxel-based NeRF optimizations [S1, S2].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] NeRFLight: Fast and Light Neural Radiance Fields using a ..., https://www.iri.upc.edu/files/scidoc/2861-NeRFLight%3A-Fast-and-Light-Neural-Radiance-Fields-using-a-Shared-Feature-Grid.pdf, fetched 2026-10-27
