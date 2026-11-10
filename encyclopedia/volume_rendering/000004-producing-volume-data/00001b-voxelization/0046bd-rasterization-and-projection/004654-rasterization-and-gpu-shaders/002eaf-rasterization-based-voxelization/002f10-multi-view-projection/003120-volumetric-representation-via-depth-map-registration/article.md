# Volumetric Representation via Depth Map Registration

Also called: Sparse Multi-View Volumetric Representation, Sparse Non-Lambertian Multi-view Volume Construction, Volume Initialization

## Overview
This method constructs a 3D volumetric representation from a sparse set of calibrated multi-view images by registering estimated depth maps into a voxel grid [S1, S2]. It specifically addresses non-Lambertian scenes by using individual minimum cost per view for depth estimation rather than a global threshold, and represents geometry as a sparse voxel cloud where voxel values indicate the number of views mapping to that location [S1, S2]. It distinguishes itself from static voxelization methods by incorporating an iterative refinement loop that removes isolated voxels and smooths sharp corners to increase coding gain for virtual view synthesis [S1, S2].

## How it works
**Inputs:**
*   $N$ calibrated reference views with known camera parameters.
*   For each view, an initial depth map $D_v$ estimated by searching along the view's geometry, compensating for non-Lambertian reflection using individual minimum cost [S1, S2].

**Outputs:**
*   A sparse 3D voxel grid where each active voxel $(i, j, k)$ stores a weight $\delta_{ijk}$ (count of views) and a color $C_{ijk}$.

**Data Structures:**
*   **Voxel Grid:** A 3D array or sparse hash map.
    *   Key: Voxel coordinates $(i, j, k)$.
    *   Value: A struct containing:
        *   `weight`: Integer $\delta_{ijk} \in \{0, 1, ..., N\}$.
        *   `color`: RGB or YUV tuple.
        *   `valid`: Boolean flag (optional, implied by weight > 0).
*   **Depth Map:** 2D array for each view, storing depth values $d$.

**Algorithm Steps:**

1.  **Volume Initialization (Registration):**
    For each reference view $v \in \{1, ..., N\}$:
    For each pixel $p$ in view $v$ with depth $d_p$:
    1.  Project the 3D point corresponding to $d_p$ into the global voxel grid coordinates $(i, j, k)$ using the view's projection matrix and the scene's axis-aligned bounding box (AABB) [S1, S2, Parent].
    2.  Increment the weight of the voxel at $(i, j, k)$:
        $$ \delta_{ijk} \leftarrow \delta_{ijk} + 1 $$
    3.  If $\delta_{ijk}$ was 0, mark the voxel as active [S1, S2].

2.  **Isolated Voxel Removal:**
    Iterate through all active voxels.
    For each voxel $x$, check its 6-connected (or 26-connected) neighbors.
    If a voxel has no active neighbors (weight > 0), remove it from the volume [S1, S2].

3.  **Depth Map Reconstruction (Back-Projection):**
    Project the current volume back to each reference view geometry to build new depth maps [S1, S2].
    For each pixel $p$ in view $v$:
    1.  Cast a ray from the camera center through $p$.
    2.  Identify all voxels intersected by the ray.
    3.  Collect the depth values $d_k$ of these voxels.
    4.  Identify the "front cluster" of depths (the closest group of voxels to the camera) [S1, S2].
    5.  Calculate the proper depth value $d$ for the pixel:
        $$ d = \begin{cases} d_n & \text{if only one depth has } \delta_n = \max(\delta) \\ \frac{\sum_{k=1}^{K} \delta_k d_k}{\sum_{k=1}^{K} \delta_k} & \text{otherwise} \end{cases} $$
        Where $K$ is the number of depth values in the front cluster, $d_k$ is the depth value, and $\delta_k$ is the voxel weight [S1, S2].

4.  **Volume Refinement (Iterative):**
    Initialize temporary colors for all active voxels as the median color of all visible views at that voxel [S2].
    Repeat until convergence or max iterations:
    1.  **Reconstruction:** Render all $N$ reference views from the current volume using the temporary colors.
    2.  **Error Calculation:** For each view $v$ (processed in order, e.g., left to right):
        1.  Calculate the Mean Square Error (MSE) between the reconstructed view and the original reference image $I_v$. Use a window-based computation (e.g., 3x3 pixels) to handle calibration/imperfect resolution issues [S2].
        2.  If $MSE > \text{threshold}$:
            a.  Remove the old depth and associated voxel from the volume.
            b.  Investigate a new depth derived from neighboring pixels in the reference view (smooth surface assumption) [S2].
            c.  If the new depth satisfies color constraints for all visible views, generate the new associated voxel with the new depth [S2].
    3.  **Color Update:** If any voxels were removed or generated, update the temporary colors (median of visible views) and repeat the reconstruction step, as voxel position changes affect visibility (invisible to visible or vice versa) [S2].

5.  **Color Selection (Post-Processing):**
    For each active voxel, determine its final color $C$ based on visible views [S2].
    For two views 1 and 2 with colors $C_1, C_2$ and angles $a_1, a_2$ between the voxel and the points on lines $L_1, L_2$:
    $$ C = \begin{cases} \frac{a_1 C_2 + a_2 C_1}{a_1 + a_2} & \text{if condition is true} \\ C_1 & \text{otherwise} \end{cases} $$
    If the difference between $C_1$ and $C_2$ exceeds a threshold, the color is placed in a "blur group" to prevent object blur. In this case, replace the color with that of a neighboring voxel in a "clear group" that has the closest value [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_representation_via_depth_map_registration(reference_views, initial_depth_maps, threshold, max_iterations):
    """
    reference_views: List of camera objects with projection matrices.
    initial_depth_maps: List of 2D arrays, one per view.
    threshold: MSE threshold for refinement.
    max_iterations: Max refinement loops.
    """
    # 1. Initialization
    voxel_grid = SparseVoxelGrid()
    
    for view_idx, view in enumerate(reference_views):
        depth_map = initial_depth_maps[view_idx]
        for y, x in pixels_in_view(view):
            d = depth_map[y, x]
            if d is valid:
                # Project 3D point to voxel coordinates
                voxel_coords = project_to_voxel_grid(view, d, x, y)
                voxel_grid.increment_weight(voxel_coords)
                
    # 2. Remove isolated voxels
    active_voxels = voxel_grid.get_active_voxels()
    for voxel in active_voxels:
        if not voxel_grid.has_active_neighbors(voxel):
            voxel_grid.remove(voxel)
            
    # 3. Back-projection to get refined depth maps
    refined_depth_maps = []
    for view in reference_views:
        new_depth_map = create_empty_depth_map(view)
        for y, x in pixels_in_view(view):
            ray = view.cast_ray(x, y)
            front_cluster = voxel_grid.get_front_cluster(ray)
            if front_cluster:
                # Calculate proper depth d
                weights = [v.weight for v in front_cluster]
                depths = [v.depth for v in front_cluster]
                max_weight = max(weights)
                max_count = weights.count(max_weight)
                
                if max_count == 1:
                    # Find the depth corresponding to the unique max weight
                    idx = weights.index(max_weight)
                    d = depths[idx]
                else:
                    # Weighted average
                    numerator = sum(w * d_val for w, d_val in zip(weights, depths))
                    denominator = sum(weights)
                    d = numerator / denominator
                new_depth_map[y, x] = d
        refined_depth_maps.append(new_depth_map)
        
    # 4. Iterative Refinement
    # Initialize temporary colors
    for voxel in voxel_grid.get_active_voxels():
        voxel.color = median_color_from_views(voxel, reference_views)
        
    for iteration in range(max_iterations):
        changed = False
        for view_idx, view in enumerate(reference_views):
            # Render view
            rendered_view = render_volume(voxel_grid, view)
            original_view = get_original_image(view)
            
            # Calculate MSE with window
            mse = calculate_windowed_mse(rendered_view, original_view, window_size=3)
            
            if mse > threshold:
                # Remove old voxel and try new depth
                # Note: This logic implies we are modifying the volume based on view errors
                # The source describes removing the "old depth" and associated voxel.
                # We need to identify which voxel caused the error.
                # Simplified: Assume we re-evaluate the voxel at the error location
                # using neighboring pixels' depth (smooth surface assumption).
                
                # Find error location (simplified)
                error_loc = find_max_error_location(rendered_view, original_view)
                if error_loc:
                    # Remove voxel at error_loc
                    voxel_grid.remove(error_loc)
                    changed = True
                    
                    # Try new depth from neighbors
                    new_depth = estimate_depth_from_neighbors(view, error_loc)
                    if new_depth is valid:
                        # Check color constraints for all visible views
                        if satisfies_color_constraints(new_depth, error_loc, reference_views):
                            new_voxel = create_voxel(error_loc, new_depth)
                            voxel_grid.add(new_voxel)
                            changed = True
                            
        if not changed:
            break
            
        # Update colors if changed
        if changed:
            for voxel in voxel_grid.get_active_voxels():
                voxel.color = median_color_from_views(voxel, reference_views)
                
    # 5. Color Selection
    for voxel in voxel_grid.get_active_voxels():
        voxel.color = select_final_color(voxel, reference_views)
        
    return voxel_grid
```

## Subtopics

### [Iterative Volume Refinement](6473-iterative-volume-refinement.md)
Iterative Volume Refinement is a post-initialization algorithm that cleans and smooths a sparse voxel cloud generated from multi-view depth maps by removing scattering voxels and smoothing sharp corners to increase coding gain. It distinguishes itself from static voxelization by dynamically regenerating voxels based on a smooth surface assumption and iteratively re-evaluating visibility, ensuring the final volume accurately represents the real scene for virtual view synthesis.

## Operating characteristics
*   **Time Complexity:** The initialization step is $O(N \cdot W \cdot H)$ where $N$ is the number of views and $W \cdot H$ is the image resolution. The refinement step is iterative and depends on the number of voxels and views, potentially $O(I \cdot N \cdot V)$ where $I$ is iterations and $V$ is active voxels.
*   **Memory Complexity:** Proportional to the number of active voxels. The sparse representation helps, but storing weights and colors for each voxel can be significant for high-resolution scenes.
*   **Performance:** The method is simple and fast for initialization [S1]. However, the iterative refinement can be computationally expensive due to repeated rendering and error calculation.
*   **Strengths:** Handles non-Lambertian reflections well by using per-view minimum cost [S1]. The iterative refinement improves visual quality for view synthesis by smoothing sharp corners and removing noise [S1, S2].
*   **Limitations:** Sensitive to calibration errors and resolution mismatches between views, requiring window-based error calculation [S2]. The average color method can cause object blur if color components differ significantly between views [S2].
*   **When to use:** Suitable for sparse multi-view setups where high-quality virtual view synthesis is required, and computational resources allow for iterative refinement.
*   **When not to use:** Not suitable for real-time applications due to the iterative nature. Not ideal for dense, highly dynamic scenes where rapid updates are needed.

## Background
The method was proposed by N. Anantrasirichai, C. Nishan Canagarajah, David W. Redmill, and David R. Bull from the University of Bristol [S2]. It was presented at the IEEE International Conference on Image Processing (ICIP) in 2006 [S2]. The work focuses on volumetric representation for sparse multi-view images of non-Lambertian scenes, aiming to provide a basis for realistic view synthesis [S2]. The approach builds on the idea that depth registration provides a good representation for natural scenery [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
[S2] Volumetric Representation for Sparse Multi-Views, https://doi.org/10.1109/icip.2006.312545, fetched 2026-10-26
