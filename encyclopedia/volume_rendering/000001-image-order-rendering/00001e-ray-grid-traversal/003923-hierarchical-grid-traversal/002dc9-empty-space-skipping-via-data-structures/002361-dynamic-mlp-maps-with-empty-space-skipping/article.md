# Dynamic MLP Maps with Empty Space Skipping

Also called: Dynamic MLP Maps

## Overview
Dynamic MLP Maps is a rendering technique for volumetric videos that combines multi-level hash tables with 2D CNN-generated MLP maps and uses an occupancy volume for empty space skipping to achieve real-time performance [S1, unofficial]. It differs from its siblings by using a hybrid representation where spatial features are stored in hash tables and network parameters are predicted by a CNN decoder onto orthogonal planes, allowing for efficient evaluation of multiple small MLPs [S1, unofficial]. Unlike [Object-Order Empty Space Skipping](865-object-order-empty-space-skipping.md) which operates on geometry, or [Space Leaping](5863-space-leaping.md) which uses fixed grid strides, this method dynamically skips empty space using a pre-computed per-frame occupancy volume [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of video frames (volumetric video), camera poses, and an optional foreground mask.
*   **Outputs:** Rendered color images and densities for each frame.

**Data Structures:**
1.  **Multi-level Hash Tables:** Stores high-dimensional feature vectors for spatial locations. The specific number of levels and hash table sizes are not given in the sources [G: Typically follows KiloNeRF/Instant-NGP conventions, but exact parameters are not provided in [S1, unofficial]].
2.  **MLP Maps:** Three 2D maps (one for each axis-aligned plane: XY, YZ, XZ) predicted by a 2D CNN decoder. Each map stores parameters for a small MLP network. The resolution of these maps is not given in the sources [S1, unofficial].
3.  **Occupancy Volume:** A 3D grid representing the spatial extent of non-empty voxels for each video frame.
    *   **Layout:** A 3D array of booleans or integers.
    *   **Resolution:** $24 \times 24 \times 48$ per video frame [S1, unofficial].
    *   **Construction:** Each volume cell is divided into a $5 \times 5 \times 5$ subgrid. Density is calculated for each subgrid point. A volume cell is marked as occupied if any subgrid point has density above threshold $\tau_1 = 5$ [S1, unofficial].

**Numbered Steps:**

1.  **Pre-computation (Training/Offline):**
    1.  Train the hash tables and CNN decoder to predict MLP maps using the training loss function.
    2.  For each video frame, compute the occupancy volume by sampling densities and applying the threshold $\tau_1$ [S1, unofficial].

2.  **Inference Sampling (Per Ray):**
    1.  Define the scene bound diagonal distance $D$.
    2.  Set sampling step size $\delta = D / 256$ [S1, unofficial].
    3.  For a given camera ray $\mathbf{r}$, determine the intersection with the occupancy volume.
    4.  Identify the set of occupied voxels along the ray.
    5.  Sample query points $\mathbf{p}_i$ only within these occupied voxels, spaced by $\delta$ [S1, unofficial].

3.  **Feature Retrieval and MLP Evaluation:**
    1.  For each query point $\mathbf{p}_i$:
        a.  **Hash Embedding:** Embed $\mathbf{p}_i$ into a high-dimensional feature vector $\mathbf{f}_i$ using the multi-level hash tables [S1, unofficial].
        b.  **MLP Map Projection:** Project $\mathbf{p}_i$ onto the three axis-aligned orthogonal MLP maps (XY, YZ, XZ) to retrieve corresponding MLP parameters $\theta_{xy}, \theta_{yz}, \theta_{xz}$ [S1, unofficial].
        c.  **MLP Evaluation:** Evaluate three small MLP networks using $\mathbf{f}_i$ and each set of parameters $\theta$. This is done efficiently using a custom PyTorch layer based on MAGMA, following KiloNeRF [S1, unofficial].
        d.  **Result Combination:** Combine the predictions (color $\mathbf{c}_i$ and density $\sigma_i$) from each MLP map via elementwise summation [S1, unofficial].

4.  **Filtering and Rendering:**
    1.  Compute accumulated transmittance $T_i$ and composition weight $\omega_i$ for each sample:
        $$
        \omega_i = T_i(1 - \exp(-\sigma_i \delta_i))
        $$
        $$
        T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)
        $$
        where $\delta_i$ is the ray step size [S1, unofficial].
    2.  **Filtering:** If $\omega_i < \tau_2$ (where $\tau_2 = 10^{-3}$), ignore the query point for color evaluation [S1, unofficial].
    3.  **Composition:** Compute the final pixel color $\tilde{C}(\mathbf{r})$:
        $$
        \tilde{C}(\mathbf{r}) = \sum_{i=1}^M \omega_i \mathbf{c}_i
        $$
        where $M$ is the number of valid samples [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

# Constants from sources
OCCUPANCY_RES = (24, 24, 48)
SUBGRID_SIZE = 5
THRESHOLD_DENSITY = 5.0
THRESHOLD_WEIGHT = 1e-3
STEP_DIVISOR = 256

def compute_occupancy_volume(scene_bound, density_function, resolution=OCCUPANCY_RES):
    """
    Pre-compute occupancy volume for a frame.
    density_function: callable that returns density for a point.
    """
    # Create grid
    x = torch.linspace(scene_bound.min.x, scene_bound.max.x, resolution[0])
    y = torch.linspace(scene_bound.min.y, scene_bound.max.y, resolution[1])
    z = torch.linspace(scene_bound.min.z, scene_bound.max.z, resolution[2])
    xx, yy, zz = torch.meshgrid(x, y, z, indexing='ij')
    
    # Create subgrid points
    # This is a simplified representation; actual implementation iterates cells
    occupied = torch.zeros(resolution, dtype=torch.bool)
    
    for i in range(resolution[0]):
        for j in range(resolution[1]):
            for k in range(resolution[2]):
                # Center of the cell
                center = torch.tensor([xx[i,j,k], yy[i,j,k], zz[i,j,k]])
                is_occupied = False
                # Check subgrid points
                for dx in range(SUBGRID_SIZE):
                    for dy in range(SUBGRID_SIZE):
                        for dz in range(SUBGRID_SIZE):
                            sub_point = center + torch.tensor([dx, dy, dz]) * (scene_bound.size / resolution)
                            if density_function(sub_point) > THRESHOLD_DENSITY:
                                is_occupied = True
                                break
                        if is_occupied: break
                    if is_occupied: break
                occupied[i,j,k] = is_occupied
    return occupied

def get_hash_features(query_points, hash_tables):
    """
    Retrieve features from multi-level hash tables.
    query_points: (N, 3) tensor of coordinates.
    """
    # Implementation depends on specific hash table structure (e.g., KiloNeRF/Instant-NGP)
    # Not fully specified in [S1, unofficial], so we assume a standard lookup
    features = []
    for level in hash_tables:
        # Hash and lookup logic [G: Standard hash grid implementation]
        feat = level.lookup(query_points)
        features.append(feat)
    return torch.cat(features, dim=-1)

def get_mlp_map_params(query_points, mlp_maps):
    """
    Retrieve MLP parameters from 2D CNN-generated maps.
    mlp_maps: List of 3 tensors (XY, YZ, XZ) containing MLP weights/biases.
    """
    # Project query points to 2D planes and interpolate parameters
    # This requires a custom layer or interpolation logic [S1, unofficial]
    params_xy = interpolate_2d(mlp_maps[0], query_points[:, [0, 1]])
    params_yz = interpolate_2d(mlp_maps[1], query_points[:, [1, 2]])
    params_xz = interpolate_2d(mlp_maps[2], query_points[:, [0, 2]])
    return params_xy, params_yz, params_xz

def evaluate_mlp_maps(features, params_xy, params_yz, params_xz):
    """
    Evaluate multiple small MLPs and sum results.
    """
    # Custom PyTorch layer based on MAGMA [S1, unofficial]
    # Assuming a function that applies MLP weights to features
    color_xy, density_xy = apply_mlp(features, params_xy)
    color_yz, density_yz = apply_mlp(features, params_yz)
    color_xz, density_xz = apply_mlp(features, params_xz)
    
    # Elementwise summation
    color = color_xy + color_yz + color_xz
    density = density_xy + density_yz + density_xz
    return color, density

def render_ray(ray_origin, ray_direction, scene_bound, occupancy_volume, 
               hash_tables, mlp_maps, step_divisor=STEP_DIVISOR):
    """
    Render a single ray using Dynamic MLP Maps with Empty Space Skipping.
    """
    # 1. Determine sampling interval
    diagonal = scene_bound.diagonal()
    step_size = diagonal / step_divisor
    
    # 2. Find occupied segments using occupancy volume
    # [G: Ray casting against 3D grid to find entry/exit and occupied cells]
    occupied_points = []
    current_t = 0.0
    max_t = diagonal # Simplified bound
    
    # Iterate along ray
    while current_t < max_t:
        point = ray_origin + ray_direction * current_t
        # Check if point is in occupied voxel
        voxel_coords = world_to_voxel(point, occupancy_volume.shape, scene_bound)
        if is_in_bounds(voxel_coords) and occupancy_volume[voxel_coords]:
            occupied_points.append(point)
        current_t += step_size
        
    if not occupied_points:
        return 0.0, 0.0 # Background

    # 3. Evaluate MLPs at occupied points
    points_tensor = torch.stack(occupied_points)
    features = get_hash_features(points_tensor, hash_tables)
    params_xy, params_yz, params_xz = get_mlp_map_params(points_tensor, mlp_maps)
    colors, densities = evaluate_mlp_maps(features, params_xy, params_yz, params_xz)
    
    # 4. Volume Rendering with Filtering
    transmittance = 1.0
    accumulated_color = torch.zeros(3)
    
    for i in range(len(points_tensor)):
        sigma = densities[i]
        delta = step_size
        alpha = 1.0 - torch.exp(-sigma * delta)
        weight = transmittance * alpha
        
        if weight < THRESHOLD_WEIGHT:
            # Filter low-weight points
            transmittance *= (1.0 - alpha)
            continue
            
        accumulated_color += weight * colors[i]
        transmittance *= (1.0 - alpha)
        
    return accumulated_color, 1.0 - transmittance
```

**Parameters:**
*   **Occupancy Volume Resolution:** $24 \times 24 \times 48$ per video frame [S1, unofficial].
*   **Density Threshold ($\tau_1$):** 5. Used to determine if a volume cell is occupied during occupancy volume construction [S1, unofficial].
*   **Weight Threshold ($\tau_2$):** $10^{-3}$. Sampling points with composition weight below this are ignored [S1, unofficial].
*   **Sampling Step Size:** $1/256$th of the diagonal distance of the scene bound [S1, unofficial].
*   **KL Loss Weight ($\lambda_{KL}$):** $10^{-6}$ [S1, unofficial].
*   **Mask Loss Weight:** 0.1 (optional, for foreground-only scenes) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Detailed Content:** The method struggles to render very detailed content, such as text and fingers [S1, unofficial].
*   **Precision:** The use of a coarse occupancy volume ($24 \times 24 \times 48$) may lead to aliasing or missing thin structures if the scene bound is large [G: Inferred from resolution].
*   **Boundary Conditions:** Ray sampling must be carefully bounded to the scene extent to avoid out-of-bounds memory access in hash tables and MLP maps [G: Standard practice].

## Subtopics

### Hash Table Feature Storage
Stores spatial features in a multi-level hash grid. Sets itself apart by providing high-resolution feature access with low memory overhead compared to dense feature grids. Choose when memory is constrained but high-frequency detail is needed. See [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) for comparison on grid-based storage.

### CNN-Generated MLP Maps
Uses a 2D CNN to predict network parameters onto 2D planes. Sets itself apart from feature maps by enabling faster rendering with smaller MLPs. Choose when rendering speed is critical and the scene has smooth variations. See [Space Leaping](5863-space-leaping.md) for comparison on acceleration structures.

## Operating characteristics
*   **Time Complexity:** Rendering is accelerated by empty space skipping, reducing the number of samples per ray. The evaluation of multiple small MLPs is faster than a single large MLP. The method achieves real-time performance [S1, unofficial].
*   **Memory Complexity:** Storage is significantly reduced compared to per-frame models. For a 300-frame video, it requires 240 MB, compared to 9 GB for per-frame KiloNeRF models [S1, unofficial].
*   **Measured Performance:**
    *   **NHR Dataset (sport1):** PSNR 32.92, SSIM 0.959, LPIPS 0.067 [S1, unofficial].
    *   **ZJU-MoCap Dataset (scene 313):** PSNR 32.15, SSIM 0.976, LPIPS 0.049 [S1, unofficial].
*   **Scaling:** Training time and storage scale linearly with the number of frames for per-frame methods, but this method maintains constant storage and low training time (16 hours for 300 frames) [S1, unofficial].
*   **Strengths:** Real-time rendering, low storage, high quality compared to baselines like Neural Volumes, C-NeRF, D-NeRF, and DyNeRF [S1, unofficial].
*   **Limitations:** Difficulty rendering very detailed content (text, fingers) [S1, unofficial].
*   **When to Use:** Real-time volumetric video playback where storage and training time are constraints.
*   **When Not to Use:** Scenes with extremely fine, high-frequency details that the hash table and MLP map resolution cannot capture.

## Background
Dynamic MLP Maps was introduced by Sida Peng, Yunzhi Yan, Qing Shuai, Hujun Bao, and Xiaowei Zhou from the State Key Lab of CAD&CG, Zhejiang University [S1, unofficial]. It is designed to represent volumetric videos efficiently, addressing the limitations of storing per-frame neural radiance fields. The method builds on [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md) by using an occupancy volume to skip empty space, and on [Space Leaping](5863-space-leaping.md) by using a grid-based structure for acceleration. It improves upon [Object-Order Empty Space Skipping](865-object-order-empty-space-skipping.md) by integrating the skipping directly into the neural rendering pipeline for dynamic content [S1, unofficial].

## Sources
[S1] PDF Supplementary Material for Representing Volumetric Videos as Dynamic ..., https://pengsida.net/project_page_assets/mlp_maps/supplementary_material.pdf, fetched 2026-10-29 (unofficial)
