# Single 2D Projection Voxelization

Also called: Dominant axis voxelization, Thin voxelization via single projection

## Overview
Single 2D Projection Voxelization is a method for producing 6-separating (tunnel-free) voxelizations of planar triangles using only a single 2D rasterization pass along the dominant axis of the triangle's normal vector [S1, unofficial]. It leverages the "thinness" property of voxelization to avoid the multiple projections required by sibling methods like [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md) or [3-Pass Voxelization](5300-3-pass-voxelization.md). By restricting the projection to one axis, it significantly reduces computational overhead while maintaining topological correctness for thin surfaces.

## How it works
**Inputs:** A single planar triangle defined by vertices $V_0, V_1, V_2$ in 3D space; a target voxel grid resolution; the scene bounding box.
**Outputs:** A set of voxel coordinates $(i, j, k)$ that form a 6-separating representation of the triangle.

**Data Structures:**
*   **Triangle:** Defined by 3 vertices.
*   **Voxel Grid:** 3D array or hash map of integer coordinates.
*   **Pixel Buffer:** 2D buffer storing rasterization results (e.g., depth, edge flags).

**Algorithm Steps:**

1.  **Identify Dominant Axis:**
    Calculate the geometric normal vector $\mathbf{n} = (n_x, n_y, n_z)$ of the triangle. Determine the dominant axis $A$ as the component with the largest absolute value:
    $$ A = \arg\max_{c \in \{x,y,z\}} |n_c| $$
    The projection direction is parallel to axis $A$. The 2D rasterization plane is orthogonal to $A$.

2.  **Project and Rasterize:**
    Project the triangle vertices onto the 2D plane orthogonal to $A$. Rasterize the projected triangle into a 2D pixel buffer. For each pixel $p$ covered by the projection:
    *   Determine if the pixel center $C_p$ lies inside the projected triangle.
    *   Determine if the pixel contains an edge of the triangle but does not contain the center.

3.  **Process Pixels Containing the Center:**
    For any pixel $p$ where the projected triangle contains the pixel center $C_p$:
    *   Evaluate the depth $d_{tri}$ of the triangle at $C_p$ along the projection axis $A$.
    *   Determine the voxel index $k$ along axis $A$ corresponding to the voxel center closest to $d_{tri}$.
    *   Output the voxel $(i, j, k)$ where $(i, j)$ are the 2D coordinates of pixel $p$.
    *   *Note:* This corresponds to intersecting the triangle against the crosshair segment parallel to the projection axis [S1, unofficial].

4.  **Process Pixels Containing Edges (Non-Center):**
    For pixels $p$ that contain an edge of the triangle but do not contain the center:
    *   **Horizontal Test:**
        *   Find the intersection points of the triangle edges with the horizontal line bisecting the pixel. Let these points be $H_1$ and $H_2$.
        *   Clamp $H_1$ and $H_2$ to the horizontal extents of the pixel.
        *   Evaluate the depth of the triangle at $H_1$ and $H_2$ along axis $A$. Let these depths be $d_1$ and $d_2$.
        *   Let $d_{vox}$ be the depth of the voxel center corresponding to pixel $p$ along axis $A$.
        *   If $d_1$ and $d_2$ are on different sides of $d_{vox}$ (i.e., $(d_1 - d_{vox})(d_2 - d_{vox}) < 0$), output the voxel $(i, j, k)$.
    *   **Vertical Test (if Horizontal Test fails):**
        *   If the horizontal test does not result in a voxel output, repeat the process using the vertical line bisecting the pixel.
        *   Find intersections $V_1, V_2$ with the vertical bisector.
        *   Clamp to vertical pixel extents.
        *   Evaluate depths $d_1, d_2$ at these points.
        *   If $d_1$ and $d_2$ are on different sides of $d_{vox}$, output the voxel $(i, j, k)$.
    *   *Note:* These tests correspond to testing the triangle against the two crosshair segments perpendicular to the projection axis [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_2d_projection_voxelization(triangle, grid_resolution, bounding_box):
    # 1. Identify dominant axis
    normal = compute_normal(triangle)
    abs_normal = abs(normal)
    dominant_axis = argmax(abs_normal) # 0 for x, 1 for y, 2 for z
    
    # 2. Project and Rasterize
    # Project vertices onto plane orthogonal to dominant_axis
    projected_vertices = project_onto_plane(triangle.vertices, dominant_axis)
    
    # Rasterize to get pixel coverage and edge info
    pixels = rasterize_triangle(projected_vertices, grid_resolution)
    
    voxels = set()
    
    for pixel in pixels:
        pixel_center = pixel.center
        voxel_center_depth = get_voxel_center_depth(pixel, dominant_axis, bounding_box)
        
        # 3. Check if pixel center is inside projected triangle
        if point_in_triangle(pixel_center, projected_vertices):
            # Evaluate triangle depth at pixel center
            tri_depth_at_center = evaluate_triangle_depth(triangle, pixel_center, dominant_axis)
            
            # Output voxel closest to this depth
            voxel_index = round_to_voxel_index(tri_depth_at_center, dominant_axis, bounding_box)
            voxels.add(pixel.grid_coords + (voxel_index,))
            
        else:
            # 4. Check edges for pixels containing edges but not center
            
            # Horizontal test
            h_points = get_edge_intersections_on_line(triangle, pixel, line='horizontal')
            if h_points:
                # Clamp to pixel horizontal extents
                h_clamped = clamp_to_pixel_extents(h_points, pixel, axis='horizontal')
                
                # Evaluate depths
                d1 = evaluate_triangle_depth(triangle, h_clamped[0], dominant_axis)
                d2 = evaluate_triangle_depth(triangle, h_clamped[1], dominant_axis)
                
                # Check if depths are on different sides of voxel center
                if (d1 - voxel_center_depth) * (d2 - voxel_center_depth) < 0:
                    voxels.add(pixel.grid_coords + (voxel_index,)) # voxel_index derived from pixel grid coords
                    continue # Voxel found, skip vertical test
            
            # Vertical test (if horizontal failed)
            v_points = get_edge_intersections_on_line(triangle, pixel, line='vertical')
            if v_points:
                # Clamp to pixel vertical extents
                v_clamped = clamp_to_pixel_extents(v_points, pixel, axis='vertical')
                
                # Evaluate depths
                d1 = evaluate_triangle_depth(triangle, v_clamped[0], dominant_axis)
                d2 = evaluate_triangle_depth(triangle, v_clamped[1], dominant_axis)
                
                # Check if depths are on different sides of voxel center
                if (d1 - voxel_center_depth) * (d2 - voxel_center_depth) < 0:
                    voxels.add(pixel.grid_coords + (voxel_index,))

    return voxels
```

**Parameters:**
*   **Dominant Axis Selection:** Based on the geometric normal. No tuning parameters.
*   **Voxel Resolution:** Determined by the target grid resolution.

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** Triangles with zero area or normal magnitude near zero may have ambiguous dominant axes. [G] In practice, such triangles should be skipped or handled via a fallback method.
*   **Grazing Angles:** While the method relies on the dominant axis, extremely grazing angles might result in very thin projections. The edge tests (horizontal/vertical) are crucial here to ensure 6-separation.
*   **Precision:** Depth evaluations must be precise enough to determine if they lie on different sides of the voxel center. Floating-point errors near the voxel center boundary can lead to missing or extra voxels.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels covered by the triangle's projection. Significantly faster than multi-view methods which require $O(3N)$ or more.
*   **Memory Complexity:** $O(1)$ auxiliary space per triangle (excluding the output voxel grid).
*   **Strengths:** High performance due to single projection; produces topologically correct (6-separating) results for thin surfaces.
*   **Limitations:** Only applicable to planar triangles or surfaces that can be approximated by planar triangles with well-defined dominant axes. Not suitable for complex, non-planar, or highly curved surfaces without significant tessellation.
*   **When to use:** When performance is critical and the input geometry consists of planar triangles (e.g., meshes).
*   **When not to use:** For non-planar primitives or when multiple views are already available for other reasons.

## Background
This method is derived from the topological approach to voxelization, specifically leveraging the "thinness" property and its equivalence to the crosshairs target [S1, unofficial]. It was presented as a corollary to the 3-axis rasterization methods [FBP09, CNS*11] in the paper "A Topological Approach to Voxelization" by Laine et al. (NVIDIA, 2013) [S1, unofficial]. The method simplifies the voxelization process by reducing the number of required projections from three to one, based on the observation that for thin surfaces, the dominant axis projection captures all necessary voxel information for 6-separation.

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
