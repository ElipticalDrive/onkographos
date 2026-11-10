# Single 2D Projection Voxelization

Also called: Dominant axis voxelization, Thin voxelization via single projection

## Overview
Single 2D Projection Voxelization is a method that produces 6-separating (tunnel-free) voxelization using only one 2D projection along the dominant axis of a triangle's normal vector, leveraging the thinness property of the resulting voxel set [S1, unofficial]. Unlike sibling methods such as [Orthographic Multi-View Voxelization](3892-orthographic-multi-view-voxelization.md) or [3-Pass Voxelization](5300-3-pass-voxelization.md) which require projections along all three principal axes, this method reduces computational cost by determining the projection direction dynamically based on the geometry's orientation [S1, unofficial]. It builds on the [Multi-view projection](5937-multi-view-projection.md) principles but optimizes the aggregation step by exploiting the fact that for planar triangles, the dominant axis projection captures all necessary voxel columns for 6-connectivity [S1, unofficial].

## How it works
**Inputs:** A planar triangle defined by its three vertices in 3D space.
**Outputs:** A set of voxel indices $(i, j, k)$ representing the 6-separating voxelization of the triangle.
**Data Structures:**
*   **Voxel Grid:** A 3D array or hash map mapping integer coordinates $(i, j, k)$ to occupancy status. The resolution $R = (R_x, R_y, R_z)$ maps to the scene's axis-aligned bounding box (AABB) defined by $bb_{min}$ and $bb_{max}$ [Parent, S1, unofficial].
*   **Projection Plane:** A 2D coordinate system aligned with the two axes perpendicular to the dominant axis.

**Algorithm Steps:**

1.  **Identify Dominant Axis:** Calculate the geometric normal vector $\mathbf{n} = (n_x, n_y, n_z)$ of the triangle. Determine the dominant axis $A$ as the component of $\mathbf{n}$ with the largest absolute value. Let the other two axes be $B$ and $C$. The projection is performed along axis $A$ onto the $BC$-plane [S1, unofficial].

2.  **Rasterize Projection:** Project the triangle vertices onto the 2D plane defined by axes $B$ and $C$. Rasterize this 2D triangle to identify all pixels (voxel columns) that the triangle covers. For each covered pixel with center coordinates $(u, v)$ in the $BC$-plane:
    *   Map $(u, v)$ to voxel indices $(i, j, k)$ using the standard coordinate transformation: scale by resolution and translate by $bb_{min}$ [Parent, S1, unofficial].

3.  **Center Test (Primary Voxel Selection):** For pixels where the projected triangle contains the pixel center:
    *   Evaluate the depth of the triangle along the dominant axis $A$ at the pixel center. Let this depth be $d_{center}$.
    *   Determine the voxel index $k_A$ along the dominant axis such that the voxel center depth $d_{voxel}$ is closest to $d_{center}$.
    *   Mark the voxel $(i, j, k_A)$ as occupied [S1, unofficial].
    *   *Note:* This step corresponds to intersecting the triangle against the crosshair segment parallel to the projection axis [S1, unofficial].

4.  **Edge Test (Secondary Voxel Selection):** For pixels where the projected triangle contains an edge but does *not* contain the pixel center:
    *   **Horizontal Test:** Find the intersection of the triangle with the horizontal line bisecting the pixel (constant $C$-coordinate, varying $B$-coordinate). This yields two extents (min and max $B$) within the pixel.
    *   Clamp these extents to the horizontal pixel boundaries (the pixel's $B$-range).
    *   Evaluate the depth of the triangle along the dominant axis $A$ at these two clamped points. Let these depths be $d_{left}$ and $d_{right}$.
    *   Let $d_{center}$ be the depth of the voxel center along axis $A$.
    *   If $d_{left}$ and $d_{right}$ are on different sides of $d_{center}$ (i.e., $(d_{left} - d_{center}) \times (d_{right} - d_{center}) < 0$), mark the corresponding voxel as occupied [S1, unofficial].
    *   **Vertical Test:** If the horizontal test fails (values are not on different sides), repeat the test for the vertical line bisecting the pixel (constant $B$-coordinate, varying $C$-coordinate). Find extents, clamp to vertical pixel boundaries, evaluate depths $d_{top}$ and $d_{bottom}$, and check if they straddle $d_{center}$. If they do, mark the voxel as occupied [S1, unofficial].
    *   *Note:* These tests correspond to testing the triangle against the two crosshair segments perpendicular to the projection axis [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_2d_projection_voxelization(triangle, voxel_grid, bb_min, resolution):
    """
    Voxelize a single triangle using one 2D projection along the dominant normal axis.
    
    Args:
        triangle: List of 3 vertices, each a tuple (x, y, z).
        voxel_grid: A data structure supporting set_item((i,j,k), True).
        bb_min: Tuple (x_min, y_min, z_min) of the bounding box.
        resolution: Tuple (Rx, Ry, Rz) of the voxel grid dimensions.
    """
    # 1. Identify Dominant Axis
    v0, v1, v2 = triangle
    edge1 = (v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2])
    edge2 = (v2[0]-v0[0], v2[1]-v0[1], v2[2]-v0[2])
    
    # Cross product to get normal
    nx = edge1[1]*edge2[2] - edge1[2]*edge2[1]
    ny = edge1[2]*edge2[0] - edge1[0]*edge2[2]
    nz = edge1[0]*edge2[1] - edge1[1]*edge2[0]
    
    abs_nx, abs_ny, abs_nz = abs(nx), abs(ny), abs(nz)
    
    if abs_nx >= abs_ny and abs_nx >= abs_nz:
        dominant_axis = 0 # X
        axis1, axis2 = 1, 2 # Y, Z
    elif abs_ny >= abs_nx and abs_ny >= abs_nz:
        dominant_axis = 1 # Y
        axis1, axis2 = 0, 2 # X, Z
    else:
        dominant_axis = 2 # Z
        axis1, axis2 = 0, 1 # X, Y
        
    # 2. Project and Rasterize
    # Project vertices onto the plane perpendicular to dominant_axis
    projected_vertices = []
    for v in triangle:
        projected_vertices.append((v[axis1], v[axis2]))
        
    # Get pixel bounds for the projected triangle
    min_u = min(p[0] for p in projected_vertices)
    max_u = max(p[0] for p in projected_vertices)
    min_v = min(p[1] for p in projected_vertices)
    max_v = max(p[1] for p in projected_vertices)
    
    # Map to voxel indices
    min_i = int((min_u - bb_min[axis1]) * resolution[axis1])
    max_i = int((max_u - bb_min[axis1]) * resolution[axis1])
    min_j = int((min_v - bb_min[axis2]) * resolution[axis2])
    max_j = max(int((max_v - bb_min[axis2]) * resolution[axis2]), min_j + 1)
    
    # Helper to evaluate depth along dominant axis given axis1 and axis2 coords
    def get_depth(u, v):
        # Barycentric coordinates or plane equation evaluation
        # Plane equation: nx*x + ny*y + nz*z + d = 0
        # We need z (or x/y) given x,y (or x/z, y/z)
        # Let's solve for dominant_axis coordinate
        # nx*x + ny*y + nz*z = -d
        # We need d. Use v0: nx*v0[0] + ny*v0[1] + nz*v0[2] + d = 0 => d = -(nx*v0[0] + ny*v0[1] + nz*v0[2])
        d = -(nx*v0[0] + ny*v0[1] + nz*v0[2])
        
        if dominant_axis == 0: # Solve for x
            return (-d - ny*u - nz*v) / nx
        elif dominant_axis == 1: # Solve for y
            return (-d - nx*u - nz*v) / ny
        else: # Solve for z
            return (-d - nx*u - ny*v) / nz

    # Helper to map 2D pixel coords to 3D voxel indices
    def get_voxel_indices(u, v, k_dom):
        i = int((u - bb_min[axis1]) * resolution[axis1])
        j = int((v - bb_min[axis2]) * resolution[axis2])
        return (i, j, k_dom)

    # Iterate over pixels in the bounding box of the projection
    for i in range(min_i, max_i + 1):
        for j in range(min_j, max_j + 1):
            # Pixel center in projected space
            u_center = (i + 0.5 + bb_min[axis1]) / resolution[axis1]
            v_center = (j + 0.5 + bb_min[axis2]) / resolution[axis2]
            
            # Check if center is inside projected triangle
            if is_point_in_triangle((u_center, v_center), projected_vertices):
                # Center Test
                d_center = get_depth(u_center, v_center)
                # Find closest voxel center depth
                # Voxel center depth in dominant axis
                # Map d_center to voxel index k
                if dominant_axis == 0:
                    k = int((d_center - bb_min[0]) * resolution[0])
                elif dominant_axis == 1:
                    k = int((d_center - bb_min[1]) * resolution[1])
                else:
                    k = int((d_center - bb_min[2]) * resolution[2])
                
                # Check if d_center is closer to k or k+1? 
                # The source says "output the voxel whose center depth is closest"
                # We can refine k by checking distance to k and k+1 centers
                # But for simplicity, we assume integer mapping is sufficient or round
                # Let's use rounding for closest center
                if dominant_axis == 0:
                    k = round((d_center - bb_min[0]) * resolution[0])
                elif dominant_axis == 1:
                    k = round((d_center - bb_min[1]) * resolution[1])
                else:
                    k = round((d_center - bb_min[2]) * resolution[2])
                
                voxel_grid.set_item((i, j, k), True)
            else:
                # Edge Test
                # Horizontal line bisecting pixel: v = v_center
                # Find intersection of triangle with v = v_center
                h_extents = get_horizontal_extents(projected_vertices, v_center)
                if h_extents is not None:
                    u_min, u_max = h_extents
                    # Clamp to pixel extents
                    u_min_clamped = max(u_min, i / resolution[axis1] + bb_min[axis1])
                    u_max_clamped = min(u_max, (i + 1) / resolution[axis1] + bb_min[axis1])
                    
                    d_left = get_depth(u_min_clamped, v_center)
                    d_right = get_depth(u_max_clamped, v_center)
                    
                    # Voxel center depth for this column
                    if dominant_axis == 0:
                        d_voxel_center = (i + 0.5) / resolution[0] + bb_min[0]
                    elif dominant_axis == 1:
                        d_voxel_center = (i + 0.5) / resolution[0] + bb_min[0] # Wait, if dom is 0, i is x
                        # Correction: if dom is 0, the voxel index is i. The center depth is along x.
                        # But here we are evaluating depth along dominant axis.
                        # If dominant axis is 0, d_left and d_right are x-coordinates.
                        # The voxel center x-coordinate is (i + 0.5) / Rx + x_min.
                        d_voxel_center = (i + 0.5) / resolution[0] + bb_min[0]
                    elif dominant_axis == 1:
                        d_voxel_center = (j + 0.5) / resolution[1] + bb_min[1]
                    else:
                        d_voxel_center = (j + 0.5) / resolution[2] + bb_min[2]
                        
                    if (d_left - d_voxel_center) * (d_right - d_voxel_center) < 0:
                        # Output voxel
                        if dominant_axis == 0:
                            voxel_grid.set_item((i, j, k), True) # k is not used here? No, the voxel is (i,j,k)
                            # Wait, the voxel index along dominant axis is determined by the depth?
                            # The source says "output the corresponding voxel".
                            # In the center test, we output the voxel whose center depth is closest.
                            # In the edge test, if the depths straddle the voxel center depth, we output the voxel.
                            # Which voxel? The one at the current column (i,j) and the dominant axis index corresponding to d_voxel_center?
                            # Yes.
                            k = round((d_voxel_center - bb_min[0]) * resolution[0])
                            voxel_grid.set_item((i, j, k), True)
                        elif dominant_axis == 1:
                            k = round((d_voxel_center - bb_min[1]) * resolution[1])
                            voxel_grid.set_item((i, j, k), True)
                        else:
                            k = round((d_voxel_center - bb_min[2]) * resolution[2])
                            voxel_grid.set_item((i, j, k), True)
                else:
                    # Vertical Test
                    v_extents = get_vertical_extents(projected_vertices, u_center)
                    if v_extents is not None:
                        v_min, v_max = v_extents
                        # Clamp to pixel extents
                        v_min_clamped = max(v_min, j / resolution[axis2] + bb_min[axis2])
                        v_max_clamped = min(v_max, (j + 1) / resolution[axis2] + bb_min[axis2])
                        
                        d_top = get_depth(u_center, v_min_clamped)
                        d_bottom = get_depth(u_center, v_max_clamped)
                        
                        if dominant_axis == 0:
                            d_voxel_center = (i + 0.5) / resolution[0] + bb_min[0]
                        elif dominant_axis == 1:
                            d_voxel_center = (j + 0.5) / resolution[1] + bb_min[1]
                        else:
                            d_voxel_center = (j + 0.5) / resolution[2] + bb_min[2]
                            
                        if (d_top - d_voxel_center) * (d_bottom - d_voxel_center) < 0:
                            if dominant_axis == 0:
                                k = round((d_voxel_center - bb_min[0]) * resolution[0])
                                voxel_grid.set_item((i, j, k), True)
                            elif dominant_axis == 1:
                                k = round((d_voxel_center - bb_min[1]) * resolution[1])
                                voxel_grid.set_item((i, j, k), True)
                            else:
                                k = round((d_voxel_center - bb_min[2]) * resolution[2])
                                voxel_grid.set_item((i, j, k), True)

def is_point_in_triangle(p, tri):
    # Barycentric coordinate test
    x, y = p
    x0, y0 = tri[0]
    x1, y1 = tri[1]
    x2, y2 = tri[2]
    
    denom = (y1 - y2)*(x0 - x2) + (x2 - x1)*(y0 - y2)
    if denom == 0: return False
    
    l1 = ((y1 - y2)*(x - x2) + (x2 - x1)*(y - y2)) / denom
    l2 = ((y2 - y0)*(x - x2) + (x0 - x2)*(y - y2)) / denom
    l3 = 1.0 - l1 - l2
    
    return (l1 >= 0) and (l2 >= 0) and (l3 >= 0)

def get_horizontal_extents(tri, v):
    # Intersect triangle with line y = v
    # Returns (u_min, u_max) or None if no intersection
    intersections = []
    for i in range(3):
        p1 = tri[i]
        p2 = tri[(i+1)%3]
        if (p1[1] - v) * (p2[1] - v) <= 0:
            if p1[1] == p2[1]:
                continue # Horizontal edge, handled by other segments or ignored
            u = p1[0] + (v - p1[1]) * (p2[0] - p1[0]) / (p2[1] - p1[1])
            intersections.append(u)
    
    if len(intersections) < 2:
        return None
    return (min(intersections), max(intersections))

def get_vertical_extents(tri, u):
    # Intersect triangle with line x = u
    intersections = []
    for i in range(3):
        p1 = tri[i]
        p2 = tri[(i+1)%3]
        if (p1[0] - u) * (p2[0] - u) <= 0:
            if p1[0] == p2[0]:
                continue
            v = p1[1] + (u - p1[0]) * (p2[1] - p1[1]) / (p2[0] - p1[0])
            intersections.append(v)
            
    if len(intersections) < 2:
        return None
    return (min(intersections), max(intersections))
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$ where $N$ is the number of triangles and $P$ is the number of pixels in the projected bounding box. This is significantly faster than 3-axis methods which require $3 \times P$ processing.
*   **Memory Complexity:** $O(V)$ where $V$ is the number of voxels in the grid.
*   **Strengths:** Reduced computational cost due to single projection; maintains 6-separating (tunnel-free) property [S1, unofficial].
*   **Limitations:** Requires careful handling of edge cases where the triangle is parallel to the projection plane (normal has zero component in dominant axis, though this is rare for planar triangles unless degenerate). Performance depends on the distribution of triangle orientations.
*   **When to use:** When computational resources are limited and 6-connectivity is required. When triangles are not aligned with the principal axes, making multi-view projection redundant for many pixels.

## Background
This method was introduced by Laine et al. in "A Topological Approach to Voxelization" (EGSR 2013) [S1, unofficial]. It leverages the "thinness" property of voxelizations, which states that the voxel set produced by the crosshairs target is minimal in a topological sense. The method was developed to optimize the voxelization process by reducing the number of required projections while preserving topological correctness (6-separating). It is used in applications requiring efficient voxelization of triangle meshes, such as volume rendering and collision detection.

## Sources
[S1] A Topological Approach to Voxelization, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
