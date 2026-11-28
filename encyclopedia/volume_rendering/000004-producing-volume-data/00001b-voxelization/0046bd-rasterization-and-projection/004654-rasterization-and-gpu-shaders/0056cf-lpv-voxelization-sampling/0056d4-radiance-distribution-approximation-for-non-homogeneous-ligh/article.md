# Radiance Distribution Approximation for Non-Homogeneous Lights

Also called: AABB Cosine Lobe Distribution

## Overview
This method approximates the radiance distribution on mesh triangles where emission direction varies across the surface, using clamped cosine lobes distributed over an Axis-Aligned Bounding Box (AABB) of the intersection area projection [S1, unofficial]. It differs from spherical harmonic approaches by using a discrete set of cosine lobes, which is sufficient for the 2-band spherical harmonics used in LPV systems [S1, unofficial].

## How it works
**Inputs:**
*   A triangle mesh with vertex directions and texture coordinates [S1, unofficial].
*   An LPV grid structure [S1, unofficial].
*   The projection face of the LPV cell where the triangle is projected [S1, unofficial].

**Outputs:**
*   A final radiance approximation with weight equal to the number of lobes, injected into LPV cells intersected along the z-axis [S1, unofficial].

**Data Structures:**
*   **AABB:** Defined by minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$ in the local coordinate system of the LPV cell face [G].
*   **Cosine Lobe:** Defined by a center direction $\mathbf{d}$ and a spread angle $\theta$ [G].

**Steps:**
1.  Compute the projection of the intersection area on the LPV face where the triangle is projected [S1, unofficial].
2.  Compute the AABB of the polygon formed by: projected vertices of the triangle inside the LPV cell face, corners of the LPV cell face inside the triangle projection, and intersection points between the edges of the LPV cell face and edges of the triangle [S1, unofficial].
3.  Compute the emission direction at each corner of the AABB by calculating barycentric coordinates of the corner inside the initial triangle and interpolating the direction information from the triangle vertices [S1, unofficial].
4.  Distribute clamped cosine lobes between the emission directions from the corners, vertically and horizontally at angular distances equal to that of the cosine lobe [S1, unofficial].
5.  Inject each generated cosine lobe into the LPV cell with a weight of 1 [S1, unofficial].
6.  Create the distribution locally and inject only the final approximation with weight equal to the number of lobes required for the approximation into all LPV cells intersected along the z-axis [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def approximate_radiance_distribution(triangle, lpv_face, lpv_grid):
    # Step 1: Project triangle vertices onto the LPV face plane
    projected_verts = [project_point_to_plane(v, lpv_face.normal, lpv_face.origin) for v in triangle.vertices]
    
    # Step 2: Compute AABB of intersection polygon
    # Clip projected triangle against LPV face rectangle to get intersection polygon
    intersection_polygon = clip_polygon_to_rectangle(projected_verts, lpv_face.bounds)
    # Add face corners inside the projected triangle
    for corner in lpv_face.corners:
        if point_in_triangle(corner, projected_verts):
            intersection_polygon.append(corner)
    # Compute AABB from all points
    aabb = compute_aabb(intersection_polygon)
    
    # Step 3: Compute emission directions for AABB corners
    aabb_corners = get_aabb_corners(aabb)
    corner_directions = []
    for corner in aabb_corners:
        # Project corner back to triangle plane to compute barycentric coordinates
        corner_on_triangle = project_point_to_plane(corner, triangle.normal, triangle.vertices[0])
        u, v, w = compute_barycentric_coordinates(corner_on_triangle, triangle.vertices)
        # Interpolate direction
        direction = u * triangle.dirs[0] + v * triangle.dirs[1] + w * triangle.dirs[2]
        corner_directions.append(normalize(direction))
        
    # Step 4: Distribute cosine lobes
    lobes = []
    spread = triangle.lobe_spread # Angular distance for distribution [S1, unofficial]
    # Distribute vertically and horizontally between corner directions
    # [G] The sources do not specify the exact interpolation grid size or scheme.
    for i in range(2):
        for j in range(2):
            # Linear interpolation of directions across the AABB grid
            dir_u = interpolate_direction(corner_directions[0], corner_directions[1], j)
            dir_v = interpolate_direction(dir_u, interpolate_direction(corner_directions[2], corner_directions[3], j), i)
            lobes.append(Lobe(center=dir_v, spread=spread, weight=1))
            
    # Step 5 & 6: Inject into LPV
    # Create distribution locally, then inject final approximation into z-intersected cells
    intersected_cells = find_cells_intersected_along_z(aabb, lpv_grid)
    final_weight = len(lobes)
    for cell in intersected_cells:
        inject_approximation(cell, final_weight)
        
    return lobes

def project_point_to_plane(point, normal, origin):
    d = dot(point - origin, normal)
    return point - d * normal

def clip_polygon_to_rectangle(verts, bounds):
    # [G] The sources do not specify the exact clipping algorithm. Standard Sutherland-Hodgman or Liang-Barsky is used.
    ...

def compute_aabb(points):
    min_x = min(p.x for p in points); max_x = max(p.x for p in points)
    min_y = min(p.y for p in points); max_y = max(p.y for p in points)
    return {'min': (min_x, min_y), 'max': (max_x, max_y)}

def get_aabb_corners(aabb):
    return [(aabb['min'][0], aabb['min'][1]), (aabb['max'][0], aabb['min'][1]),
            (aabb['min'][0], aabb['max'][1]), (aabb['max'][0], aabb['max'][1])]

def compute_barycentric_coordinates(point, verts):
    v0 = verts[1] - verts[0]; v1 = verts[2] - verts[0]; v2 = point - verts[0]
    d00 = dot(v0, v0); d01 = dot(v0, v1); d11 = dot(v1, v1)
    d20 = dot(v2, v0); d21 = dot(v2, v1)
    denom = d00 * d11 - d01 * d01
    v = (d11 * d20 - d01 * d21) / denom
    w = (d00 * d21 - d01 * d20) / denom
    u = 1.0 - v - w
    return u, v, w

def interpolate_direction(d1, d2, t):
    return normalize(d1 + t * (d2 - d1))

def find_cells_intersected_along_z(aabb, grid):
    # [G] The sources do not specify the exact grid traversal algorithm. Standard axis-aligned slab test is used.
    ...
```

**Parameters:**
*   **Cosine Lobe Spread:** The angular distance for distribution [S1, unofficial].
*   **Weight:** Each lobe has a weight of 1 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Visual Errors:** The approach can produce minor visual errors because it generates a single radiance distribution for the AABB of the intersection area projection, which can cover two LPV cells, rather than processing each intersected LPV cell separately [S1, unofficial].
*   **Precision:** Barycentric coordinate calculation must be robust to avoid artifacts at triangle boundaries [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
This method was proposed in the context of Light Propagation Volumes (LPV) to handle textured area lights by voxelizing light triangles into the LPV grid [S1, unofficial]. It builds on the idea of creating Virtual Point Lights (VPLs) to simulate direct illumination efficiently [S1, unofficial]. The approach references earlier work on spherical harmonics for radiance approximation [S1, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2022/108416/, fetched 2026-10-29 (unofficial)
