# Surface-Constrained Gaussian Splatting Training

Also called: surface splat training

## Overview
A training procedure for 3D Gaussian Splatting that initializes splats from a mesh surface or point cloud, estimates normals, and optimizes splat parameters using surface constraints (depth, normal, flatness) and adaptive spawning strategies. It differs from standard Gaussian Splatting by enforcing geometric fidelity to input surface data during optimization, rather than relying solely on photometric reconstruction.

## How it works
**Inputs:**
- Mesh data (vertices, faces, vertex colors) or point cloud data.
- COLMAP dataset directory.
- Training parameters (see Parameters).

**Outputs:**
- Optimized Gaussian splat parameters (means, quaternions, scales, opacities, spherical harmonics coefficients).

**Data Structures:**
- `splats`: Dictionary containing tensors for `means` ($N \times 3$), `quats` ($N \times 4$), `scales` ($N \times 3$, stored as log-scale), `opacities` ($N \times 1$, stored as logit), and `sh_coeffs` ($N \times K$).
- `anchor_pts`: Initial set of points from mesh vertices or subsampled point cloud.
- `splat_normals`: Per-splat surface normals ($N \times 3$).
- `cloud_pts_full`, `cloud_normals_full`, `cloud_colors_full`: Reservoirs for adaptive spawning.

**Parameters:**
- `iterations`: Total training steps. Default: 3000 [S1, unofficial].
- `n_samples`: Number of points to sample/subsample. Default: 50000 [S1, unofficial].
- `anchor_weight_start`: Initial anchor loss weight. Default: 0.1 [S1, unofficial].
- `anchor_weight_end`: Final anchor loss weight. Default: 0.001 [S1, unofficial].
- `depth_lambda`: Depth loss weight. Default: 0.5 [S1, unofficial].
- `aniso_lambda`: Anisotropy loss weight. Default: 0.0 [S1, unofficial].
- `flatness_lambda`: Flatness loss weight. Default: 0.01 [S1, unofficial].
- `normal_lambda`: Normal loss weight. Default: 0.05 [S1, unofficial].
- `opacity_decay`: Opacity decay rate. Default: 0.001 [S1, unofficial].
- `prune_threshold`: Minimum opacity for pruning. Default: 0.002 [S1, unofficial].
- `max_scale`: Maximum allowed scale. Default: 2.0 [S1, unofficial].
- `target_splats`: Target splat count for MCMC. Default: 0 (uses cap 500000) [S1, unofficial].
- `strategy_name`: Spawning strategy ('simple', 'mcmc', 'absgrad', 'mrnf', 'adaptive'). Default: 'simple' [S1, unofficial].
- `multi_view`: Use multiple views per step. Default: False [S1, unofficial].
- `multi_view_count`: Views per step if `multi_view` is True. Default: 2 [S1, unofficial].
- `smooth_strength`: Smoothing strength. Default: 0.0 [S1, unofficial].
- `coverage_lambda`: Coverage loss weight. Default: 0.5 [S1, unofficial].

**Steps:**

1. **Initialization:**
   - If mesh data is provided, sample surface points: `points, normals, scolors = sample_mesh_surface(verts, faces, colors_mesh, n_samples)`. Set `anchor_pts = verts` [S1, unofficial].
   - If point cloud is provided, subsample if `len(cloud_pts) > n_samples` using random choice; otherwise use all points [S1, unofficial].
   - Estimate normals from point cloud using KNN ($k=8$): for each point, compute centered neighbors, perform SVD, take smallest singular vector as normal [S1, unofficial].
   - Normalize normals: `normals /= (np.linalg.norm(normals, axis=-1, keepdims=True) + 1e-8)` [S1, unofficial].
   - Build anchor KD-tree: `anchor_tree = cKDTree(anchor_pts)` [S1, unofficial].
   - Load COLMAP dataset: `images, _, _ = load_colmap_dataset(colmap_dir, max_resolution=max_resolution)` [S1, unofficial].
   - Compute scene scale: `scene_scale = float(np.median(pdist(cam_centers)))` if `len(cam_centers) > 1` else `1.0` [S1, unofficial].
   - Compute depth priors: render depth maps from all points (`anchor_pts + points`) for each image [S1, unofficial].
   - Initialize splats: `splats = init_surface_splats(points, normals, scolors, device)` [S1, unofficial].
   - Create optimizers: `optimizers = create_optimizers(splats, scene_scale)` [S1, unofficial].
   - Compute max allowed scale: `max_scale_abs = float(np.median(init_knn) * 3.0)`, where `init_knn = knn_distances(points, k=3)` [S1, unofficial].
   - Store per-splat surface normals: `splat_normals = torch.from_numpy(normals / norm(normals))` [S1, unofficial].
   - Set up means scheduler: `means_scheduler = optim.lr_scheduler.ExponentialLR(optimizers['means'], gamma=0.01 ** (1.0 / iterations))` [S1, unofficial].
   - Initialize cloud reservoir: `cloud_pts_full = points.copy()`, `cloud_normals_full = n_norm.copy()`, `cloud_colors_full = scolors.copy()` [S1, unofficial].
   - Setup strategy based on `strategy_name`:
     - 'mcmc': Uses `MCMCStrategy` with `cap_max`, `noise_lr`, refine intervals [S1, unofficial].
     - 'absgrad': Uses `DefaultStrategy` with `absgrad=True`, `grow_grad2d=0.0008`, `grow_scale3d=0.01`, `prune_opa`, refine intervals, `reset_every` [S1, unofficial].
     - 'mrnf', 'adaptive', 'simple': Print strategy name [S1, unofficial].
   - Prepare image tensors: `img['pixels_t']`, `img['K_t']`, `img['w2c_t']` converted to torch tensors on device [S1, unofficial].
   - Yield initial state with splat means, colors, scales [S1, unofficial].

2. **Training Loop (for each step):**
   - Compute anchor weight decay: `anchor_w = anchor_weight_start + (anchor_weight_end - anchor_weight_start) * (step / max(iterations - 1, 1))` [S1, unofficial].
   - Extract splat parameters: `means`, `quats`, `scales` (exp of `splats['scales']`), `opacities` (sigmoid of `splats['opacities']`) [S1, unofficial].
   - Clamp scales: `splats['scales'].data.clamp_(min=-10.0, max=math.log(max_scale_abs))` [S1, unofficial].
   - For each selected view: handle random cropping if not `multi_view` and `CROP > 0` and `step % 50 != 0` [S1, unofficial].
   - Render Gaussians using `gsplat` if available, passing `sh_coeffs`, `viewmat`, `K`, `W`, `H`, `bg_color`, `sh_degree`, `absgrad`, `return_meta` [S1, unofficial].
   - If using `gsplat` strategy and `need_meta`, extract `view_info` from first view result [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_surface_constrained_gaussians(mesh_data, point_cloud, colmap_dir, iterations=3000, n_samples=50000, strategy_name='simple'):
    # 1. Initialization
    if mesh_data is not None:
        points, normals, scolors = sample_mesh_surface(mesh_data.verts, mesh_data.faces, mesh_data.colors, n_samples)
        anchor_pts = mesh_data.verts
    else:
        cloud_pts = point_cloud.points
        if len(cloud_pts) > n_samples:
            indices = np.random.choice(len(cloud_pts), n_samples, replace=False)
            points = cloud_pts[indices]
            normals = point_cloud.normals[indices]
            scolors = point_cloud.colors[indices]
        else:
            points = cloud_pts
            normals = point_cloud.normals
            scolors = point_cloud.colors

    # Estimate normals if not provided
    if normals is None:
        k = 8
        neighbors = knn(points, k=k)
        centered_neighbors = neighbors - points[:, np.newaxis, :]
        # SVD on centered neighbors to get normals
        normals = estimate_normals_svd(centered_neighbors)
    
    # Normalize normals
    norms = np.linalg.norm(normals, axis=-1, keepdims=True)
    normals = normals / (norms + 1e-8)

    # Build anchor KD-tree
    anchor_tree = cKDTree(anchor_pts)

    # Load dataset
    images, cam_centers, _ = load_colmap_dataset(colmap_dir)
    
    # Compute scene scale
    if len(cam_centers) > 1:
        scene_scale = float(np.median(pdist(cam_centers)))
    else:
        scene_scale = 1.0

    # Compute depth priors
    depth_maps = render_depth_maps(anchor_pts + points, images)

    # Initialize splats
    splats = init_surface_splats(points, normals, scolors, device='cuda')
    
    # Create optimizers
    optimizers = create_optimizers(splats, scene_scale)

    # Compute max allowed scale
    init_knn = knn_distances(points, k=3)
    max_scale_abs = float(np.median(init_knn) * 3.0)

    # Store per-splat surface normals
    splat_normals = torch.from_numpy(normals / np.linalg.norm(normals, axis=-1, keepdims=True))

    # Set up means scheduler
    means_scheduler = optim.lr_scheduler.ExponentialLR(optimizers['means'], gamma=0.01 ** (1.0 / iterations))

    # Initialize cloud reservoir
    cloud_pts_full = points.copy()
    cloud_normals_full = normals.copy()
    cloud_colors_full = scolors.copy()

    # Setup strategy
    if strategy_name == 'mcmc':
        strategy = MCMCStrategy(cap_max=500000, noise_lr=0.01, refine_intervals=[1000, 2000])
    elif strategy_name == 'absgrad':
        strategy = DefaultStrategy(absgrad=True, grow_grad2d=0.0008, grow_scale3d=0.01, prune_opa=0.002, refine_intervals=[1000, 2000], reset_every=1000)
    else:
        print(f"Strategy {strategy_name} not fully implemented in pseudocode")
        strategy = DefaultStrategy()

    # Training Loop
    for step in range(iterations):
        # Compute anchor weight decay
        anchor_w = 0.1 + (0.001 - 0.1) * (step / max(iterations - 1, 1))

        # Extract splat parameters
        means = splats['means']
        quats = splats['quats']
        scales = torch.exp(splats['scales'])
        opacities = torch.sigmoid(splats['opacities'])

        # Clamp scales
        splats['scales'].data.clamp_(min=-10.0, max=math.log(max_scale_abs))

        # Select views
        if not multi_view:
            view_indices = [random.choice(len(images))]
        else:
            view_indices = random.sample(len(images), multi_view_count)

        for view_idx in view_indices:
            img = images[view_idx]
            # Handle random cropping
            if not multi_view and CROP > 0 and step % 50 != 0:
                img = random_crop(img, CROP)
            
            # Prepare tensors
            pixels_t = torch.from_numpy(img.pixels).to(device)
            K_t = torch.from_numpy(img.K).to(device)
            w2c_t = torch.from_numpy(img.w2c).to(device)

            # Render
            rendered_img, meta = gsplat_render(
                splats['sh_coeffs'], 
                means, 
                quats, 
                scales, 
                opacities, 
                w2c_t, 
                K_t, 
                img.W, 
                img.H, 
                bg_color=0.0, 
                sh_degree=3, 
                absgrad=True, 
                return_meta=True
            )

            # Compute losses
            loss = compute_loss(rendered_img, pixels_t, anchor_w, depth_maps, splats, strategy)
            
            # Backpropagate
            optimizers.zero_grad()
            loss.backward()
            optimizers.step()

        # Update scheduler
        means_scheduler.step()

        # Apply strategy (spawn/prune)
        strategy.apply(splats, meta, step)

    return splats
```

## Subtopics

### [Geometry-aware Gaussian Splatting](12476-geometry-aware-gaussian-splatting.md)
Uses geometric priors during training but may not initialize from surface data. Choose when surface data is unavailable but geometry is known.

### [Mask and Weight Filtering for Patch Points](16626-mask-and-weight-filtering-for-patch-points.md)
Filters splats based on patch weights. Choose when dealing with patch-based representations.

## Operating characteristics
- **Time Complexity:** Depends on the number of splats $N$ and views $V$. Rendering is $O(N \cdot V)$. Spawning/pruning adds overhead proportional to $N$.
- **Memory Complexity:** $O(N)$ for splat parameters.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with number of splats and views. Adaptive strategies help manage $N$.
- **Strengths:** High geometric fidelity to input surface.
- **Limitations:** Requires high-quality input mesh/point cloud. Sensitive to initialization.
- **When to use:** When surface geometry is known and high fidelity is required.
- **When not to use:** When only images are available without surface data.

## Background
Originated in the `dust3r-3dgs` repository [S1, unofficial]. It extends 3D Gaussian Splatting to incorporate surface constraints for better geometric reconstruction.

## Sources
[S1] SpaceCollectors/dust3r-3dgs, https://github.com/SpaceCollectors/dust3r-3dgs, fetched 2026-10-27 (unofficial)
