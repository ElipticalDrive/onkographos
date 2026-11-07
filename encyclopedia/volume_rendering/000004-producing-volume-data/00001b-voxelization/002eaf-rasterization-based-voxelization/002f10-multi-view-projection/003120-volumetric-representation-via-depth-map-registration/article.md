# Volumetric Representation via Depth Map Registration

Also called: Sparse Multi-View Volumetric Representation, Sparse Non-Lambertian Multi-view Volume Construction, Volume Initialization

## Overview
This method constructs a 3D volumetric representation from a sparse set of calibrated multi-view images by registering estimated depth maps into a voxel grid [S1, S2]. It specifically addresses non-Lambertian reflections by using individual minimum cost per view for depth estimation rather than a global threshold, and represents scene geometry as a sparse voxel cloud where voxel values indicate the number of views mapping to that location [S1, S2]. It distinguishes itself from static voxelization by incorporating an iterative refinement loop that cleans scattering voxels and smooths sharp corners to increase coding gain, ensuring the final volume accurately represents the real scene for virtual view synthesis [S1, S2].

## How it works
**Inputs:** A set of $N$ calibrated reference views (images and camera parameters) of a non-Lambertian scene.
**Outputs:** A sparse 3D voxel grid where each active voxel contains a position, a view-count weight, and a color value.

**Data Structures:**
*   **Voxel Grid:** A 3D array or sparse hash map.
*   **Voxel Structure:**
    *   `position`: $(i, j, k)$ integer coordinates.
    *   `weight` ($\delta$): Integer count of views mapping to this voxel.
    *   `color`: $(Y, U, V)$ or $(R, G, B)$ components.
    *   `active`: Boolean flag indicating if the voxel is part of the surface.

**Algorithm Steps:**

1.  **Depth Estimation:** For each reference view, estimate a depth map by searching along the view's geometry. Compensate for non-Lambertian reflection by exploiting the individual minimum cost for that view [S1].

2.  **Volume Initialization (Registration):** Register the estimated depth maps into the 3D volume. For each voxel at location $(i, j, k)$, calculate its value $\delta_{ijk}$ based on the number of views mapping to it [S1, S2]:
    $$
    \delta_{ijk} = \begin{cases} 
    n & \text{if registering depth} \\
    0 & \text{otherwise}
    \end{cases}
    $$
    where $n \in \{1, 2, ..., N\}$ is the number of views mapped to voxel $x$, and $N$ is the total number of reference views [S1, S2].

3.  **Isolated Voxel Removal:** Detect and remove isolated voxels that have no neighbors, as they are likely errors [S1, S2].

4.  **Re-projection and Depth Refinement:** Project the current volume back to the reference view geometry to build new depth maps. This step incorporates depth information from other views, removing errors from the previous step [S1, S2].
    *   If multiple depth values exist for a pixel, consider the front cluster of depths [S1, S2].
    *   Calculate the proper depth value $d$ for a pixel using the weighted average of the front cluster [S1, S2]:
        $$
        d = \begin{cases} 
        d_n & \text{if only one depth has } \delta_n = \max(\delta) \\
        \frac{\sum_{k=1}^{K} \delta_k d_k}{\sum_{k=1}^{K} \delta_k} & \text{otherwise}
        \end{cases}
        $$
        where $K$ is the number of depth values in the front cluster, $d_k$ is the depth value, and $\delta_k$ is the voxel weight [S1, S2].

5.  **Interpolation:** Reconstruct the volume again. Interpolate voxels to connect the surface of the volume among near voxels, necessary because reference views may have different resolutions [S1, S2].

6.  **Iterative Refinement:**
    *   **Temporary Coloring:** Assign temporary colors to voxels by taking the median of color values from all visible views. The median is used because it is less sensitive to extreme values than the mean [S2].
    *   **Error Calculation:** Iterate through each reference view (e.g., left to right). Calculate the Mean Square Error (MSE) between the reconstructed view and the original reference image. Use a window-based computation (e.g., $3 \times 3$ pixels) to handle imperfect camera calibration and unequal resolutions [S2].
    *   **Voxel Update:** If the MSE exceeds a threshold:
        1.  Remove the old depth and its associated voxel [S2].
        2.  Investigate a new depth derived from neighboring pixels based on a smooth surface assumption [S2].
        3.  If the new depth satisfies color constraints for all visible views, generate the new associated voxel [S2].
    *   **Reconstruction:** If voxels are removed or generated, iteratively reconstruct the images, as voxel position changes affect visibility (invisible to visible or vice versa) [S2].

7.  **Color Selection (Post-processing):**
    *   For each voxel, determine colors from visible views.
    *   Blend colors from two views 1 and 2 using angle-based weighting [S2]:
        $$
        C = \begin{cases} 
        \frac{a_1 C_2 + a_2 C_1}{a_1 + a_2} & \text{if condition is true} \\
        C_1 & \text{otherwise}
        \end{cases}
        $$
        where $C_1, C_2$ are intensity (Y) or color (U,V) components, and $a_1, a_2$ are angles between the voxel $x$ and points on lines $L_1, L_2$ respectively [S2].
    *   **Blur Handling:** If the difference between $C_1$ and $C_2$ is significant, the average method causes object blur [S2]. In this case, place the color in a "blur group." Replace the color with that of a neighboring voxel in a "clear group" (where color difference is small) that has the closest value to the original color to preserve sharpness [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_representation_via_depth_map_registration(reference_views):
    """
    reference_views: List of dicts containing 'image', 'camera_params', 'depth_map' (estimated).
    Returns: SparseVoxelGrid
    """
    N = len(reference_views)
    voxel_grid = SparseVoxelGrid()
    
    # Step 1 & 2: Initialization
    for view in reference_views:
        depth_map = view['depth_map']
        for pixel in depth_map:
            voxel_pos = project_to_voxel(pixel.position, view['camera_params'])
            voxel_grid.increment_weight(voxel_pos)
            
    # Step 3: Remove isolated voxels
    voxel_grid.remove_isolated_voxels()
    
    # Step 4: Re-projection and Depth Refinement
    for view in reference_views:
        new_depth_map = {}
        for pixel in view['image'].pixels:
            depths_in_front_cluster = voxel_grid.get_depths_at_pixel(pixel.position, view['camera_params'])
            if not depths_in_front_cluster:
                continue
            
            # Calculate proper depth d
            max_delta = max(d.weight for d in depths_in_front_cluster)
            unique_max = [d for d in depths_in_front_cluster if d.weight == max_delta]
            
            if len(unique_max) == 1:
                d = unique_max[0].depth
            else:
                sum_weighted_depth = sum(d.weight * d.depth for d in depths_in_front_cluster)
                sum_weights = sum(d.weight for d in depths_in_front_cluster)
                d = sum_weighted_depth / sum_weights
            
            new_depth_map[pixel.position] = DepthValue(depth=d, weight=1) # Weight reset for new map
        
        # Register new depth map back to volume (simplified: re-run Step 2 logic with new depths)
        # Note: The source implies re-registering the *refined* depths.
        for pos, depth_val in new_depth_map.items():
            voxel_pos = project_to_voxel(pos, view['camera_params'])
            voxel_grid.set_depth(voxel_pos, depth_val.depth)
            voxel_grid.increment_weight(voxel_pos)

    # Step 5: Interpolation
    voxel_grid.interpolate_near_voxels()

    # Step 6: Iterative Refinement
    converged = False
    while not converged:
        converged = True
        for view in reference_views:
            # Calculate MSE with window-based computation
            mse = calculate_windowed_mse(view['image'], voxel_grid, view['camera_params'])
            
            if mse > THRESHOLD:
                # Remove old depth/voxel
                voxel_pos = get_voxel_for_view(view, pixel) # Identify problematic voxel
                voxel_grid.remove_voxel(voxel_pos)
                converged = False # Need to re-evaluate visibility
                
                # Investigate new depth from neighbors
                new_depth = estimate_smooth_depth_from_neighbors(voxel_pos)
                if check_color_constraints(new_depth, voxel_grid, view):
                    voxel_grid.add_voxel(voxel_pos, new_depth)
                    converged = False # Need to re-evaluate

    # Step 7: Color Selection
    for voxel in voxel_grid.active_voxels():
        visible_colors = get_visible_colors(voxel, reference_views)
        voxel.color = blend_colors_with_blur_handling(visible_colors)
        
    return voxel_grid
```

## Subtopics

### [Iterative Volume Refinement](6473-iterative-volume-refinement.md)
Iterative Volume Refinement is a post-initialization algorithm that cleans and smooths a sparse voxel cloud generated from multi-view depth maps by removing scattering voxels and smoothing sharp corners to increase coding gain. It distinguishes itself from static voxelization by dynamically regenerating voxels based on a smooth surface assumption and iteratively re-evaluating visibility, ensuring the final volume accurately represents the real scene for virtual view synthesis.

## Operating characteristics
*   **Time Complexity:** The method involves multiple passes over the voxel grid and reference views. The iterative refinement step (Step 6) can be computationally expensive as it requires full view reconstruction and MSE calculation for every iteration until convergence [S2].
*   **Memory Complexity:** The volume is sparse, storing only voxels with $\delta > 0$. Memory usage scales with the number of visible surface voxels and the number of views $N$ [S1, S2].
*   **Strengths:** Handles non-Lambertian reflections effectively by using per-view minimum cost [S1]. The iterative refinement reduces sharp corners and scattering noise, improving coding gain for compression [S1, S2]. Window-based MSE calculation handles calibration errors [S2].
*   **Limitations:** The average color blending method can cause object blur if color components differ significantly between views [S2]. The iterative process may be slow for high-resolution volumes or large numbers of views [S2].
*   **When to use:** Suitable for sparse multi-view setups where non-Lambertian effects are present and virtual view synthesis is the goal [S1, S2].
*   **When not to use:** Not suitable for dense, fully covered scenes where standard dense voxelization is more appropriate, or when real-time performance is critical due to the iterative refinement overhead [S2].

## Background
The method was proposed by N. Anantrasirichai, C. Nishan Canagarajah, David W. Redmill, and David R. Bull from the University of Bristol [S2]. It was introduced in 2006 as a novel volumetric representation for sparse multi-view images of non-Lambertian scenes [S2]. The work builds on the principle that depth registration provides a good representation for natural scenery [S1]. It was developed to address the challenges of virtual view synthesis, specifically aiming to improve coding gain by smoothing the volume representation [S1, S2].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
[S2] Volumetric Representation for Sparse Multi-Views, https://doi.org/10.1109/icip.2006.312545, fetched 2026-10-26
