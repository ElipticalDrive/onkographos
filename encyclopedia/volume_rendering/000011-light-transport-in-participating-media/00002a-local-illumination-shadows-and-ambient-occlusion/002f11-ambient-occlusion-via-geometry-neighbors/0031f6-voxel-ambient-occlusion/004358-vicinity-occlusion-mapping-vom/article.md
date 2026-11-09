# Vicinity Occlusion Mapping (VOM)
## Overview
Vicinity Occlusion Mapping (VOM) is a technique for incorporating view-dependent ambient occlusion and halo effects into volume rendered images by utilizing two Summed Area Tables (SATs) computed from a depth map [S1]. It distinguishes itself from sibling methods like Histogram-based Voxel Ambient Occlusion by operating in image space using depth information rather than volumetric density, allowing for efficient, view-dependent shading with a fixed cost per pixel [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A depth map representing the volume rendering geometry.
*   **Output:** A color image with applied ambient occlusion darkening and halo effects.

**Data Structures:**
The method utilizes two 2D Summed Area Tables (SATs) [S1]:
1.  **Depth SAT:** Stores the accumulated depth for each pixel.
2.  **Count SAT:** Stores the number of contributed values (pixel count) in the neighborhood of each pixel.

**Formulas:**
The 2D SAT is defined such that each cell $(x, y)$ stores the sum of every cell located between the initial position $(0, 0)$ and $(x, y)$ [S1]:

$$ SAT_{image}(x, y) = \sum_{i=0}^{x} \sum_{j=0}^{y} p_{i,j} $$

where $p_{i,j}$ is the value at pixel $(i, j)$ [S1].

**Algorithm Steps:**
1.  **Compute SATs:** Compute the two 2D SATs (Depth SAT and Count SAT) incrementally during a single pass over the depth map. The computation cost increases linearly with the number of cells [S1].
2.  **Evaluate Region Sums:** For a given pixel and a defined neighborhood region, evaluate the sum of depths and the count of pixels using only four texture accesses to the corners of the region in the respective SATs [S1].
3.  **Determine Occlusion Status:** Calculate the average depth of the neighborhood by dividing the sum of depths by the count of pixels [S1].
    *   If the average depth is larger than the current pixel's depth, the neighboring structures are further away, and the current structure is **not occluded** [S1].
    *   If the average depth is smaller than or equal to the current pixel's depth, the structure is **occluded** [S1].
4.  **Calculate Darkening:** If occluded, determine the amount of darkening based on the differences of depths around the corresponding pixel [S1].
5.  **Render Halo Effect:** Consider the average depth from the neighborhood to determine if a halo effect is needed [S1].
    *   If the structure in the pixel lies outside the object (implied by the depth relationship), render a halo color that decays with respect to the distance to the object [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vom(depth_map, neighborhood_radius):
    """
    Computes Vicinity Occlusion Mapping for a given depth map.
    neighborhood_radius: The radius of the square neighborhood to consider.
    """
    height, width = depth_map.shape
    
    # Initialize SATs
    # depth_sat stores accumulated depth
    # count_sat stores accumulated count (number of pixels)
    depth_sat = np.zeros((height + 1, width + 1))
    count_sat = np.zeros((height + 1, width + 1))
    
    # Step 1: Compute SATs incrementally in a single pass
    for y in range(height):
        for x in range(width):
            d = depth_map[y, x]
            # Accumulate depth
            depth_sat[y+1, x+1] = d + depth_sat[y, x+1] + depth_sat[y+1, x] - depth_sat[y, x]
            # Accumulate count (each pixel contributes 1)
            count_sat[y+1, x+1] = 1 + count_sat[y, x+1] + count_sat[y+1, x] - count_sat[y, x]
            
    output_image = np.zeros((height, width, 3)) # Assuming RGB output
    
    # Step 2-5: Evaluate each pixel
    for y in range(height):
        for x in range(width):
            current_depth = depth_map[y, x]
            
            # Define region corners for the neighborhood
            # Assuming a square neighborhood of radius r centered at (x,y)
            # Corners in SAT coordinates (1-based)
            x1 = max(0, x - neighborhood_radius)
            y1 = max(0, y - neighborhood_radius)
            x2 = min(width - 1, x + neighborhood_radius)
            y2 = min(height - 1, y + neighborhood_radius)
            
            # Map to SAT indices (1-based)
            # Top-left corner of region in image is (x1, y1) -> SAT index (x1+1, y1+1)
            # Bottom-right corner of region in image is (x2, y2) -> SAT index (x2+1, y2+1)
            # To get sum of region [x1, x2] x [y1, y2], we use:
            # SAT(x2+1, y2+1) - SAT(x1, y2+1) - SAT(x2+1, y1) + SAT(x1, y1)
            
            # Note: The source says "four texture accesses to the corners".
            # Standard 2D SAT query for rect [x1, x2] x [y1, y2] uses 4 corners.
            
            # Calculate sum of depths in region
            sum_depth = (depth_sat[y2+1, x2+1] - depth_sat[y1, x2+1] - 
                         depth_sat[y2+1, x1] + depth_sat[y1, x1])
            
            # Calculate count of pixels in region
            count_pixels = (count_sat[y2+1, x2+1] - count_sat[y1, x2+1] - 
                            count_sat[y2+1, x1] + count_sat[y1, x1])
            
            # Avoid division by zero
            if count_pixels == 0:
                continue
                
            avg_depth = sum_depth / count_pixels
            
            # Step 3: Determine occlusion
            is_occluded = avg_depth <= current_depth
            
            # Step 4: Calculate darkening if occluded
            if is_occluded:
                # Darkening based on differences of depths
                # Source does not specify exact formula for darkening amount.
                # [G] Assuming a simple linear relationship with depth difference for implementation completeness.
                depth_diff = current_depth - avg_depth
                occlusion_factor = min(1.0, depth_diff * scale_factor) # scale_factor is a parameter
                darkening = 1.0 - occlusion_factor
            else:
                darkening = 1.0
                
            # Step 5: Render halo effect
            # If structure lies outside the object, render halo color decaying with distance.
            # [G] The condition "outside the object" is not explicitly defined in the source text 
            # beyond the depth comparison. We assume if not occluded by closer geometry, 
            # it might be on the boundary. However, the source links halo to "average depth from neighborhood".
            # A common interpretation is that if the pixel is on the surface, the neighborhood average 
            # might be deeper (inside) or shallower (outside).
            # The source says: "If the structure in the pixel lies outside the object, a halo color is rendered".
            # This is ambiguous without a definition of "outside". 
            # [G] We will implement a placeholder for halo calculation based on distance to object boundary.
            # Distance to object is often approximated by depth.
            halo_color = np.array([0.5, 0.5, 0.5]) # Placeholder halo color
            distance_to_object = current_depth # Placeholder
            halo_intensity = max(0.0, 1.0 - distance_to_object / max_depth) # Decay with distance
            
            # Combine darkening and halo
            # [G] Standard blending: base color * occlusion + halo * halo_intensity
            base_color = get_base_color(x, y) # Placeholder for base volume color
            final_color = base_color * darkening + halo_color * halo_intensity
            
            output_image[y, x] = final_color
            
    return output_image
```

**Parameters:**
*   **Neighborhood Radius:** Determines the size of the region for average depth calculation. Not given in the sources.
*   **Scale Factor:** Used in darkening calculation. Not given in the sources.
*   **Max Depth:** Used in halo decay. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** SAT accumulation may suffer from precision loss with large depth values or large images. Not given in the sources.
*   **Boundaries:** Handling pixels near the image edge requires clamping or padding when accessing SAT corners. Not given in the sources.
*   **Degenerate Input:** Empty regions (count = 0) must be handled to avoid division by zero.

## Subtopics

### 3D Density Summed Area Table
A technique using a 3D SAT calculated from density values to provide view-independent ambient occlusion that supports semi-transparent structures [S1]. It estimates a region using 8 texture accesses in the 3D SAT structure [S1]. It is more accurate than the 2D image space approach (VOM) but has higher preprocessing cost and memory footprint [S1]. A key advantage is that the 3D SAT does not need to be recomputed if the view changes [S1].

## Operating characteristics
*   **Time Complexity:** The SAT computation is linear with the number of cells ($O(N)$) [S1]. Evaluation per pixel requires a constant number of texture accesses (4 for 2D SAT) [S1].
*   **Memory Complexity:** Requires storage for two 2D SATs of the same resolution as the depth map [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Computation cost increases linearly with the number of cells in the depth map [S1].
*   **Strengths:** Efficient evaluation with only four texture accesses per pixel; view-dependent effects; simple implementation [S1].
*   **Limitations:** View-dependent (requires recomputation if view changes); less accurate than 3D density-based methods; limited to depth information [S1].
*   **When to use:** When view-dependent ambient occlusion is desired and computational efficiency is prioritized over physical accuracy [S1].
*   **When not to use:** When view-independent results are required or when semi-transparent structures need accurate occlusion [S1].

## Background
Vicinity Occlusion Mapping was created by Diaz et al. [DYV08] and is described in the context of Interactive Volume Rendering with Volumetric Illumination [S1]. It effectively utilizes two SATs computed from the depth map to incorporate view-dependent ambient occlusion and halo effects in the volume rendered image [S1].

## Sources
[S1] Interactive Volume Rendering with Volumetric Illumination, https://doi.org/10.2312/conf/eg2012/stars/053-074, fetched 2026-10-26
