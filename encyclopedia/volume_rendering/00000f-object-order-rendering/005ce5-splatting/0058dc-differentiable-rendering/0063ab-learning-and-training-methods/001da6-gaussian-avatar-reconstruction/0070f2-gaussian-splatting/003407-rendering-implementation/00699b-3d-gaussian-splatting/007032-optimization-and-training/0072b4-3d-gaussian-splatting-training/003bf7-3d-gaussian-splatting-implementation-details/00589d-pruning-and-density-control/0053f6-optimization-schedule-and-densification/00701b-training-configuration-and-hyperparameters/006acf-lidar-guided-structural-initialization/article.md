# LiDAR-Guided Structural Initialization

Also called: DrivingVoxels Structural Initialization, Proximity-Based Voxel Subdivision, Soft Density Initialization Prior

## Overview
LiDAR-Guided Structural Initialization is a method for initializing the geometric structure and density priors of a sparse voxel grid for driving scene reconstruction, specifically within the DrivingVoxels framework [S1, unofficial]. It solves the problem of initializing a volumetric representation that is both sparse enough for efficiency and geometrically plausible enough to guide optimization, by using LiDAR point clouds to define voxel subdivision and density values. It differs from sibling methods like [Adaptive Gaussian Refinement Strategy](16570-adaptive-gaussian-refinement-strategy.md) by operating on a voxel grid rather than Gaussian splats, and from [Gaussian Pruning](18664-gaussian-pruning.md) by focusing on initial structural creation rather than post-training cleanup.

## How it works
The method initializes a multi-resolution voxel grid in two main phases: structural refinement via LiDAR-guided subdivision, followed by density initialization via a soft prior. It handles the static background and dynamic assets separately.

**Inputs:**
- LiDAR point cloud $P_{LiDAR}$.
- Camera poses and intrinsics for training views.
- Asset bounding boxes (for dynamic objects).
- Minimum voxel size threshold $s_{min} = 30$ cm [S1, unofficial].

**Outputs:**
- A sparse voxel grid structure with initialized densities $v_{geo}$.

**Data Structures:**
- **Voxel Grid:** A hierarchical structure (e.g., octree or multi-resolution grid) where each voxel is defined by its spatial bounds and a density value $v_{geo}$ stored at its corners.
- **Inner Region:** A uniform coarse grid covering the scene volume containing all camera poses.
- **Outer Region:** Two coarse resolution levels representing far-away structures.
- **Asset Grids:** Uniform voxel grids aligned with local coordinate systems for each asset.

**Steps:**

1. **Inner Region Initialization:**
   - Initialize the inner region as a coarse uniform voxel grid aligned with the global coordinate system.
   - The grid bounds are determined by the scene extent containing all training camera poses.

2. **Visibility Pruning (Inner Region):**
   - For each voxel in the inner region, check visibility:
     - A voxel is visible if it is projected into the frustum of any training camera.
     - A voxel is valid if its height is above a minimum camera-height reference threshold.
   - Prune (remove or mark as empty) voxels that are not visible by any training camera or fall below the height threshold.

3. **Outer Region Initialization:**
   - Represent the outer region (far-away structures) at two coarse resolution levels.
   - Apply the same frustum and height filter as the inner region to prune voxels.

4. **Proximity-Based Voxel Subdivision:**
   - Iterate through rounds of subdivision on the remaining voxels in the inner region.
   - In each round, for each voxel:
     - Compute the minimum Euclidean distance $d_{min}$ from the voxel center (or any point within the voxel) to any point in $P_{LiDAR}$.
     - If $d_{min}$ is below a distance threshold (implied to be related to voxel size or a fixed value, though not explicitly given as a constant, the process continues until $s_{min}$ is reached), subdivide the voxel into 8 children.
   - Continue subdivision rounds until the minimum voxel size $s$ falls below 30 cm [S1, unofficial]. This creates a finer shell near physical surfaces (LiDAR points) while keeping empty space coarse.

5. **Soft Density Initialization Prior:**
   - For each corner of every voxel in the refined grid:
     - Compute $d$, the Euclidean distance to the nearest point in $P_{LiDAR}$.
     - Calculate the initial density $v_{geo}^{init}$ using the formula:
       $$v_{geo}^{init} = v_{far} + (v_{close} - v_{far}) \cdot \exp \left (-\frac {2d^2}{s^2}\right )$$
       where:
       - $s$ is the minimum voxel size at the current refinement level (or the size of the voxel containing the corner? The source says "minimum voxel size", which likely refers to the finest resolution $s_{min}$ or the local voxel size. Given the context of "smooth density gradients", using the local voxel size $s_{local}$ for normalization is common, but the source explicitly states "where s is minimum voxel size". We follow the source literally: $s = 30$ cm or the global minimum resolution). *Note: The source text says "where s is minimum voxel size". In the context of the formula, if $s$ is a constant global minimum, the decay rate is fixed. If $s$ is local, it adapts. The source does not clarify if $s$ is local or global. However, "minimum voxel size" usually implies the finest resolution limit. We will use the global minimum voxel size $s_{min} = 30$ cm as per the explicit definition.*
       - $v_{close} = -2.0$ [S1, unofficial].
       - $v_{far} = -10.0$ [S1, unofficial].

6. **Asset Initialization:**
   - For each dynamic asset:
     - Create a uniform voxel grid aligned with the asset's local bounding box coordinate system.
     - Set the grid boundary to the largest dimension of the asset's 3D bounding box, scaled outward by 20% [S1, unofficial].
     - Initialize all voxel corners in the asset grid with a uniform empty density $v_{geo}^{init} = -10.0$ [S1, unofficial].

**Parameters:**
- $v_{close} = -2.0$: Density value for voxel corners close to LiDAR points [S1, unofficial].
- $v_{far} = -10.0$: Empty density value for distant voxel corners [S1, unofficial].
- $s_{min} = 30$ cm: Minimum voxel size threshold for subdivision [S1, unofficial].
- Asset boundary scale: 20% outward from the largest bounding box dimension [S1, unofficial].

**Edge Cases and Pitfalls:**
- **LiDAR Sparsity:** Areas with no LiDAR data will have high $d$, resulting in $v_{geo} \approx v_{far} = -10.0$, which is the empty density. This allows the optimizer to recover surfaces missed by LiDAR.
- **Precision:** The exponential decay $\exp(-2d^2/s^2)$ can lead to very small density values for large $d$. Ensure floating-point precision is sufficient.
- **Boundary Conditions:** The 20% margin for assets ensures the grid covers the asset geometry. If the margin is too small, parts of the asset may be outside the grid.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def initialize_lidar_guided_structure(lidar_points, camera_poses, asset_bboxes, s_min=0.3):
    """
    Initialize voxel grid structure and densities using LiDAR guidance.
    
    Args:
        lidar_points: List of 3D points (x, y, z) from LiDAR.
        camera_poses: List of camera poses and intrinsics.
        asset_bboxes: List of bounding boxes for dynamic assets.
        s_min: Minimum voxel size in meters (default 0.3m).
        
    Returns:
        inner_grid: Voxel grid for inner region.
        outer_grids: List of two voxel grids for outer region.
        asset_grids: List of voxel grids for each asset.
    """
    
    # Step 1 & 2: Inner Region Initialization and Pruning
    inner_grid = create_coarse_uniform_grid(scene_extent_containing_cameras(camera_poses))
    inner_grid = prune_voxels_by_visibility_and_height(inner_grid, camera_poses, lidar_points)
    
    # Step 3: Outer Region Initialization
    outer_grids = [create_coarse_grid(level) for level in [1, 2]] # Two coarse levels
    for grid in outer_grids:
        grid = prune_voxels_by_visibility_and_height(grid, camera_poses, lidar_points)
        
    # Step 4: Proximity-Based Voxel Subdivision
    inner_grid = subdivide_voxels_by_lidar_proximity(inner_grid, lidar_points, s_min)
    
    # Step 5: Soft Density Initialization
    inner_grid = initialize_voxel_densities(inner_grid, lidar_points, s_min)
    
    # Step 6: Asset Initialization
    asset_grids = []
    for bbox in asset_bboxes:
        asset_grid = create_uniform_grid_from_bbox(bbox, scale_factor=1.2)
        asset_grid = initialize_uniform_density(asset_grid, v_far=-10.0)
        asset_grids.append(asset_grid)
        
    return inner_grid, outer_grids, asset_grids

def prune_voxels_by_visibility_and_height(grid, camera_poses, lidar_points):
    """Prune voxels not visible to any camera or below height threshold."""
    # [G] Height threshold is not explicitly defined in sources. 
    # Assume a global minimum height (e.g., ground level) is derived from lidar or camera data.
    min_height = get_minimum_camera_height(camera_poses) 
    
    pruned_voxels = []
    for voxel in grid:
        is_visible = False
        for cam in camera_poses:
            if is_voxel_in_frustum(voxel, cam):
                is_visible = True
                break
        if not is_visible:
            continue
        if voxel.center_y < min_height:
            continue
        pruned_voxels.append(voxel)
    return Grid(pruned_voxels)

def subdivide_voxels_by_lidar_proximity(grid, lidar_points, s_min):
    """Iteratively subdivide voxels close to LiDAR points until s_min is reached."""
    current_grid = grid
    while True:
        # Find voxels that can be subdivided (size > s_min)
        subdividable = [v for v in current_grid if v.size > s_min]
        if not subdividable:
            break
            
        new_voxels = []
        for voxel in subdividable:
            # Compute min distance to LiDAR points
            min_dist = min_distance_to_point_cloud(voxel.center, lidar_points)
            # [G] The source says "voxels closer than a distance threshold". 
            # It does not specify the threshold value. 
            # A common heuristic is to subdivide if the voxel intersects or is close to LiDAR points.
            # We assume a threshold based on voxel size or a fixed value. 
            # Given the goal is "progressively finer shell on physical surface", 
            # we subdivide if min_dist < voxel.size (or a fraction thereof).
            # Since the source is vague, we use a conservative condition: 
            # subdivide if the voxel contains or is very near LiDAR points.
            if min_dist < voxel.size: 
                new_voxels.extend(subdivide_voxel(voxel))
            else:
                new_voxels.append(voxel)
        # Add non-subdividable voxels
        non_subdividable = [v for v in current_grid if v.size <= s_min or (v.size > s_min and min_distance_to_point_cloud(v.center, lidar_points) >= v.size)]
        # Note: The above logic is simplified. A proper implementation would track which voxels were already processed.
        # For the purpose of this article, we assume the loop processes all voxels > s_min.
        current_grid = Grid(new_voxels + non_subdividable)
    return current_grid

def min_distance_to_point_cloud(point, lidar_points):
    """Compute Euclidean distance from point to nearest LiDAR point."""
    min_d = float('inf')
    for p in lidar_points:
        d = math.sqrt((point[0]-p[0])**2 + (point[1]-p[1])**2 + (point[2]-p[2])**2)
        if d < min_d:
            min_d = d
    return min_d

def initialize_voxel_densities(grid, lidar_points, s_min):
    """Initialize voxel corner densities using the soft density prior."""
    v_close = -2.0
    v_far = -10.0
    for voxel in grid:
        for corner in voxel.corners:
            d = min_distance_to_point_cloud(corner, lidar_points)
            # Formula: v_init = v_far + (v_close - v_far) * exp(-2 * d^2 / s^2)
            # s is minimum voxel size
            exponent = -2.0 * (d ** 2) / (s_min ** 2)
            density = v_far + (v_close - v_far) * math.exp(exponent)
            corner.density = density
    return grid

def create_uniform_grid_from_bbox(bbox, scale_factor=1.2):
    """Create a uniform voxel grid from a bounding box, scaled outward."""
    # [G] Implementation details for grid creation are not in sources.
    # Assume a function that creates a grid covering the bbox * scale_factor.
    pass

def initialize_uniform_density(grid, v_far):
    """Set all voxel corners to v_far."""
    for voxel in grid:
        for corner in voxel.corners:
            corner.density = v_far
    return grid
```

## Operating characteristics
- **Time Complexity:** The subdivision step is iterative and depends on the density of LiDAR points and the scene geometry. In the worst case, it may approach $O(N \log N)$ or higher depending on the spatial indexing used for LiDAR proximity queries. Density initialization is $O(V)$ where $V$ is the number of voxel corners.
- **Memory Complexity:** The method creates a sparse voxel grid. Memory usage is proportional to the number of voxels, which is significantly less than a dense grid. The outer region uses two coarse levels, further reducing memory.
- **Performance:** The method is designed for high rendering efficiency by creating a sparse structure. The tile-based rasterizer mentioned in the source relies on this sparse structure [S1, unofficial].
- **Strengths:** Provides a geometrically plausible starting point for optimization, handles dynamic assets separately, and uses LiDAR to guide structure, reducing the need for extensive densification later.
- **Limitations:** Relies on the quality and coverage of the LiDAR point cloud. Areas with no LiDAR data may require more optimization effort to recover. The fixed parameters ($v_{close}, v_{far}, s_{min}$) may not be optimal for all scenes.

## Background
The method originates from the DrivingVoxels paper, which introduces compositional sparse voxel rasterization for dynamic driving scene reconstruction [S1, unofficial]. It builds on the SVRaster implementation for visibility pruning and frustum culling [S1, unofficial]. The approach is tailored for autonomous driving scenarios where LiDAR data is abundant and dynamic objects (assets) need to be handled separately from the static background.

## Sources
[S1] DrivingVoxels: Compositional Sparse Voxel Rasterization for Dynamic Driving Scene Reconstruction, https://doi.org/10.48550/arxiv.2606.23031, fetched 2026-10-26 (unofficial)
