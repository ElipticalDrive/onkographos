# Conservative Voxelization


## Overview
Conservative voxelization is a hardware-accelerated algorithm for converting polygonal models into a volumetric representation that guarantees **conservative correctness**: every voxel intersecting the input model is correctly recognized, solving the problem where previous approaches miss intersecting voxels [S1, unofficial]. It sets itself apart from standard rasterization-based voxelization by computing the exact depth range of a triangle within a fragment's column, rather than sampling a single depth value at the fragment center [S1, unofficial]. This ensures no intersecting voxels are omitted, which is critical for applications like collision detection and occlusion culling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh (arbitrary polygonal models must be tessellated first), a projection direction (typically orthogonal), and a volumetric resolution along the projection direction, denoted as $res_z$ [S1, unofficial].
*   **Output:** A set of voxels $(x, y, z^*)$ representing the intersection of the triangle with the voxel grid [S1, unofficial].

**Data Structures:**
*   **Fragment Rectangle ($R$):** The 2D area on the viewing plane corresponding to a single voxel column $(x, y)$.
*   **Projected Triangle ($T_p$):** The 2D projection of the 3D triangle $T$ onto the viewing plane.
*   **Intersection Polygon ($I$):** The convex polygon resulting from the intersection of $R$ and $T_p$. It has up to seven vertices [S1, unofficial].

**Numbered Steps:**

1.  **Tessellation:** If the input model is not triangular, tessellate it into triangular models [S1, unofficial].
2.  **Rasterization:** Rasterize the projected triangle $T_p$ to generate fragments. Each fragment $(x, y)$ corresponds to a column of voxels $V(x, y)$ [S1, unofficial].
3.  **Classification:** Determine if the fragment rectangle $R$ is an **interior fragment** (fully covered by $T_p$) or a **boundary fragment** (partially covered by $T_p$) [S1, unofficial].
4.  **Depth Range Calculation:** Compute the minimal depth $z_{min}$ and maximal depth $z_{max}$ of the triangle within the fragment rectangle $R$.
    *   **For Interior Fragments:**
        Calculate the depth range using the center depth and partial derivatives [S1, unofficial]:
        $$z_{min} = z_c - \Delta z$$
        $$z_{max} = z_c + \Delta z$$
        $$\Delta z = \frac{1}{2} \left( \left| \frac{\partial z}{\partial x} \right| + \left| \frac{\partial z}{\partial y} \right| \right)$$
        Where $z_c$ is the depth value at the fragment center, and $\frac{\partial z}{\partial x}, \frac{\partial z}{\partial y}$ are the partial derivatives of the depth value [S1, unofficial].
    *   **For Boundary Fragments:**
        Compute the intersection polygon $I$ between $R$ and $T_p$. The vertices of $I$ consist of:
        1.  Intersection points between the triangle edges and the fragment rectangle edges.
        2.  Vertices of the triangle $T$ that lie inside $R$.
        3.  Vertices of the fragment rectangle $R$ that lie inside $T_p$ [S1, unofficial].
        
        Identify $v_{min}$, the vertex of $R$ that satisfies $z(v_{min}) = \min_{p \in R} z(p)$ [S1, unofficial].
        
        Compute $z_{min}$:
        *   If $v_{min} \in T_p$ (i.e., the minimum depth vertex of the rectangle is inside the triangle), then $z_{min} = z(v_{min})$ [S1, unofficial].
        *   Otherwise, initialize $z_{min} = \text{HUGE\_VALUE}$. Iterate through all intersection points $Q_i$ and all triangle vertices $P_i$ inside $R$. Update $z_{min} = \min(z_{min}, z(Q_i))$ and $z_{min} = \min(z_{min}, z(P_i))$ [S1, unofficial].
        
        Compute $z_{max}$ in the same manner, identifying the vertex with maximum depth [S1, unofficial].
5.  **Voxel Generation:** Convert the continuous depth range $[z_{min}, z_{max}]$ to voxel indices:
    $$z^*_{min} = z_{min} \cdot res_z$$
    $$z^*_{max} = z_{max} \cdot res_z$$
    Generate voxels $(x, y, z^*)$ for all integer $z^*$ such that $z^*_{min} \le z^* \le z^*_{max}$ [S1, unofficial].

**Formulas and Definitions:**

*   **Signed Area of $T_p$ ($S$):**
    $$S = x_1 y_2 - x_2 y_1 + x_2 y_3 - x_3 y_2 + x_3 y_1 - x_1 y_3$$
    Where $(x_i, y_i)$ are the 2D coordinates of the vertices of $T_p$ [S1, unofficial].

*   **Partial Derivative of Barycentric Coordinate ($\mathbf{C}$) with respect to $x$:**
    $$\frac{\partial \mathbf{C}}{\partial x} = \frac{(y_2 - y_3, y_3 - y_1, y_1 - y_2)}{S}$$
    Where $\mathbf{C}$ is the barycentric coordinate vector of the fragment center [S1, unofficial].

*   **Partial Derivative of Barycentric Coordinate ($\mathbf{C}$) with respect to $y$:**
    $$\frac{\partial \mathbf{C}}{\partial y} = \frac{(x_3 - x_2, x_1 - x_3, x_2 - x_1)}{S}$$
    [S1, unofficial]

*   **Partial Derivative of Depth ($z$) with respect to $x$:**
    $$\frac{\partial z}{\partial x} = (z_1, z_2, z_3) \cdot \frac{\partial \mathbf{C}}{\partial x}$$
    Where $z_i$ are the depth values of the triangle vertices [S1, unofficial].

*   **Partial Derivative of Depth ($z$) with respect to $y$:**
    $$\frac{\partial z}{\partial y} = (z_1, z_2, z_3) \cdot \frac{\partial \mathbf{C}}{\partial y}$$
    [S1, unofficial]

*   **Barycentric Coordinate Difference ($\Delta \mathbf{C}$):**
    $$\Delta \mathbf{C} = \frac{1}{2} \cdot \text{sign}\left(\frac{\partial z}{\partial x}\right) \cdot \frac{\partial \mathbf{C}}{\partial x} + \frac{1}{2} \cdot \text{sign}\left(\frac{\partial z}{\partial y}\right) \cdot \frac{\partial \mathbf{C}}{\partial y}$$
    [S1, unofficial]

**Parameters:**
*   $res_z$: The volumetric resolution along the projection direction [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Over-conservative Results:** Using the simple interior fragment depth range formula for boundary fragments leads to over-conservative results, especially when the voxelizing triangle is nearly parallel to the projection direction [S1, unofficial].
*   **Degenerate Triangles:** The signed area $S$ must be non-zero. If $S=0$, the triangle is degenerate in projection and should be skipped or handled separately [G].
*   **Precision:** Computing intersection points $Q_i$ requires robust geometric predicates to avoid gaps or overlaps in the voxel grid [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def conservative_voxelization(triangle, res_z):
    """
    triangle: List of 3 vertices, each with (x, y, z) in 3D space.
    res_z: Volumetric resolution along the projection direction (e.g., depth axis).
    Returns: List of voxel indices (x, y, z_idx).
    """
    # 1. Project triangle to 2D (assuming orthogonal projection along Z)
    # For this implementation, we assume the projection plane is XY and depth is Z.
    # Vertices are (x, y, z). Projected vertices are (x, y).
    v0, v1, v2 = triangle
    p0 = (v0.x, v0.y)
    p1 = (v1.x, v1.y)
    p2 = (v2.x, v2.y)
    
    # 2. Compute Signed Area S
    S = (p0[0]*p1[1] - p1[0]*p0[1] + 
         p1[0]*p2[1] - p2[0]*p1[1] + 
         p2[0]*p0[1] - p0[0]*p2[1])
    
    if S == 0:
        return [] # Degenerate triangle

    # 3. Compute Partial Derivatives of Barycentric Coordinates
    # dC/dx = (y2-y3, y3-y1, y1-y2) / S
    dC_dx = ((p1[1] - p2[1]), (p2[1] - p0[1]), (p0[1] - p1[1])) / S
    # dC/dy = (x3-x2, x1-x3, x2-x1) / S
    dC_dy = ((p2[0] - p1[0]), (p0[0] - p2[0]), (p1[0] - p0[0])) / S
    
    # 4. Compute Partial Derivatives of Depth
    # z_vals = (z0, z1, z2)
    z_vals = (v0.z, v1.z, v2.z)
    
    dz_dx = dot(z_vals, dC_dx)
    dz_dy = dot(z_vals, dC_dy)
    
    # 5. Determine Bounding Box of Projected Triangle in Fragment Space
    # Assume fragments are unit squares. We need to rasterize.
    # For simplicity, we iterate over fragments covered by the triangle's bounding box.
    # In a real GPU implementation, this is done via rasterization hardware.
    # Here we simulate the logic for a single fragment rectangle R defined by [fx, fx+1] x [fy, fy+1].
    
    voxels = []
    
    # Get bounding box of triangle in integer fragment coordinates
    min_x = floor(min(p0[0], p1[0], p2[0]))
    max_x = ceil(max(p0[0], p1[0], p2[0]))
    min_y = floor(min(p0[1], p1[1], p2[1]))
    max_y = ceil(max(p0[1], p1[1], p2[1]))
    
    for fx in range(min_x, max_x):
        for fy in range(min_y, max_y):
            # Define Fragment Rectangle R: [fx, fx+1] x [fy, fy+1]
            R = {
                'min': (fx, fy),
                'max': (fx + 1, fy + 1)
            }
            
            # Check if triangle intersects this fragment
            if not triangle_intersects_rect(p0, p1, p2, R):
                continue
                
            # Compute Center Depth and check if interior
            center_x = fx + 0.5
            center_y = fy + 0.5
            
            # Compute barycentric coords at center to check if inside
            # C = (C0, C1, C2)
            # C0 = ((y1-y2)*(x-x2) + (x2-x1)*(y-y2)) / (-S) ... wait, S definition varies.
            # Using the derivative approach for interior check is complex.
            # Let's use standard barycentric test.
            C = compute_barycentric(center_x, center_y, p0, p1, p2, S)
            
            is_interior = (C[0] >= 0 and C[1] >= 0 and C[2] >= 0)
            
            if is_interior:
                # Interior Fragment
                z_c = dot(z_vals, C)
                delta_z = 0.5 * (abs(dz_dx) + abs(dz_dy))
                z_min = z_c - delta_z
                z_max = z_c + delta_z
            else:
                # Boundary Fragment
                # Compute Intersection Polygon I
                I = compute_intersection_polygon(p0, p1, p2, R)
                
                if not I:
                    continue
                    
                # Find vmin: vertex of R with min depth
                R_vertices = [
                    (fx, fy),
                    (fx + 1, fy),
                    (fx + 1, fy + 1),
                    (fx, fy + 1)
                ]
                
                # Compute depth for each R vertex
                R_depths = []
                for rv in R_vertices:
                    C_rv = compute_barycentric(rv[0], rv[1], p0, p1, p2, S)
                    # Only consider if inside triangle for vmin candidate? 
                    # No, vmin is min depth vertex OF R, regardless of position.
                    # But we need to know if vmin is inside Tp.
                    z_rv = dot(z_vals, C_rv)
                    R_depths.append((rv, z_rv, C_rv))
                
                # Find vertex with min depth
                vmin_entry = min(R_depths, key=lambda k: k[1])
                vmin = vmin_entry[0]
                z_vmin = vmin_entry[1]
                C_vmin = vmin_entry[2]
                
                # Check if vmin is inside Tp
                if C_vmin[0] >= 0 and C_vmin[1] >= 0 and C_vmin[2] >= 0:
                    z_min = z_vmin
                else:
                    z_min = float('inf')
                    
                    # Check intersection points Qi
                    for qi in I:
                        C_qi = compute_barycentric(qi[0], qi[1], p0, p1, p2, S)
                        z_qi = dot(z_vals, C_qi)
                        if z_qi < z_min:
                            z_min = z_qi
                            
                    # Check triangle vertices Pi inside R
                    for pi_idx, pi in enumerate([p0, p1, p2]):
                        if point_in_rect(pi, R):
                            z_pi = [v0.z, v1.z, v2.z][pi_idx]
                            if z_pi < z_min:
                                z_min = z_pi
                                
                # Compute z_max similarly
                vmax_entry = max(R_depths, key=lambda k: k[1])
                vmax = vmax_entry[0]
                z_vmax = vmax_entry[1]
                C_vmax = vmax_entry[2]
                
                if C_vmax[0] >= 0 and C_vmax[1] >= 0 and C_vmax[2] >= 0:
                    z_max = z_vmax
                else:
                    z_max = float('-inf')
                    
                    for qi in I:
                        C_qi = compute_barycentric(qi[0], qi[1], p0, p1, p2, S)
                        z_qi = dot(z_vals, C_qi)
                        if z_qi > z_max:
                            z_max = z_qi
                            
                    for pi_idx, pi in enumerate([p0, p1, p2]):
                        if point_in_rect(pi, R):
                            z_pi = [v0.z, v1.z, v2.z][pi_idx]
                            if z_pi > z_max:
                                z_max = z_pi

            # Generate Voxels
            z_idx_min = floor(z_min * res_z)
            z_idx_max = ceil(z_max * res_z)
            
            for z_idx in range(z_idx_min, z_idx_max + 1):
                voxels.append((fx, fy, z_idx))
                
    return voxels

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def compute_barycentric(x, y, p0, p1, p2, S):
    # Barycentric coordinates for point (x,y) relative to triangle p0,p1,p2
    # Using area method
    # C0 = Area(p, p1, p2) / Area(p0, p1, p2)
    # C1 = Area(p0, p, p2) / Area(p0, p1, p2)
    # C2 = Area(p0, p1, p) / Area(p0, p1, p2)
    
    def area(p1, p2, p3):
        return (p1[0]*(p2[1]-p3[1]) + p2[0]*(p3[1]-p1[1]) + p3[0]*(p1[1]-p2[1])) / 2.0
        
    A = area(p0, p1, p2)
    if A == 0: return (0,0,0)
    
    C0 = area((x,y), p1, p2) / A
    C1 = area(p0, (x,y), p2) / A
    C2 = area(p0, p1, (x,y)) / A
    
    return (C0, C1, C2)

def triangle_intersects_rect(p0, p1, p2, R):
    # Simplified check: bounding box overlap
    min_x = min(p0[0], p1[0], p2[0])
    max_x = max(p0[0], p1[0], p2[0])
    min_y = min(p0[1], p1[1], p2[1])
    max_y = max(p0[1], p1[1], p2[1])
    
    return not (max_x < R['min'][0] or min_x > R['max'][0] or 
                max_y < R['min'][1] or min_y > R['max'][1])

def compute_intersection_polygon(p0, p1, p2, R):
    # Clip triangle against rectangle R
    # Return list of points
    # This is a standard Sutherland-Hodgman or similar clip
    # For brevity, assume a helper exists
    pass

def point_in_rect(p, R):
    return R['min'][0] <= p[0] <= R['max'][0] and R['min'][1] <= p[1] <= R['max'][1]
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of fragments generated by rasterization. The boundary fragment calculation involves computing intersection polygons and checking vertices, which is constant time per fragment (up to 7 vertices) [S1, unofficial].
*   **Memory Complexity:** Low. It requires no preprocessing and stores only the voxel data structure [S1, unofficial].
*   **Measured Performance:** Experiments show lower memory consumption than previous approaches and higher efficiency when the volume resolution is high [S1, unofficial].
*   **Strengths:** Guarantees conservative correctness; no preprocessing required; suitable for deformable models; hardware-accelerated [S1, unofficial].
*   **Limitations:** More complex fragment shader logic for boundary fragments compared to simple center-depth sampling [S1, unofficial].
*   **When to Use:** Collision detection, occlusion culling, visibility processing, and any application where missing intersecting voxels is unacceptable [S1, unofficial].
*   **When Not to Use:** When performance is critical and approximate voxelization is acceptable, or when the triangle is nearly parallel to the projection direction and simple interior formulas are mistakenly used for boundary fragments [S1, unofficial].

## Background
Conservative voxelization was proposed by researchers including David S. Ebert from Purdue University and authors from Zhejiang University [S1, unofficial]. The algorithm addresses the limitations of previous hardware-accelerated voxelization techniques that missed intersecting voxels by using only the center depth value [S1, unofficial]. It builds on the concept of conservative rasterization [S1, unofficial].

## Sources
[S1] Conservative voxelization, https://wo1fsea.github.io/files/About_NavMesh/P1_Conservative_voxelization.pdf, fetched 2026-10-27 (unofficial)
