# Single 2D Projection Voxelization

Also called: Dominant axis voxelization, Thin voxelization via single projection

## Overview
This method produces a 6-separating (tunnel-free) voxelization of a planar triangle using only a single 2D projection along the dominant axis of the triangle's normal vector [S1, unofficial]. It leverages the "thinness" property of voxelization to avoid the multiple projections required by sibling methods like [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md) or [3-Pass Voxelization](5300-3-pass-voxelization.md). By projecting along the normal's dominant axis, the algorithm ensures that every pixel whose center lies within the projected triangle yields a unique voxel in the corresponding column, while edge cases are resolved via crosshair-segment intersection tests [S1, unofficial].

## How it works
**Inputs:** A single planar triangle defined by vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ in 3D space; a target voxel grid resolution $R = (R_x, R_y, R_z)$; and a bounding box defining the mapping from world coordinates to voxel indices.
**Outputs:** A set of voxel coordinates $(i, j, k)$ occupied by the triangle.

**Data Structures:**
*   **Voxel Grid:** A 3D array or hash map storing occupied voxels.
*   **Triangle:** Defined by 3 vertices.
*   **Pixel:** Defined by integer screen coordinates $(u, v)$ and center $(u+0.5, v+0.5)$.

**Steps:**

1.  **Identify Dominant Axis:** Compute the geometric normal vector $\mathbf{n} = (\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0)$. Determine the dominant axis $A \in \{x, y, z\}$ corresponding to the component of $\mathbf{n}$ with the largest absolute magnitude. If $|n_A| < \epsilon$, the triangle is degenerate or perpendicular to all axes; handle as degenerate [G].
2.  **Project and Rasterize:** Project the triangle onto the 2D plane perpendicular to axis $A$. Rasterize this 2D projection to identify all pixels $(u, v)$ that the triangle covers.
3.  **Center Test (Primary Voxels):** For each pixel $(u, v)$ where the triangle contains the pixel center $C_{uv}$:
    *   Evaluate the depth of the triangle at $C_{uv}$ along axis $A$. Let this be $d_{center}$.
    *   Map $d_{center}$ to the voxel index $k$ along axis $A$.
    *   Output the voxel $(i, j, k)$ where $(i, j)$ are the voxel indices corresponding to $(u, v)$ in the other two dimensions. This corresponds to intersecting the triangle against the crosshair segment parallel to the projection axis [S1, unofficial].
4.  **Edge Test (Boundary Voxels):** For each pixel $(u, v)$ that contains an edge of the triangle but does *not* contain the pixel center:
    *   **Horizontal Test:** Find the intersection of the triangle with the horizontal line bisecting the pixel (line $v+0.5$). This yields two extents (x-coordinates) $x_{min}$ and $x_{max}$ within the pixel bounds.
    *   Clamp $x_{min}$ and $x_{max}$ to the horizontal pixel extents $[u, u+1]$.
    *   Evaluate the depth of the triangle at these two clamped points, yielding $d_{left}$ and $d_{right}$.
    *   Determine the voxel center depth $d_{voxel}$ for the voxel column at $(u, v)$.
    *   If $d_{left}$ and $d_{right}$ are on different sides of $d_{voxel}$ (i.e., $(d_{left} - d_{voxel}) \times (d_{right} - d_{voxel}) < 0$), output the corresponding voxel.
    *   **Vertical Test:** If the horizontal test fails (values are on the same side or one is equal), repeat the test using the vertical line bisecting the pixel (line $u+0.5$) to find vertical extents and evaluate depths [S1, unofficial]. This corresponds to testing the triangle against the two crosshair segments perpendicular to the projection axis [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_2d_projection_voxelization(triangle, resolution, bounding_box):
    """
    Voxelize a single triangle using one 2D projection along the dominant normal axis.
    Returns a set of voxel coordinates (i, j, k).
    """
    occupied_voxels = set()
    
    # 1. Identify dominant axis
    v0, v1, v2 = triangle.vertices
    normal = cross(v1 - v0, v2 - v0)
    abs_normal = abs(normal)
    
    # Determine dominant axis index (0=x, 1=y, 2=z)
    dominant_axis_idx = argmax(abs_normal)
    
    # Define projection plane axes (the two axes perpendicular to dominant)
    other_axes = [i for i in range(3) if i != dominant_axis_idx]
    
    # 2. Project triangle to 2D
    # Project vertices onto the plane defined by other_axes
    projected_vertices = [project_onto_plane(v, dominant_axis_idx) for v in [v0, v1, v2]]
    
    # Determine 2D bounding box in screen space
    min_u, max_u = min(p[other_axes[0]] for p in projected_vertices), max(p[other_axes[0]] for p in projected_vertices)
    min_v, max_v = min(p[other_axes[1]] for p in projected_vertices), max(p[other_axes[1]] for p in projected_vertices)
    
    # 3. Rasterize and process pixels
    for u in range(floor(min_u), ceil(max_u) + 1):
        for v in range(floor(min_v), ceil(max_v) + 1):
            pixel_center_2d = (u + 0.5, v + 0.5)
            
            # Check if pixel center is inside projected triangle
            if point_in_triangle(pixel_center_2d, projected_vertices):
                # Center Test
                # Reconstruct 3D point at pixel center
                p3d_center = reconstruct_3d_from_2d_and_axis(pixel_center_2d, dominant_axis_idx, triangle)
                depth_center = p3d_center[dominant_axis_idx]
                
                # Map to voxel index
                k = world_to_voxel_index(depth_center, dominant_axis_idx, resolution, bounding_box)
                i = world_to_voxel_index(pixel_center_2d[0], other_axes[0], resolution, bounding_box)
                j = world_to_voxel_index(pixel_center_2d[1], other_axes[1], resolution, bounding_box)
                
                occupied_voxels.add((i, j, k))
            else:
                # Edge Test: Check if triangle intersects pixel boundary
                if triangle_intersects_pixel_edge(projected_vertices, u, v):
                    # Horizontal Test
                    # Find intersection of triangle with horizontal line v+0.5
                    h_extents = get_triangle_line_intersection(projected_vertices, y_line=v+0.5)
                    
                    if h_extents:
                        x1, x2 = h_extents
                        # Clamp to pixel horizontal extents [u, u+1]
                        x1_clamped = clamp(x1, u, u + 1)
                        x2_clamped = clamp(x2, u, u + 1)
                        
                        # Evaluate depth at clamped points
                        p3d_left = reconstruct_3d_from_2d_and_axis((x1_clamped, v+0.5), dominant_axis_idx, triangle)
                        p3d_right = reconstruct_3d_from_2d_and_axis((x2_clamped, v+0.5), dominant_axis_idx, triangle)
                        
                        d_left = p3d_left[dominant_axis_idx]
                        d_right = p3d_right[dominant_axis_idx]
                        
                        # Get voxel center depth for this column
                        voxel_center_depth = get_voxel_center_depth(u, v, dominant_axis_idx, resolution, bounding_box)
                        
                        # Check if depths are on different sides of voxel center
                        if (d_left - voxel_center_depth) * (d_right - voxel_center_depth) < 0:
                            k = world_to_voxel_index(voxel_center_depth, dominant_axis_idx, resolution, bounding_box)
                            i = world_to_voxel_index(u, other_axes[0], resolution, bounding_box)
                            j = world_to_voxel_index(v, other_axes[1], resolution, bounding_box)
                            occupied_voxels.add((i, j, k))
                            continue # Skip vertical test if horizontal succeeded
                    
                    # Vertical Test (if horizontal failed)
                    # Find intersection of triangle with vertical line u+0.5
                    v_extents = get_triangle_line_intersection(projected_vertices, x_line=u+0.5)
                    
                    if v_extents:
                        y1, y2 = v_extents
                        # Clamp to pixel vertical extents [v, v+1]
                        y1_clamped = clamp(y1, v, v + 1)
                        y2_clamped = clamp(y2, v, v + 1)
                        
                        # Evaluate depth at clamped points
                        p3d_top = reconstruct_3d_from_2d_and_axis((u+0.5, y1_clamped), dominant_axis_idx, triangle)
                        p3d_bottom = reconstruct_3d_from_2d_and_axis((u+0.5, y2_clamped), dominant_axis_idx, triangle)
                        
                        d_top = p3d_top[dominant_axis_idx]
                        d_bottom = p3d_bottom[dominant_axis_idx]
                        
                        voxel_center_depth = get_voxel_center_depth(u, v, dominant_axis_idx, resolution, bounding_box)
                        
                        if (d_top - voxel_center_depth) * (d_bottom - voxel_center_depth) < 0:
                            k = world_to_voxel_index(voxel_center_depth, dominant_axis_idx, resolution, bounding_box)
                            i = world_to_voxel_index(u, other_axes[0], resolution, bounding_box)
                            j = world_to_voxel_index(v, other_axes[1], resolution, bounding_box)
                            occupied_voxels.add((i, j, k))

    return occupied_voxels
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R^2)$ in the worst case, where $N$ is the number of triangles and $R$ is the resolution, as each triangle is rasterized to a 2D grid. However, it is significantly faster than multi-view methods because it performs only one projection pass per triangle instead of three.
*   **Memory Complexity:** $O(R^3)$ for the voxel grid. The algorithm itself requires only $O(1)$ auxiliary space per triangle during rasterization.
*   **Strengths:** Reduces computational cost by eliminating two of the three required projections. Guarantees 6-separating (tunnel-free) voxelization for planar triangles [S1, unofficial].
*   **Limitations:** Only applicable to planar triangles. Non-planar geometry must be tessellated. Degenerate triangles (zero area) or those perpendicular to the dominant axis require special handling [G].
*   **When to use:** When performance is critical and the input geometry consists of planar triangles. When tunnel-freeness is required but multi-view projection is too expensive.

## Background
This method is derived from the topological approach to voxelization, specifically leveraging the equivalence between 3-axis rasterization and the crosshairs target [S1, unofficial]. It was presented as a corollary to the thinness property of voxelization, allowing for 6-separating voxelization with a single projection [S1, unofficial]. The approach builds on earlier work in 3-axis rasterization [FBP09, CNS*11] but optimizes it by recognizing that the dominant axis projection covers the majority of voxel assignments, with edge cases handled by crosshair tests [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
