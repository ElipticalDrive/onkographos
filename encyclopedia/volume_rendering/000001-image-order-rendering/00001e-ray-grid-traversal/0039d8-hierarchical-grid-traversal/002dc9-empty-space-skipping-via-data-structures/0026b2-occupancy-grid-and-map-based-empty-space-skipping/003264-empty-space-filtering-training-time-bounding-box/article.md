# Empty Space Filtering (Training-Time Bounding Box)

Also called: Alpha-based Bounding Box Filtering, Training-time Empty Space Filtering

## Overview
Empty Space Filtering is a training-time acceleration strategy for dynamic neural volume rendering that restricts the optimization domain to the scene's active geometry. It solves the problem of inefficient computation in empty space by identifying inactive regions using alpha values from a coarse deformation feature grid and filtering out sampled points that fall within these regions [S1]. Unlike rendering-time acceleration methods that skip ray segments during inference, this method reduces the number of queries to the light-weight MLP during the training phase, thereby accelerating both training and subsequent rendering speeds [S1]. It sets itself apart from siblings like [Distance Map Acceleration](972-distance-map-acceleration.md) by operating on the training data sampling pipeline rather than the ray traversal logic, and from [Two-stage Ray Marching with Empty-Space Skipping](2985-two-stage-ray-marching-with-empty-space-skipping.md) by using a coarse grid prior to define a bounding box rather than skipping steps in a marching loop.

## How it works
**Inputs:**
*   Coarse deformation feature grid (voxel grid storing deformation features).
*   Alpha values associated with grid vertices.
*   Training time steps $t \in [0, T]$.
*   Ray sampling points in canonical space and deformation space.
*   Alpha threshold $\tau$ (scalar).

**Outputs:**
*   A coarse geometry prior (axis-aligned bounding box) enclosing all active object points across all time steps.
*   Filtered set of sampled points for training, excluding those identified as empty space.

**Data Structures:**
*   **Deformation Feature Grid:** A 3D grid where each vertex stores deformation features. The grid is queried to retrieve alpha values.
*   **Alpha Value:** A scalar derived from the deformation feature grid, representing the likelihood of a point belonging to the object.

**Steps:**

1.  **Query Alpha Values:** For every grid vertex in the coarse deformation feature grid, retrieve its alpha value for all training time steps $t$.
2.  **Classify Vertices:** Classify each grid vertex as an "object point" if its alpha value exceeds the threshold $\tau$ at any time step; otherwise, classify it as "empty space" [S1].
    *   Condition: A point is an object point if $\alpha > \tau$.
3.  **Compute Bounding Box:** Determine the smallest axis-aligned bounding box (AABB) that fully encloses all vertices classified as object points across all time steps. This defines the "coarse geometry prior" [S1].
4.  **Filter Sampled Points:** During training, when sampling points for rays:
    *   **Deformation Module:** Filter out sampled points if their alpha values are below the threshold $\tau$ for *all* training time steps. This assumes empty points are static [S1].
    *   **Canonical Module:** Filter out sampled points if their alpha values are below the threshold $\tau$ at the *canonical time* only [S1].
5.  **Fine Stage Initialization:** Use the coarse geometry prior (bounding box) to reduce the volume of the grid in the fine model stage, avoiding empty points [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def empty_space_filtering(coarse_deform_grid, alpha_threshold, time_steps):
    """
    Identifies active region and filters empty points during training.
    
    Args:
        coarse_deform_grid: 3D grid of deformation features.
        alpha_threshold: Scalar threshold for alpha value.
        time_steps: List of training time steps.
        
    Returns:
        bounding_box: AABB enclosing all object points.
        filter_function: Function to check if a point is empty.
    """
    # Step 1 & 2: Query alpha values and classify vertices
    object_points = []
    
    # Iterate over all grid vertices
    for vertex in coarse_deform_grid.vertices:
        is_object = False
        # Check alpha values across all time steps
        for t in time_steps:
            alpha = get_alpha_from_grid(vertex, t)
            if alpha > alpha_threshold:
                is_object = True
                break # Found at least one time step where it's an object
        
        if is_object:
            object_points.append(vertex.position)
            
    # Step 3: Compute bounding box
    if not object_points:
        return None, lambda p: False # No object points found
        
    min_coords = min(p.x for p in object_points), min(p.y for p in object_points), min(p.z for p in object_points)
    max_coords = max(p.x for p in object_points), max(p.y for p in object_points), max(p.z for p in object_points)
    bounding_box = AABB(min_coords, max_coords)
    
    # Step 4: Define filter function
    def is_empty(point, canonical_time, all_times):
        """
        Checks if a point is empty based on alpha values.
        """
        # Check deformation module condition: empty if alpha < threshold for ALL times
        # Note: The source implies filtering based on the coarse grid's alpha prediction for the point's location
        # at various times. In implementation, this requires querying the grid at the point's deformed positions.
        
        # For simplicity, assuming we can query alpha at a point's location for a given time
        # In practice, this involves warping the canonical point to time t and querying the grid.
        
        always_below_threshold = True
        for t in all_times:
            # Warp canonical point to time t
            deformed_point = warp_point(point, t)
            alpha = get_alpha_at_position(deformed_point)
            if alpha > alpha_threshold:
                always_below_threshold = False
                break
                
        if always_below_threshold:
            return True # Empty in deformation module
            
        # For canonical module: filter if alpha < threshold at canonical time
        alpha_canonical = get_alpha_at_position(point) # Point is in canonical space
        if alpha_canonical <= alpha_threshold:
            return True # Empty in canonical module
            
        return False

    return bounding_box, is_empty
```

**Parameters:**
*   **Alpha Threshold ($\tau$):** A predefined scalar value used to distinguish object points from empty space. Typical values are not explicitly given in the sources, but it must be tuned to avoid filtering low-density regions [S1].

**Edge Cases and Pitfalls:**
*   **Low-Density Regions:** Regions with low density (e.g., a white plate on a white background) may have low image reconstruction error even if the learned density is close to zero. These regions might be incorrectly filtered out as empty space [S1]. This can be alleviated by using a background with a different color or disabling the filtering strategy for such scenes [S1].

## Subtopics

### [Depth-guided Voxel Pre-pruning](6897-depth-guided-voxel-pre-pruning.md)
A preprocessing strategy that removes voxels in free space using dense depth priors from 3D vision foundation models before optimization. It differs by using external depth priors rather than internal alpha values from a coarse grid, and it prunes the voxel grid directly rather than filtering sampled points during training.

## Operating characteristics
*   **Time Complexity:** Reduces the number of queries to the light-weight MLP during training, significantly decreasing training time [S1].
*   **Memory Complexity:** Reduces the volume of the grid in the fine stage by using the coarse geometry prior, potentially saving memory [S1].
*   **Performance:** The full model with empty space filtering (NDGV full) achieves better training speed, render speed, and PSNR compared to the model without filtering (NDGV w/o filter) [S1]. The training speed of the unfiltered model is on the same level as the filtered model, indicating that the filtering primarily aids in reducing the search space for optimization rather than just raw computation per step, but the overall efficiency gain is significant [S1].
*   **Strengths:** Accelerates training and rendering; reduces the volume of the grid in the fine stage; implicitly regularizes scene geometry.
*   **Limitations:** May incorrectly filter out low-density regions with low reconstruction error (e.g., transparent or background-colored objects) [S1].

## Background
The method was proposed by X. Guo et al. in the paper "Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis" [S1]. It is part of the Neural Deformable Voxel Grid (NDGV) framework, which uses a coarse-to-fine optimization strategy. The empty space filtering strategy is introduced to speed up the training process by avoiding computation in empty space and reducing the number of MLP queries [S1].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
