# Single 2D Projection Voxelization

Also called: Dominant axis voxelization, Thin voxelization via single projection

## Overview
Single 2D Projection Voxelization produces 6-separating voxel grids from planar triangles by leveraging the geometric "thinness" property [S1, unofficial]. It solves volumetric coverage with minimal computational overhead by requiring only one 2D rasterization pass along the dominant axis of the triangle's normal, rather than three orthogonal passes [S1, unofficial]. Unlike [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md), it relies on the fact that for any pixel center covered by the projection, the dominant axis projection uniquely determines the voxel in that column, handling edge cases via crosshair-segment intersection tests [S1, unofficial].

## How it works
**Inputs:** A planar triangle defined by three vertices in 3D space, a target voxel grid resolution $R = (R_x, R_y, R_z)$, and the scene's axis-aligned bounding box defined by $bb_{min}$ and $bb_{max}$ [G].
**Outputs:** A 3D voxel grid where occupied voxels are marked [G].
**Data Structures:** A 3D voxel grid of size $R_x \times R_y \times R_z$ storing occupancy and depth, and a 2D buffer for the projection plane [G].

**Algorithm Steps:**
1. Identify the dominant axis of the geometric normal vector of the planar triangle [S1, unofficial].
2. Rasterize the triangle projected into 2D along the dominant axis of the normal [S1, unofficial].
3. For pixels where the projected triangle contains the pixel center, output the voxel whose center depth is closest to the depth of the triangle evaluated at the pixel center [S1, unofficial].
4. For pixels that contain an edge of the triangle but where the triangle does not overlap the center, find the extents of the triangle on a horizontal line bisecting the pixel [S1, unofficial].
5. Clamp these horizontal extents to the horizontal pixel extents [S1, unofficial].
6. Evaluate the depth of the triangle at these two clamped points [S1, unofficial].
7. If the evaluated depths are on different sides of a voxel's center depth, output the corresponding voxel [S1, unofficial].
8. If the horizontal test fails, repeat the test for the vertical direction (vertical line bisecting the pixel) [S1, unofficial].

**Formulas:**
Coordinate mapping from the parent article transforms 3D points to voxel indices:
$$ i = \text{floor}\left( \frac{p_{clip}.xyz - bb_{min}}{bb_{max} - bb_{min}} \times R \right) $$
[G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_2d_projection_voxelization(triangle, resolution, bb_min, bb_max):
    # triangle: list of 3 vertices, each (x, y, z)
    # resolution: (Rx, Ry, Rz)
    # bb_min, bb_max: (x, y, z)
    
    # 1. Identify dominant axis
    n = cross(triangle[1] - triangle[0], triangle[2] - triangle[0])
    dominant_axis = argmax([abs(n[0]), abs(n[1]), abs(n[2])])
    
    # 2. Project and Rasterize
    proj_tri = drop_axis(triangle, dominant_axis)
    pixels = rasterize_2d(proj_tri, resolution[dominant_axis])
    
    voxel_grid = {}
    
    for u, v in pixels:
        center_2d = (u + 0.5, v + 0.5)
        center_3d = project_2d_to_3d(proj_tri, dominant_axis, center_2d)
        
        if center_3d is not None:
            # 3. Pixel center covered
            depth_tri = center_3d[dominant_axis]
            voxel_idx = map_to_voxel(center_3d, resolution, bb_min, bb_max)
            voxel_grid[voxel_idx] = True
        else:
            # 4. Edge pixel: triangle covers pixel but not center
            # Horizontal test
            h_extents = get_line_extents(proj_tri, dominant_axis, center_2d, horizontal=True)
            if h_extents:
                clamped_h = clamp_extents(h_extents, u, v)
                depths_h = [project_2d_to_3d(proj_tri, dominant_axis, p)[dominant_axis] for p in clamped_h]
                voxel_center_depth = get_voxel_center_depth(u, v, dominant_axis, resolution, bb_min, bb_max)
                
                if (depths_h[0] < voxel_center_depth < depths_h[1]) or (depths_h[0] > voxel_center_depth > depths_h[1]):
                    for p in clamped_h:
                        p3d = project_2d_to_3d(proj_tri, dominant_axis, p)
                        voxel_grid[map_to_voxel(p3d, resolution, bb_min, bb_max)] = True
                else:
                    # Vertical test
                    v_extents = get_line_extents(proj_tri, dominant_axis, center_2d, horizontal=False)
                    if v_extents:
                        clamped_v = clamp_extents(v_extents, u, v)
                        depths_v = [project_2d_to_3d(proj_tri, dominant_axis, p)[dominant_axis] for p in clamped_v]
                        if (depths_v[0] < voxel_center_depth < depths_v[1]) or (depths_v[0] > voxel_center_depth > depths_v[1]):
                            for p in clamped_v:
                                p3d = project_2d_to_3d(proj_tri, dominant_axis, p)
                                voxel_grid[map_to_voxel(p3d, resolution, bb_min, bb_max)] = True

# Helper definitions
def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def drop_axis(vertices, axis):
    return [(v[i] for i in range(3) if i != axis) for v in vertices]

def rasterize_2d(tri_2d, res):
    # Standard scanline rasterization returning covered (u, v) pixel coordinates
    # Implementation omitted for brevity; returns list of (u, v) tuples
    pass

def project_2d_to_3d(proj_tri, dominant_axis, p2d):
    # Barycentric interpolation to reconstruct 3D point at 2D location
    v0, v1, v2 = proj_tri
    # Reconstruct full 3D vertices by inserting dominant axis coordinate back
    full_tri = []
    for v in proj_tri:
        full_v = list(v)
        full_v.insert(dominant_axis, 0) # Placeholder, will be replaced
        full_tri.append(tuple(full_v))
    # Actual interpolation uses barycentric weights on 2D coords
    # Returns 3D point or None if outside triangle
    pass

def get_line_extents(proj_tri, dominant_axis, center_2d, horizontal):
    # Intersects triangle with line y=center_2d[1] (horizontal) or x=center_2d[0] (vertical)
    # Returns list of 2D intersection points
    pass

def clamp_extents(extents, u, v):
    # Clamp x to [u, u+1] and y to [v, v+1]
    return [(max(u, min(u+1, p[0])), max(v, min(v+1, p[1]))) for p in extents]

def map_to_voxel(point_3d, resolution, bb_min, bb_max):
    scale = [resolution[i] / (bb_max[i] - bb_min[i]) for i in range(3)]
    offset = [-bb_min[i] * scale[i] for i in range(3)]
    return tuple(int((point_3d[i] * scale[i]) + offset[i]) for i in range(3))

def get_voxel_center_depth(u, v, dominant_axis, resolution, bb_min, bb_max):
    # Computes the depth coordinate of the voxel center at pixel (u, v)
    scale = resolution[dominant_axis] / (bb_max[dominant_axis] - bb_min[dominant_axis])
    offset = -bb_min[dominant_axis] * scale
    return (u + 0.5) * scale + offset
```

**Parameters:**
*   **Resolution $R$:** Determines the granularity of the voxel grid [G].
*   **Bounding Box:** Defines the spatial extent of the voxel grid [G].

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** Triangles with zero area or collinear vertices may produce invalid normals [G]. Ensure normal computation handles degenerate cases by returning a default axis or skipping the triangle.
*   **Precision:** Depth comparisons for the edge tests must be robust to floating-point errors [G]. Use a small epsilon when comparing depths to voxel center depths.
*   **Boundary Conditions:** The rasterization must ensure that pixels on the boundary of the projection are correctly handled to avoid missing voxels at the edges of the bounding box [G].

## Subtopics

### [Multi-pass Projection Voxelization](14585-multi-pass-projection-voxelization.md)
Multi-pass Projection Voxelization is the initial surface voxelization step that projects input surface primitives onto 2D planes and rasterizes them into a 3D volume, producing an "incorrect" surface voxelization that includes hidden primitives. It differs from its parent, Single 2D Projection Voxelization, by using multiple orthogonal projection directions and compositing the results to handle surfaces perpendicular to the projection plane, which would otherwise have zero area and fail to rasterize.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Significantly faster than multi-view methods due to the reduction in rasterization passes. It produces 6-separating voxelization, ensuring tunnel-freeness [S1, unofficial].
*   **Limitations:** Relies on the "thinness" property, which may not hold for all geometric configurations or non-planar inputs. It is specifically designed for planar triangles [S1, unofficial].
*   **When to Use:** When computational efficiency is critical and the input geometry consists of planar triangles [G].
*   **When Not to Use:** For non-planar surfaces or when higher-order topological guarantees beyond 6-separation are required without additional processing [G].

## Background
The method is derived from the "thinness" property of voxelization and the equivalence of 3-axis rasterization with the crosshairs target [S1, unofficial]. It was presented in the paper "A Topological Approach to Voxelization" by Laine et al. (2013) [S1, unofficial]. The approach leverages the observation that projecting along the dominant axis of the normal is sufficient to capture the unique voxel in each column for pixel centers, with edge cases handled by crosshair-segment intersection tests [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
