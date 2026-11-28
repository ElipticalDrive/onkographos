# Empty Space Filtering (Training-Time Bounding Box)

Also called: Alpha-based Bounding Box Filtering, Ray skipping, Training-Time Bounding Box, Training-time Empty Space Filtering

## Overview
Empty Space Filtering is a training-time optimization strategy that accelerates neural volume rendering by restricting computation to the scene's actual content [S1, S2]. It solves the problem of wasted computation in empty space by identifying a tight bounding box and filtering out empty samples during ray traversal [S1, S2]. Unlike static acceleration structures that rely on pre-computed occupancy grids or distance maps, this method derives its acceleration structure dynamically from the learned alpha values of a coarse model during the initial training stages [S1, S2]. It sets itself apart from siblings like [Distance Map Acceleration](972-distance-map-acceleration.md) by using learned opacity rather than geometric distance, and from [Two-stage Ray Marching with Empty-Space Skipping](2985-two-stage-ray-marching-with-empty-space-skipping.md) by integrating the bounding box shrinkage directly into the training loop of dynamic scenes [S1].

## How it works
The method operates in two phases: bounding box determination and sample filtering [S1, S2]. It builds on the shared principles of [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) by constructing a secondary structure (the bounding box) and defining activity via alpha thresholds [S1, S2].

### Inputs and Outputs
*   **Inputs:**
    *   Coarse deformation feature grid (or coarse geometry prior) from the initial training stage [S1].
    *   Alpha threshold $\tau$ [S1].
    *   Ray origin $o$ and direction $d$ [S2].
*   **Outputs:**
    *   A tight axis-aligned bounding box (AABB) $[min, max]$ enclosing all object points [S1].
    *   A filtered set of sample points along the ray, excluding those identified as empty [S1].

### Data Structures
1.  **Coarse Deformation Feature Grid:** A 3D grid storing deformation features where each vertex $v$ has an associated alpha value $\alpha(v)$ derived from the density field [S1].
2.  **Bounding Box:** An axis-aligned bounding box defined by minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$ [S1, S2].
3.  **Empty Mask (Optional):** A binary volume or mask recording whether each voxel is empty, used to skip empty voxels along valid rays [S2].

### Steps
1.  **Query Alpha Values:** Query all grid vertices of the coarse deformation feature grid across all training time steps $t \in [0, T]$ to retrieve their alpha values $\alpha(v, t)$ [S1].
2.  **Classify Vertices:** Classify each grid vertex $v$ as an object point if its alpha value exceeds a predefined threshold $\tau$ for at least one time step [S1]. Specifically, a point is assumed to belong to an object if $\alpha(v, t) > \tau$ [S1]. If $\alpha(v, t) \le \tau$ for all $t$, the point is classified as empty space [S1].
3.  **Compute Bounding Box:** Compute the smallest axis-aligned bounding box that fully encloses all vertices classified as object points across all time steps [S1]. This box is initialized to the full extent of the scene and shrunk after the initial training stage completes [S2].
4.  **Ray Intersection Test:** For each ray, test if it intersects the computed bounding box [S2]. If the ray does not intersect the box, skip the ray completely [S2].
5.  **Sample Filtering:** During ray marching, filter out sampled points that are recognized as empty [S1].
    *   For points in the deformation module, filter out points whose alpha values are always below the threshold during all training times [S1].
    *   For points in the canonical module, filter out empty points only at the canonical time [S1].
    *   Alternatively, use a pre-calculated occupancy grid to prune non-occupied points along rays [S3].

### Formula
A point is assumed to belong to an object if its alpha value is larger than a predefined threshold:
$$ \alpha(v, t) > \tau $$
where $\alpha(v, t)$ is the alpha value of vertex $v$ at time step $t$, and $\tau$ is the alpha threshold [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_bounding_box(coarse_grid, alpha_threshold):
    """
    Computes the tight bounding box enclosing all object points.
    coarse_grid: 3D grid of alpha values for all time steps.
    alpha_threshold: scalar threshold for object vs empty space.
    Returns: (min_coords, max_coords)
    """
    min_coords = [float('inf'), float('inf'), float('inf')]
    max_coords = [float('-inf'), float('-inf'), float('-inf')]
    
    # Query all grid vertices across all time steps
    for t in range(coarse_grid.num_time_steps):
        for v in coarse_grid.vertices:
            if coarse_grid.get_alpha(v, t) > alpha_threshold:
                # Vertex is an object point
                for i in range(3):
                    if v.coords[i] < min_coords[i]:
                        min_coords[i] = v.coords[i]
                    if v.coords[i] > max_coords[i]:
                        max_coords[i] = v.coords[i]
                        
    return min_coords, max_coords

def ray_aabb_intersect(origin, direction, min_c, max_c):
    """
    Standard slab method for ray-AABB intersection.
    Returns (t_min, t_max) or (None, None) if no intersection.
    """
    t_min = (min_c[0] - origin[0]) / direction[0] if direction[0] != 0 else float('-inf')
    t_max = (max_c[0] - origin[0]) / direction[0] if direction[0] != 0 else float('inf')
    if t_min > t_max: t_min, t_max = t_max, t_min
    for i in range(1, 3):
        if direction[i] != 0:
            t1 = (min_c[i] - origin[i]) / direction[i]
            t2 = (max_c[i] - origin[i]) / direction[i]
            if t1 > t2: t1, t2 = t2, t1
            if t1 > t_min: t_min = t1
            if t2 < t_max: t_max = t2
        else:
            if origin[i] < min_c[i] or origin[i] > max_c[i]: return None, None
    if t_max < 0: return None, None
    return t_min, t_max

def ray_marching_with_filtering(ray_origin, ray_direction, bounding_box, coarse_grid, alpha_threshold, step_size):
    """
    Performs ray marching, skipping empty space.
    ray_origin: (x, y, z)
    ray_direction: (x, y, z)
    bounding_box: (min_coords, max_coords)
    coarse_grid: 3D grid of alpha values
    alpha_threshold: scalar threshold
    step_size: distance between samples
    Returns: rendered color
    """
    min_c, max_c = bounding_box
    t_start, t_end = ray_aabb_intersect(ray_origin, ray_direction, min_c, max_c)
    if t_start is None: return 0.0
    
    color = 0.0
    alpha_accum = 0.0
    t = t_start
    
    while t < t_end and alpha_accum < 1.0:
        point = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
        
        # Check if point is empty
        alpha = coarse_grid.query_alpha(point)
        
        if alpha > alpha_threshold:
            # Point is not empty, compute contribution
            # Standard volume rendering accumulation [G]
            density, color_val = 1.0, [0.0, 0.0, 0.0]
            weight = density * step_size * (1.0 - alpha_accum)
            color += weight * color_val
            alpha_accum += weight
        else:
            # Point is empty, skip sampling
            pass
            
        t += step_size
        
    return color
```

### Parameters
*   **Alpha Threshold ($\tau$):** A predefined scalar value used to distinguish object points from empty space [S1]. Typical values are not explicitly given in the sources.
*   **Bounding Box Initialization:** Initialized to the full extent of the scene, then shrunk based on the occupancy grid at step 2,000 of training [S3].

### Edge Cases and Pitfalls
*   **Low-Density Regions:** Low-density regions (e.g., a white plate on a white background) may be incorrectly filtered out because the image reconstruction error is low even if the learned density is close to zero [S1]. This can be alleviated by using a background with a different color or disabling the filtering strategy [S1].
*   **Bounding Box Shrinkage Timing:** If the bounding box is not shrunk after the initial training stage, it will contain empty voxels, reducing the effectiveness of ray skipping [S2].
*   **Dynamic Scenes:** For dynamic scenes, the bounding box must account for object motion across all time steps to ensure no object points are excluded [S1].

## Subtopics

### [Depth-guided Voxel Pre-pruning](6897-depth-guided-voxel-pre-pruning.md)
Depth-guided voxel pre-pruning is a preprocessing strategy that removes voxels located in free space using a dense depth prior before the optimization phase begins. It reduces the number of samples in free space and the parameters to be optimized, thereby accelerating training while implicitly regularizing scene geometry without requiring explicit depth supervision. Unlike methods that rely on explicit depth supervision using sparse Structure-from-Motion (SfM) or imperfect monocular depth estimators, this method leverages dense, multi-view consistent depth priors from 3D vision foundation models to prune the voxel grid directly.

### Occupancy Pruning and Bounding Box Shrinkage
Occupancy Pruning and Bounding Box Shrinkage is a training and inference acceleration technique that pre-calculates an occupancy grid to prune non-occupied points along rays and shrinks the scene bounding box for more accurate modeling [S3]. The occupancy grid is pre-calculated at training steps 2,000 and 4,000 [S3]. The bounding box is shrunk based on the grid at step 2,000 [S3]. The grid is kept for scene composition, recorded per object, and used with warped rays to query occupancy [S3]. L1 regularization is applied to the factored matrices for density to encourage sparsity in tensor rank decomposition parameters [S3].

## Operating characteristics
*   **Time Complexity:** Reduces training and rendering time by avoiding computation in empty space [S1]. Without filtering, training speed, render speed, and PSNR are all clearly worse than the full model [S1].
*   **Memory Complexity:** Requires storing the coarse deformation feature grid and potentially an occupancy grid [S1, S3].
*   **Strengths:** Significantly reduces time consumption of training and rendering by reducing the number of queries to the light-weight MLP [S1]. It allows for full use of grid resolution by locating the smallest box region that fully covers the whole scene [S1].
*   **Limitations:** May incorrectly filter out low-density regions if the background color is similar to the object [S1]. Requires an initial training stage to compute the bounding box and occupancy grid [S2, S3].
*   **When to Use:** When training dynamic neural volume rendering models where empty space constitutes a significant portion of the volume [S1, S2].
*   **When Not to Use:** When the scene contains low-density objects with colors similar to the background, as they may be incorrectly filtered out [S1].

## Background
Empty Space Filtering was proposed in the context of Neural Deformable Voxel Grid (NDGV) for fast optimization of dynamic view synthesis [S1]. It was introduced to speed up training by reducing the number of queries to the light-weight MLP and locating the smallest box region that fully covers the whole scene [S1]. The method builds on the idea of using alpha values from a coarse deformation feature grid to identify empty space [S1]. It has been adopted in other works such as ViSNeRF for efficient multidimensional neural radiance field representation [S2] and Compressible-Composable NeRF via rank-residual decomposition [S3].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes, https://doi.org/10.1109/pacificvis64226.2025.00029, fetched 2026-10-26
[S3] Compressible-Composable NeRF Via Rank-Residual Decomposition, https://doi.org/10.52202/068431-1076, fetched 2026-10-26
