# Conservative Voxelization

## Overview
Conservative voxelization is a hardware-accelerated algorithm for polygonal models that guarantees conservative correctness: every voxel intersecting the input model is correctly recognized [S1, unofficial]. It differs from standard rasterization-based voxelization [Rasterization-based Voxelization] by computing the exact depth range of the intersection between the triangle and the voxel column, rather than using a single depth value which leads to missed voxels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A triangular model (tessellated from arbitrary polygons), a projection direction (typically orthogonal), and a volumetric resolution along the projection direction, $res_z$ [S1, unofficial].
- **Output:** A set of voxels $(x, y, z^*)$ corresponding to all voxels intersected by the triangles [S1, unofficial].

**Data Structures:**
- **Triangle Vertices:** Three vertices $P_1, P_2, P_3$ with 3D coordinates $(x_i, y_i, z_i)$ [S1, unofficial].
- **Projected Triangle ($T_p$):** The 2D projection of $T$ onto the viewing plane with vertices $(x_i, y_i)$ [S1, unofficial].
- **Fragment Rectangle ($R$):** The unit square in the viewing plane corresponding to a fragment $(x, y)$ [S1, unofficial].
- **Intersection Polygon ($I$):** The convex polygon resulting from the intersection of $T_p$ and $R$, with up to seven vertices [S1, unofficial].

**Numbered Steps:**
1. **Tessellation:** Tessellate arbitrary polygonal models into triangular models [S1, unofficial].
2. **Rasterization:** Rasterize the projected triangle $T_p$ to generate fragments, where each fragment $(x, y)$ corresponds to a column of voxels $V(x, y) = \{x, y, z | z = 0, ..., z_{res} - 1\}$ [S1, unofficial].
3. **Classification:** For each fragment, determine if it is an **interior fragment** (fully covered by $T_p$) or a **boundary fragment** (partially covered by $T_p$) [S1, unofficial].
4. **Depth Range Calculation:** Compute the minimal depth $z_{min}$ and maximal depth $z_{max}$ of the intersection between the triangle $T$ and the prism formed by $V(x, y)$ [S1, unofficial].
    - **Interior Fragments:** Use the center depth and partial derivatives [S1, unofficial].
    - **Boundary Fragments:** Compute the intersection polygon $I$ and evaluate depths at its vertices [S1, unofficial].
5. **Voxel Generation:** Convert the depth range $[z_{min}, z_{max}]$ to voxel indices: $z^*_{min} = z_{min} \cdot res_z$ and $z^*_{max} = z_{max} \cdot res_z$ [S1, unofficial]. Generate all voxels $(x, y, z)$ for $z \in [z^*_{min}, z^*_{max}]$ [S1, unofficial].

**Formulas:**
**1. Interior Fragment Depth Range:**
For an interior fragment, the depth range is calculated as:
$$ z_{min} = z_c - \Delta z $$
$$ z_{max} = z_c + \Delta z $$
where $z_c$ is the depth value at the fragment center, and:
$$ \Delta z = \frac{1}{2} \left( \left| \frac{\partial z}{\partial x} \right| + \left| \frac{\partial z}{\partial y} \right| \right) $$
[S1, unofficial]

**2. Partial Derivatives:**
The partial derivatives of depth and barycentric coordinates are computed using the triangle vertices $(x_i, y_i, z_i)$ for $i=1,2,3$:

Signed area of $T_p$:
$$ S = x_1 y_2 - x_2 y_1 + x_2 y_3 - x_3 y_2 + x_3 y_1 - x_1 y_3 $$
[S1, unofficial]

Partial derivative of barycentric coordinate vector $\mathbf{C}$ with respect to $x$:
$$ \frac{\partial \mathbf{C}}{\partial x} = \frac{(y_2 - y_3, y_3 - y_1, y_1 - y_2)}{S} $$
[S1, unofficial]

Partial derivative of barycentric coordinate vector $\mathbf{C}$ with respect to $y$:
$$ \frac{\partial \mathbf{C}}{\partial y} = \frac{(x_3 - x_2, x_1 - x_3, x_2 - x_1)}{S} $$
[S1, unofficial]

Partial derivative of depth $z$ with respect to $x$:
$$ \frac{\partial z}{\partial x} = (z_1, z_2, z_3) \cdot \frac{\partial \mathbf{C}}{\partial x} $$
[S1, unofficial]

Partial derivative of depth $z$ with respect to $y$:
$$ \frac{\partial z}{\partial y} = (z_1, z_2, z_3) \cdot \frac{\partial \mathbf{C}}{\partial y} $$
[S1, unofficial]

**3. Boundary Fragment Depth Range:**
The intersection polygon $I$ vertices consist of:
- Intersection points between $T_p$ and the edges of $R$ [S1, unofficial].
- Vertices of $T_p$ inside $R$ [S1, unofficial].
- Vertices of $R$ inside $T_p$ [S1, unofficial].

Let $v_{min}$ be the vertex of $R$ such that $z(v_{min}) = \min_{p \in R} z(p)$ [S1, unofficial].
If $v_{min} \in T_p$, then $z_{min} = z(v_{min})$ [S1, unofficial].
Otherwise, initialize $z_{min} = \text{HUGE\_VALUE}$ and update:
$$ z_{min} = \min \left( z_{min}, z(Q_i) \right) \quad \text{for each intersection point } Q_i $$
$$ z_{min} = \min \left( z_{min}, z(P_i) \right) \quad \text{for each vertex } P_i \text{ of } T \text{ inside } R $$
$z_{max}$ is computed similarly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def conservative_voxelization(triangles, res_z):
    """
    triangles: List of triangles, each triangle is a list of 3 vertices.
               Each vertex is a tuple (x, y, z).
    res_z: Volumetric resolution along the projection direction.
    Returns: List of voxels (x, y, z_idx).
    """
    voxels = []
    
    for tri in triangles:
        p1, p2, p3 = tri
        x1, y1, z1 = p1
        x2, y2, z2 = p2
        x3, y3, z3 = p3
        
        S = x1*y2 - x2*y1 + x2*y3 - x3*y2 + x3*y1 - x1*y3
        if S == 0:
            continue
            
        dC_dx = ((y2 - y3)/S, (y3 - y1)/S, (y1 - y2)/S)
        dC_dy = ((x3 - x2)/S, (x1 - x3)/S, (x2 - x1)/S)
        
        dz_dx = z1*dC_dx[0] + z2*dC_dx[1] + z3*dC_dx[2]
        dz_dy = z1*dC_dy[0] + z2*dC_dy[1] + z3*dC_dy[2]
        
        delta_z = 0.5 * (abs(dz_dx) + abs(dz_dy))
        
        # Hardware/software rasterization generates fragments covering the triangle
        fragments = rasterize_triangle((x1, y1), (x2, y2), (x3, y3), (z1, z2, z3))
        
        for frag in fragments:
            x, y, z_c, is_interior, poly_verts = frag
            
            if is_interior:
                z_min = z_c - delta_z
                z_max = z_c + delta_z
            else:
                # Find v_min: vertex of fragment rectangle R with minimum depth
                r_corners = [(x, y), (x+1, y), (x, y+1), (x+1, y+1)]
                v_min_depth = float('inf')
                v_min_pt = None
                for cx, cy in r_corners:
                    d = get_depth_at_point(cx, cy, (x1,y1,z1), (x2,y2,z2), (x3,y3,z3), S)
                    if d < v_min_depth:
                        v_min_depth = d
                        v_min_pt = (cx, cy)
                
                if point_in_triangle(v_min_pt[0], v_min_pt[1], (x1,y1), (x2,y2), (x3,y3), S):
                    z_min = v_min_depth
                else:
                    z_min = float('inf')
                    z_max = float('-inf')
                    
                    for q in poly_verts:
                        d = get_depth_at_point(q[0], q[1], (x1,y1,z1), (x2,y2,z2), (x3,y3,z3), S)
                        z_min = min(z_min, d)
                        z_max = max(z_max, d)
                        
                    for p in [(x1,y1,z1), (x2,y2,z2), (x3,y3,z3)]:
                        if x <= p[0] <= x+1 and y <= p[1] <= y+1:
                            z_min = min(z_min, p[2])
                            z_max = max(z_max, p[2])
                            
                if z_min == float('inf'):
                    z_min = z_c - delta_z
                    z_max = z_c + delta_z

            z_idx_min = int(math.floor(z_min * res_z))
            z_idx_max = int(math.ceil(z_max * res_z))
            
            for z_idx in range(z_idx_min, z_idx_max + 1):
                voxels.append((x, y, z_idx))
                
    return voxels

def get_depth_at_point(px, py, p1, p2, p3, S):
    x1, y1, z1 = p1
    x2, y2, z2 = p2
    x3, y3, z3 = p3
    lambda1 = ((y2 - y3)*(px - x3) + (x3 - x2)*(py - y3)) / S
    lambda2 = ((y3 - y1)*(px - x3) + (x1 - x3)*(py - y3)) / S
    lambda3 = 1.0 - lambda1 - lambda2
    return lambda1*z1 + lambda2*z2 + lambda3*z3

def point_in_triangle(px, py, p1, p2, p3, S):
    x1, y1 = p1
    x2, y2 = p2
    x3, y3 = p3
    lambda1 = ((y2 - y3)*(px - x3) + (x3 - x2)*(py - y3)) / S
    lambda2 = ((y3 - y1)*(px - x3) + (x1 - x3)*(py - y3)) / S
    lambda3 = 1.0 - lambda1 - lambda2
    return lambda1 >= 0 and lambda2 >= 0 and lambda3 >= 0

def rasterize_triangle(v2d_1, v2d_2, v2d_3, v3d_1, v3d_2, v3d_3):
    """Simulates hardware rasterization by scanning the bounding box."""
    xs = [v2d_1[0], v2d_2[0], v2d_3[0]]
    ys = [v2d_1[1], v2d_2[1], v2d_3[1]]
    min_x, max_x = int(math.floor(min(xs))), int(math.ceil(max(xs)))
    min_y, max_y = int(math.floor(min(ys))), int(math.ceil(max(ys)))
    fragments = []
    for x in range(min_x, max_x + 1):
        for y in range(min_y, max_y + 1):
            cx, cy = x + 0.5, y + 0.5
            if point_in_triangle(cx, cy, v2d_1, v2d_2, v2d_3, 
                                 (v2d_1[0]*v2d_2[1] - v2d_2[0]*v2d_1[1] + v2d_2[0]*v2d_3[1] - v2d_3[0]*v2d_2[1] + v2d_3[0]*v2d_1[1] - v2d_1[0]*v2d_3[1])):
                is_interior = point_in_triangle(x+0.25, y+0.25, v2d_1, v2d_2, v2d_3, 
                                                (v2d_1[0]*v2d_2[1] - v2d_2[0]*v2d_1[1] + v2d_2[0]*v2d_3[1] - v2d_3[0]*v2d_2[1] + v2d_3[0]*v2d_1[1] - v2d_1[0]*v2d_3[1])) and \
                              point_in_triangle(x+0.75, y+0.75, v2d_1, v2d_2, v2d_3, 
                                                (v2d_1[0]*v2d_2[1] - v2d_2[0]*v2d_1[1] + v2d_2[0]*v2d_3[1] - v2d_3[0]*v2d_2[1] + v2d_3[0]*v2d_1[1] - v2d_1[0]*v2d_3[1]))
                z_c = get_depth_at_point(cx, cy, v3d_1, v3d_2, v3d_3, 
                                         (v2d_1[0]*v2d_2[1] - v2d_2[0]*v2d_1[1] + v2d_2[0]*v2d_3[1] - v2d_3[0]*v2d_2[1] + v2d_3[0]*v2d_1[1] - v2d_1[0]*v2d_3[1]))
                poly_verts = []
                if not is_interior:
                    # Compute intersection polygon vertices for boundary fragments
                    edges = [(v2d_1, v2d_2), (v2d_2, v2d_3), (v2d_3, v2d_1)]
                    r_edges = [((x,y),(x+1,y)), ((x+1,y),(x+1,y+1)), ((x+1,y+1),(x,y+1)), ((x,y+1),(x,y))]
                    for e1, e2 in edges:
                        for r1, r2 in r_edges:
                            inter = line_segment_intersection(e1, e2, r1, r2)
                            if inter:
                                poly_verts.append(inter)
                    for v in [v3d_1, v3d_2, v3d_3]:
                        if x <= v[0] <= x+1 and y <= v[1] <= y+1:
                            poly_verts.append((v[0], v[1]))
                    for rv in [(x,y),(x+1,y),(x,y+1),(x+1,y+1)]:
                        if point_in_triangle(rv[0], rv[1], v2d_1, v2d_2, v2d_3, 
                                             (v2d_1[0]*v2d_2[1] - v2d_2[0]*v2d_1[1] + v2d_2[0]*v2d_3[1] - v2d_3[0]*v2d_2[1] + v2d_3[0]*v2d_1[1] - v2d_1[0]*v2d_3[1])):
                            poly_verts.append(rv)
                fragments.append((x, y, z_c, is_interior, poly_verts))
    return fragments

def line_segment_intersection(p1, p2, p3, p4):
    d = (p2[0]-p1[0])*(p4[1]-p3[1]) - (p2[1]-p1[1])*(p4[0]-p3[0])
    if d == 0: return None
    t = ((p4[0]-p3[0])*(p3[1]-p1[1]) - (p4[1]-p3[1])*(p3[0]-p1[0])) / d
    u = ((p2[0]-p1[0])*(p3[1]-p1[1]) - (p2[1]-p1[1])*(p3[0]-p1[0])) / d
    if 0 <= t <= 1 and 0 <= u <= 1:
        return (p1[0] + t*(p2[0]-p1[0]), p1[1] + t*(p2[1]-p1[1]))
    return None
```

**Parameters:**
- $res_z$: The volumetric resolution along the projection direction [S1, unofficial].

**Edge Cases and Pitfalls:**
- Using the simple interior fragment depth range formula for boundary fragments leads to over-conservative results, especially when the voxelizing triangle is nearly parallel to the projection direction [S1, unofficial].
- Degenerate triangles (area $S=0$) must be handled to avoid division by zero [G].

## Operating characteristics
- **Measured Performance:** Experiments show lower memory consumption than previous approaches and higher efficiency when the volume resolution is high [S1, unofficial].
- **Strengths:** Guarantees conservative correctness; no preprocessing required; suitable for deformable models; hardware-accelerated [S1, unofficial].
- **Limitations:** More complex implementation than simple rasterization-based methods; potential for over-conservative results if boundary handling is incorrect [S1, unofficial].
- **When to Use:** Collision detection, occlusion culling, visibility processing, and applications requiring guaranteed intersection detection [S1, unofficial].
- **When Not to Use:** When performance is the sole priority and missing some intersecting voxels is acceptable; when input is not polygonal [S1, unofficial].

## Background
The algorithm was proposed by researchers from Zhejiang University and Purdue University, as indicated by the affiliations in the source [S1, unofficial]. It addresses the limitations of previous hardware-accelerated voxelization algorithms that missed intersecting voxels, which is critical for applications like collision detection and occlusion culling [S1, unofficial]. The method leverages GPU rasterization capabilities to achieve efficiency [S1, unofficial].

## Sources
[S1] Conservative voxelization, https://wo1fsea.github.io/files/About_NavMesh/P1_Conservative_voxelization.pdf, fetched 2026-10-27 (unofficial)
