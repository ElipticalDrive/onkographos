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
import math

def single_2d_projection_voxelization(triangle, resolution, bb_min, bb_max):
    # triangle: list of 3 vertices, each (x, y, z)
    # resolution: (Rx, Ry, Rz)
    # bb_min, bb_max: (x, y, z)
    
    # 1. Identify dominant axis
    n = cross(triangle[1] - triangle[0], triangle[2] - triangle[0])
    dominant_axis = 0 if abs(n[0]) >= abs(n[1]) and abs(n[0]) >= abs(n[2]) else (1 if abs(n[1]) >= abs(n[2]) else 2)
    
    # 2. Project and Rasterize
    proj_tri = drop_axis(triangle, dominant_axis)
    pixels = rasterize_2d(proj_tri, resolution[dominant_axis])
    
    voxel_grid = {}
    scale = resolution[dominant_axis] / (bb_max[dominant_axis] - bb_min[dominant_axis])
    offset = -bb_min[dominant_axis] * scale
    
    for u, v in pixels:
        center_2d = (u + 0.5, v + 0.5)
        center_3d = project_2d_to_3d(proj_tri, dominant_axis, center_2d)
        
        if center_3d is not None:
            # 3. Pixel center covered
            depth_tri = center_3d[dominant_axis]
            idx = int(math.floor(depth_tri * scale + offset))
            voxel_idx = list(map_to_voxel_2d(center_3d, resolution, bb_min, bb_max))
            voxel_idx[dominant_axis] = idx
            voxel_grid[tuple(voxel_idx)] = True
        else:
            # 4. Edge pixel: triangle covers pixel but not center
            # Horizontal test
            h_extents = get_line_extents(proj_tri, dominant_axis, center_2d, horizontal=True)
            if h_extents:
                clamped_h = clamp_extents(h_extents, u, v)
                depths_h = sorted([project_2d_to_3d(proj_tri, dominant_axis, p)[dominant_axis] for p in clamped_h])
                voxel_idx = int(math.floor((u + 0.5) * scale + offset))
                voxel_center_depth = (voxel_idx + 0.5) / scale + bb_min[dominant_axis]
                
                if depths_h[0] < voxel_center_depth < depths_h[1]:
                    for p in clamped_h:
                        p3d = project_2d_to_3d(proj_tri, dominant_axis, p)
                        voxel_grid[tuple(map_to_voxel_2d(p3d, resolution, bb_min, bb_max))] = True
                else:
                    # Vertical test
                    v_extents = get_line_extents(proj_tri, dominant_axis, center_2d, horizontal=False)
                    if v_extents:
                        clamped_v = clamp_extents(v_extents, u, v)
                        depths_v = sorted([project_2d_to_3d(proj_tri, dominant_axis, p)[dominant_axis] for p in clamped_v])
                        if depths_v[0] < voxel_center_depth < depths_v[1]:
                            for p in clamped_v:
                                p3d = project_2d_to_3d(proj_tri, dominant_axis, p)
                                voxel_grid[tuple(map_to_voxel_2d(p3d, resolution, bb_min, bb_max))] = True

    return voxel_grid

# Helper definitions
def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def drop_axis(vertices, axis):
    return [(v[i] for i in range(3) if i != axis) for v in vertices]

def rasterize_2d(tri_2d, res):
    # Standard scanline rasterization returning covered (u, v) pixel coordinates [G]
    v0, v1, v2 = tri_2d
    min_u, max_u = int(min(v[0] for v in tri_2d)), int(max(v[0] for v in tri_2d)) + 1
    min_v, max_v = int(min(v[1] for v in tri_2d)), int(max(v[1] for v in tri_2d)) + 1
    covered = []
    for u in range(min_u, max_u):
        for v in range(min_v, max_v):
            if point_in_triangle((u + 0.5, v + 0.5), tri_2d):
                covered.append((u, v))
    return covered

def point_in_triangle(p, tri):
    v0, v1, v2 = tri
    denom = (v1[0] - v0[0]) * (v2[1] - v0[1]) - (v2[0] - v0[0]) * (v1[1] - v0[1])
    if denom == 0: return False
    w0 = ((v1[1] - v0[1]) * (p[0] - v0[0]) + (v0[0] - v1[0]) * (p[1] - v0[1])) / denom
    w1 = ((v2[1] - v0[1]) * (p[0] - v0[0]) + (v0[0] - v2[0]) * (p[1] - v0[1])) / denom
    w2 = 1.0 - w0 - w1
    return w0 >= 0 and w1 >= 0 and w2 >= 0

def project_2d_to_3d(proj_tri, dominant_axis, p2d):
    # Barycentric interpolation to reconstruct 3D point at 2D location
    v0, v1, v2 = proj_tri
    full_tri = []
    for v in proj_tri:
        full_v = list(v)
        full_v.insert(dominant_axis, 0)
        full_tri.append(tuple(full_v))
    denom = (full_tri[1][0] - full_tri[0][0]) * (full_tri[2][1] - full_tri[0][1]) - (full_tri[2][0] - full_tri[0][0]) * (full_tri[1][1] - full_tri[0][1])
    if denom == 0: return None
    w0 = ((full_tri[1][1] - full_tri[0][1]) * (p2d[0] - full_tri[0][0]) + (full_tri[0][0] - full_tri[1][0]) * (p2d[1] - full_tri[0][1])) / denom
    w1 = ((full_tri[2][1] - full_tri[0][1]) * (p2d[0] - full_tri[0][0]) + (full_tri[0][0] - full_tri[2][0]) * (p2d[1] - full_tri[0][1])) / denom
    w2 = 1.0 - w0 - w1
    if w0 < 0 or w1 < 0 or w2 < 0: return None
    return (
        w0 * full_tri[0][0] + w1 * full_tri[1][0] + w2 * full_tri[2][0],
        w0 * full_tri[0][1] + w1 * full_tri[1][1] + w2 * full_tri[2][1],
        w0 * full_tri[0][2] + w1 * full_tri[1][2] + w2 * full_tri[2][2]
    )

def get_line_extents(proj_tri, dominant_axis, center_2d, horizontal):
    # Intersects triangle with line y=center_2d[1] (horizontal) or x=center_2d[0] (vertical)
    v0, v1, v2 = proj_tri
    line_val = center_2d[1] if horizontal else center_2d[0]
    line_coord = 1 if horizontal else 0
    points = []
    edges = [(v0, v1), (v1, v2), (v2, v0)]
    for a, b in edges:
        if (a[line_coord] - line_val) * (b[line_coord] - line_val) <= 0:
            t = (line_val - a[line_coord]) / (b[line_coord] - a[line_coord] + 1e-9)
            other_coord = a[1 - line_coord] + t * (b[1 - line_coord] - a[1 - line_coord])
            points.append((other_coord, line_val) if horizontal else (line_val, other_coord))
    return points

def clamp_extents(extents, u, v):
    # Clamp x to [u, u+1] and y to [v, v+1]
    return [(max(u, min(u+1, p[0])), max(v, min(v+1, p[1]))) for p in extents]

def map_to_voxel_2d(point_3d, resolution, bb_min, bb_max):
    scale = [resolution[i] / (bb_max[i] - bb_min[i]) for i in range(3)]
    offset = [-bb_min[i] * scale[i] for i in range(3)]
    return tuple(int(math.floor(point_3d[i] * scale[i] + offset[i])) for i in range(3))
```

**Parameters:**
*   **Resolution $R$:** Determines the granularity of the voxel grid [G].
*   **Bounding Box:** Defines the spatial extent of the voxel grid [G].

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** Triangles with zero area or collinear vertices may produce invalid normals [G]. Ensure normal computation handles degenerate cases by returning a default axis or skipping the triangle [G].
*   **Precision:** Depth comparisons for the edge tests must be robust to floating-point errors [G]. Use a small epsilon when comparing depths to voxel center depths [G].
*   **Boundary Conditions:** The rasterization must ensure that pixels on the boundary of the projection are correctly handled to avoid missing voxels at the edges of the bounding box [G].

## Subtopics

### [Multi-pass Projection Voxelization](14585-multi-pass-projection-voxelization.md)
Multi-pass Projection Voxelization is the initial surface voxelization step that projects input surface primitives onto 2D planes and rasterizes them into a 3D volume, producing an "incorrect" surface voxelization that includes hidden primitives. It differs from its parent, Single 2D Projection Voxelization, by using multiple orthogonal projection directions and compositing the results to handle surfaces perpendicular to the projection plane, which would otherwise have zero area and fail to rasterize.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Significantly faster than multi-view methods due to the reduction in rasterization passes. It produces 6-separating voxelization, ensuring tunnel-freeness [S1, unofficial].
*   **Limitations:** Relies on the "thinness" property, which may not hold for all geometric configurations or non-planar inputs [S1, unofficial]. It is specifically designed for planar triangles [S1, unofficial].
*   **When to Use:** When computational efficiency is critical and the input geometry consists of planar triangles [G].
*   **When Not to Use:** For non-planar surfaces or when higher-order topological guarantees beyond 6-separation are required without additional processing [G].

## Background
The method is derived from the "thinness" property of voxelization and the equivalence of 3-axis rasterization with the crosshairs target [S1, unofficial]. It was presented in the paper "A Topological Approach to Voxelization" by Laine et al. (2013) [S1, unofficial]. The approach leverages the observation that projecting along the dominant axis of the normal is sufficient to capture the unique voxel in each column for pixel centers, with edge cases handled by crosshair-segment intersection tests [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
