# Voxel Carving

Also called: Carve Voxelization

## Overview
Voxel Carving is a technique that creates a dense, filled voxel grid of a given shape by intersecting the viewing frustums of multiple cameras, unlike surface-only voxelization methods [S1]. It reconstructs a volumetric representation from 2D depth maps or silhouettes by iteratively marking voxels as occupied based on visibility constraints [S1]. It differs from [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md) and [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md) by producing an explicit, dense grid rather than a boundary representation, and from [Multi-Plane Projection Voxelization and Aggregation](12100-multi-plane-projection-voxelization-and-aggregation.md) by directly modifying voxel occupancy states per view [S1].

## How it works
**Inputs:**
*   Depth maps (float values representing distance from camera center) or silhouette masks (binary) for each camera pose [S1, S2].
*   Camera poses (extrinsic matrices) and intrinsic parameters [S1, S2].
*   Grid definition: `width`, `height`, `depth` (in voxels), `voxel_size`, and `origin` (bounding box corner) [S1].

**Outputs:**
*   A dense voxel grid where each voxel is marked as occupied or empty [S1].

**Data Structures:**
*   **Voxel Grid:** A dense 3D array initialized with specified `width`, `height`, `depth`, `voxel_size`, `origin`, and optional `color` [S1]. The physical coordinate $P$ of a voxel center is calculated from its index relative to the `origin` and `voxel_size` [S1].

**Steps:**
1.  **Initialize Grid:** Create a dense voxel grid with specified `width`, `height`, `depth`, `voxel_size`, `origin`, and `color` using `create_dense` [S1].
2.  **Preprocess Model:** Center the model at the origin and scale it such that the bounding box norm is 2.0 (radius 1.0) [S2].
3.  **Iterate Cameras:** For each camera position on a sphere, compute the extrinsic transformation matrix using spherical coordinates $(r, r_x, r_y)$ where $r_x = \arccos(y/r)$ and $r_y = \arctan2(z, x)$, combined with a translation vector $t=[0, 0, 2]$ [S2].
4.  **Render Depth:** Render a depth image from the current camera pose [S1].
5.  **Carve Grid:** Carve the voxel grid using either the depth map or silhouette method, passing the depth image and camera parameters (intrinsic and extrinsic) [S1, S2].
6.  **Finalize Surface (Optional):** Create a surface voxel grid from the aggregated point cloud or mesh within bounds and add it to the carved grid [S2].

**Formulas:**
*   Extrinsic rotation: $r_x = \arccos(y/r)$, $r_y = \arctan2(z, x)$ [S2].
*   Translation: $t = [0, 0, 2]$ [S2].
*   Voxel size: $voxel\_size = cubic\_size / voxel\_resolution$ [S1].
*   Grid bounds: Centered at origin with bounds $[-cubic\_size/2, cubic\_size/2]$ [S1].

**Parameters:**
*   `voxel_resolution`: Controls the density of the voxel grid [S1].
*   `cubic_size`: Defines the dimensions of the bounding box for the voxel grid [S1].
*   `-dc` (Dilated Carving): Stops carving in each direction 1 voxel before the intersection point to mitigate artifacts [S3].

**Edge Cases and Pitfalls:**
*   **Historical Bugs:** binvox v0.48 contained a rendering bug for carve voxelization; v1.38 fixed a y-direction bug in carve voxelization [S3].
*   **Calibration:** The method requires accurately calibrated cameras; errors in intrinsic or extrinsic parameters lead to reconstruction artifacts [S4, unofficial].
*   **Boundaries:** Voxels on the boundary of the grid must be checked to ensure they are within the defined `origin` and `dimensions` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_carving(depth_maps, camera_poses, intrinsic_matrix, grid_config):
    # Initialize dense voxel grid
    grid = create_dense_grid(
        width=grid_config['width'],
        height=grid_config['height'],
        depth=grid_config['depth'],
        voxel_size=grid_config['voxel_size'],
        origin=grid_config['origin'],
        color=grid_config.get('color', [1.0, 0.7, 0.0])
    )
    # Preprocess model: center at origin, scale bounding box norm to 2.0
    model = preprocess_model(model)
    for pose in camera_poses:
        # Compute extrinsic matrix from spherical coordinates
        xyz = pose.position
        r = norm(xyz)
        r_x = acos(xyz.y / r)
        r_y = atan2(xyz.z, xyz.x)
        extrinsic = compute_extrinsic(r_x, r_y, translation=[0, 0, 2])
        # Render depth map for current pose
        depth_map = render_depth(model, intrinsic_matrix, extrinsic)
        # Carve grid using depth map or silhouette
        if grid_config['use_depth']:
            for y in range(depth_map.height):
                for x in range(depth_map.width):
                    d = depth_map[y, x]
                    if d > 0:
                        ray_origin = inv(extrinsic)[:3, 3]
                        ray_dir = normalize(inv(extrinsic)[:3, :3] @ inv(intrinsic_matrix) @ [x, y, 1])
                        surface_point = ray_origin + d * ray_dir
                        mark_voxels_between(grid, ray_origin, surface_point, grid_config['dilated_carving'])
        else:
            for y in range(depth_map.height):
                for x in range(depth_map.width):
                    if depth_map[y, x] > 0:  # Silhouette mask
                        ray_origin = inv(extrinsic)[:3, 3]
                        ray_dir = normalize(inv(extrinsic)[:3, :3] @ inv(intrinsic_matrix) @ [x, y, 1])
                        mark_voxels_ray(grid, ray_origin, ray_dir, grid_config['dilated_carving'])
    # Optionally add surface voxels
    if grid_config['add_surface']:
        surface_grid = create_surface_grid(pcd_agg, grid_config['voxel_size'], grid_config['bounds'])
        grid.merge(surface_grid)
    return grid

def create_dense_grid(width, height, depth, voxel_size, origin, color):
    # Standard initialization of a 3D occupancy grid [G]
    return Grid3D(width, height, depth, voxel_size, origin, color)

def preprocess_model(model):
    min_bound = model.get_min_bound()
    max_bound = model.get_max_bound()
    center = min_bound + (max_bound - min_bound) / 2.0
    scale = norm(max_bound - min_bound) / 2.0
    model.vertices = (model.vertices - center) / scale
    return model

def compute_extrinsic(r_x, r_y, translation):
    rot_x = [[1, 0, 0], [0, cos(r_x), -sin(r_x)], [0, sin(r_x), cos(r_x)]]
    rot_y = [[cos(r_y), 0, sin(r_y)], [0, 1, 0], [-sin(r_y), 0, cos(r_y)]]
    R = mat_mul(rot_y, rot_x)
    T = identity_matrix(4)
    T[:3, :3] = R
    T[:3, 3] = translation
    return T

def mark_voxels_between(grid, start, end, dilated):
    # Standard 3D line rasterization (DDA/Bresenham) [G]
    voxels = trace_line(start, end)
    for v in voxels:
        if dilated and is_last_voxel(v, end):
            continue
        grid.set_occupied(v, True)

def mark_voxels_ray(grid, origin, direction, dilated):
    # Standard ray traversal to grid bounds [G]
    voxels = trace_ray(origin, direction, grid.bounds)
    for v in voxels:
        if dilated and is_last_voxel(v, origin + direction * grid.max_dist):
            continue
        grid.set_occupied(v, True)
```

## Subtopics

### [Streaming GPU-based Voxel Carving](4436-streaming-gpu-based-voxel-carving.md)
A streaming GPU-based voxel carving method that creates an explicit volume, avoids 3D texture drawbacks by using only surface voxels in a GPU-optimal 2D data representation, and minimizes GPU-CPU data transfer through effective load balancing. It differs from its parent method, Voxel Carving, by explicitly optimizing for graphics hardware trends to maintain interactivity, and from previous approaches like [LiMS04], [WoeKoch04], and [ZacKar04] by avoiding the continuous access and dual-texture overhead inherent in 3D texture-based representations. Choose it when real-time interactivity and GPU memory efficiency are prioritized over dense volumetric filling.

### Interactive Multi-resolution Voxel Carving
A CPU-based voxel carving method that uses GVC-IB as its core but adds user-driven interactive refinement to create a model with varying resolution and level of detail across the voxel structure [S4, unofficial]. It differs from standard voxel carving by allowing human-guided input to assign perceptual significance to selected features, manipulating complexity via simple 2D image editing operations [S4, unofficial]. Choose it when perceptual quality and adaptive detail are more important than uniform resolution or processing speed.

### Cell Carving
An algorithm that uses correspondence between arbitrary image regions to enable reconstruction of concavities difficult or impossible to reconstruct with other methods [S4, unofficial]. Choose it when the target geometry contains deep concavities or complex internal structures that standard silhouette/depth carving fails to capture.

### Voxel Space Warping
A method that warps the voxel space to model large scale scenes without requiring an excessive number of voxels [S4, unofficial]. Choose it for outdoor or large-scale environments where uniform voxel resolution would exceed memory limits, accepting spatial distortion in exchange for scalability.

## Operating characteristics
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the dense voxel grid, where $W, H, D$ are the grid dimensions [S1].
*   **Performance:** Computationally expensive, typically requiring tens of seconds to tens of minutes for high-resolution reconstructions [S4, unofficial].
*   **Strengths:** Produces dense, filled volumes suitable for volumetric rendering; robust to noise in individual views due to intersection logic [S1].
*   **Limitations:** Requires accurate camera calibration; sensitive to missing data (occlusions) which can create holes; high memory usage for large scenes [S4, unofficial].
*   **When to use:** When a dense volumetric representation is required from multi-view depth data, and computational resources allow for iterative processing [S1].
*   **When not to use:** For real-time applications with limited hardware, or when only surface geometry is needed (use surface voxelization instead) [S1][S4, unofficial].

## Background
Voxel Carving is supported in tools like Open3D (`carve_depth_map`, `carve_silhouette`) and binvox (z-buffer based carving) [S1, S2, S3]. Early hardware-accelerated approaches were explored by Culbertson [Culbertson99] and Sainz et al. [SBS02] [S4, unofficial]. Theoretical foundations were laid by researchers like Seitz and Dyer [ProDy, S4, unofficial]. The method has been extended to handle uncalibrated views (Approximate Space Carving by Kutulakos [Kutulakos00, S4, unofficial]) and interactive refinement [S4, unofficial].

## Sources
[S1] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S2] Voxelization - Open3D 0.19.0 documentation, https://www.open3d.org/docs/release/tutorial/geometry/voxelization.html, fetched 2026-10-28
[S3] binvox 3D mesh voxelizer, keywords: voxelization, voxelisation, 3D model, https://www.patrickmin.com/binvox/, fetched 2026-10-29
[S4] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
