# Single 2D Projection Voxelization

Also called: Dominant axis voxelization, Thin voxelization via single projection

## Overview
Single 2D Projection Voxelization is a method for producing 6-separating (tunnel-free) voxel grids from planar triangles by leveraging the geometric "thinness" property [S1, unofficial]. It solves the problem of volumetric coverage with minimal computational overhead by requiring only a single 2D rasterization pass along the dominant axis of the triangle's normal, rather than the three orthogonal passes used in standard multi-view methods [S1, unofficial]. Unlike [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md) which projects along all three axes, this method relies on the fact that for any pixel center covered by the projection, the dominant axis projection is sufficient to determine the unique voxel in that column, handling edge cases via crosshair-segment intersection tests [S1, unofficial].

## How it works
**Inputs:** A planar triangle defined by three vertices in 3D space; a target voxel grid resolution $R = (R_x, R_y, R_z)$; and the scene's axis-aligned bounding box (AABB) defined by $bb_{min}$ and $bb_{max}$.
**Outputs:** A 3D voxel grid where occupied voxels are marked.

**Data Structures:**
*   **Voxel Grid:** A 3D array or texture of size $R_x \times R_y \times R_z$. Each voxel stores occupancy (boolean) and potentially depth information.
*   **2D Buffer:** A 2D buffer corresponding to the projection plane, storing rasterized fragments.

**Algorithm Steps:**

1.  **Identify Dominant Axis:** Compute the geometric normal vector $\mathbf{n} = (n_x, n_y, n_z)$ of the triangle. Determine the dominant axis $A$ as the component with the largest absolute value: $A = \text{argmax}_{i \in \{x,y,z\}} |n_i|$.
2.  **Project and Rasterize:** Project the triangle onto the 2D plane perpendicular to axis $A$. Rasterize this 2D projection into the 2D buffer.
3.  **Process Pixel Centers:** For each pixel $(u, v)$ in the 2D buffer where the projected triangle contains the pixel center:
    *   Evaluate the depth $d_{tri}$ of the triangle at the pixel center coordinates in 3D space.
    *   Map the pixel center and depth to a voxel index $(i, j, k)$ using the coordinate mapping formula from the parent [Multi-view projection](5937-multi-view-projection.md).
    *   Output the voxel whose center depth is closest to $d_{tri}$ [S1, unofficial].
4.  **Process Edge Pixels:** For pixels where the projected triangle contains an edge but does *not* overlap the pixel center:
    *   **Horizontal Test:** Find the extents of the triangle intersection on a horizontal line bisecting the pixel. Clamp these extents to the horizontal pixel boundaries. Evaluate the depth of the triangle at these two clamped points. If the evaluated depths lie on different sides of the voxel's center depth, output the corresponding voxel [S1, unofficial].
    *   **Vertical Test:** If the horizontal test fails (values are not on different sides), repeat the test for the vertical direction (vertical line bisecting the pixel) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_2d_projection_voxelization(triangle, resolution, bb_min, bb_max):
    # 1. Identify dominant axis
    normal = compute_triangle_normal(triangle)
    abs_normal = abs(normal)
    dominant_axis = argmax(abs_normal) # 0 for x, 1 for y, 2 for z
    
    # 2. Project and Rasterize
    # Project triangle onto 2D plane perpendicular to dominant_axis
    projected_tri = project_triangle(triangle, dominant_axis)
    rasterized_pixels = rasterize(projected_tri, resolution)
    
    # 3. Process Pixel Centers
    for pixel in rasterized_pixels:
        if pixel.contains_center:
            # Evaluate depth at pixel center
            center_3d = pixel_center_to_3d(pixel, dominant_axis)
            depth_tri = evaluate_triangle_depth(triangle, center_3d)
            
            # Map to voxel index
            voxel_idx = map_to_voxel(center_3d, depth_tri, resolution, bb_min, bb_max)
            
            # Output voxel with center depth closest to depth_tri
            set_voxel(voxel_idx, occupied=True)
            
        elif pixel.contains_edge:
            # 4. Process Edge Pixels
            # Horizontal test
            h_extents = get_triangle_extents_horizontal(pixel, dominant_axis)
            clamped_h = clamp(h_extents, pixel.horizontal_bounds)
            depths_h = [evaluate_triangle_depth(triangle, point_to_3d(p, dominant_axis)) for p in clamped_h]
            
            voxel_center_depth = get_voxel_center_depth(pixel, dominant_axis)
            
            if depths_h[0] < voxel_center_depth and depths_h[1] > voxel_center_depth or \
               depths_h[0] > voxel_center_depth and depths_h[1] < voxel_center_depth:
                set_voxel(map_to_voxel(clamped_h[0], depths_h[0], resolution, bb_min, bb_max), occupied=True)
                set_voxel(map_to_voxel(clamped_h[1], depths_h[1], resolution, bb_min, bb_max), occupied=True)
            else:
                # Vertical test
                v_extents = get_triangle_extents_vertical(pixel, dominant_axis)
                clamped_v = clamp(v_extents, pixel.vertical_bounds)
                depths_v = [evaluate_triangle_depth(triangle, point_to_3d(p, dominant_axis)) for p in clamped_v]
                
                if depths_v[0] < voxel_center_depth and depths_v[1] > voxel_center_depth or \
                   depths_v[0] > voxel_center_depth and depths_v[1] < voxel_center_depth:
                    set_voxel(map_to_voxel(clamped_v[0], depths_v[0], resolution, bb_min, bb_max), occupied=True)
                    set_voxel(map_to_voxel(clamped_v[1], depths_v[1], resolution, bb_min, bb_max), occupied=True)

def map_to_voxel(point_3d, depth, resolution, bb_min, bb_max):
    # Standard mapping from parent article
    scale = resolution / (bb_max - bb_min)
    offset = -bb_min * scale
    voxel_coords = floor((point_3d * scale) + offset)
    return voxel_coords
```

**Parameters:**
*   **Resolution $R$:** Determines the granularity of the voxel grid.
*   **Bounding Box:** Defines the spatial extent of the voxel grid.

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** Triangles with zero area or collinear vertices may produce invalid normals. [G] Ensure normal computation handles degenerate cases by returning a default axis or skipping the triangle.
*   **Precision:** Depth comparisons for the edge tests must be robust to floating-point errors. [G] Use a small epsilon when comparing depths to voxel center depths.
*   **Boundary Conditions:** The rasterization must ensure that pixels on the boundary of the projection are correctly handled to avoid missing voxels at the edges of the bounding box.

## Subtopics

### [Multi-pass Projection Voxelization](14585-multi-pass-projection-voxelization.md)
Multi-pass Projection Voxelization is the initial surface voxelization step that projects input surface primitives onto 2D planes and rasterizes them into a 3D volume, producing an "incorrect" surface voxelization that includes hidden primitives. It differs from its parent, Single 2D Projection Voxelization, by using multiple orthogonal projection directions and compositing the results to handle surfaces perpendicular to the projection plane, which would otherwise have zero area and fail to rasterize.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels in the projection buffer, as it requires only one rasterization pass per triangle.
*   **Memory Complexity:** $O(R_x R_y R_z)$ for the voxel grid.
*   **Strengths:** Significantly faster than multi-view methods due to the reduction in rasterization passes. It produces 6-separating voxelization, ensuring tunnel-freeness.
*   **Limitations:** Relies on the "thinness" property, which may not hold for all geometric configurations or non-planar inputs. It is specifically designed for planar triangles.
*   **When to Use:** When computational efficiency is critical and the input geometry consists of planar triangles.
*   **When Not to Use:** For non-planar surfaces or when higher-order topological guarantees beyond 6-separation are required without additional processing.

## Background
The method is derived from the "thinness" property of voxelization and the equivalence of 3-axis rasterization with the crosshairs target [S1, unofficial]. It was presented in the paper "A Topological Approach to Voxelization" by Laine et al. (2013) [S1, unofficial]. The approach leverages the observation that projecting along the dominant axis of the normal is sufficient to capture the unique voxel in each column for pixel centers, with edge cases handled by crosshair-segment intersection tests.

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
