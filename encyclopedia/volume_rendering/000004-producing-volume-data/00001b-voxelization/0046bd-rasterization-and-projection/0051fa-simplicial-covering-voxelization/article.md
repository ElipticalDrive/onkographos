# Simplicial Covering Voxelization

Also called: P-Buffer method, Solid Voxelization via Simplicial Covering, Voxelization of solids using simplicial coverings

## Overview
Simplicial Covering Voxelization is a method to rasterize and voxelize 3D solids by decomposing their surface into tetrahedra connected to an origin, then using a presence buffer (P-buffer) to toggle voxel states [S1, unofficial]. Unlike boundary-only methods such as [Voxel Carving](1213-voxel-carving.md) or [Sramek's method](1213-voxel-carving.md), this approach produces a complete solid voxelization in a single pass by exploiting the parity of tetrahedron coverage [S1, unofficial]. It differs from [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md) by operating directly on polygonal meshes without requiring implicit field evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D solid defined by a set of planar faces (manifold or non-manifold, with or without holes), represented as a set of triangles. A coordinate origin $O$ (typically the centroid or arbitrary point).
- **Output:** A 3D voxel grid (P-buffer) where each voxel stores a presence bit (0 or 1) indicating if it is inside the solid. Optionally, a framebuffer with colors applied.

**Data Structures:**
- **P-buffer:** A 3D array of bits. Each position $(x_i, y_i, z_i)$ stores a single bit: 0 (outside) or 1 (inside) [S1, unofficial].
- **Vertices:** Points $A, B, C$ defining the base triangle of a tetrahedron.
- **Slopes:** Incremental slope pairs $m_{XY} = (m_{XY}^x, m_{XY}^z)$ for edges, defined as $\frac{X_x - Y_x}{X_y - Y_y}$ and $\frac{X_z - Y_z}{X_y - Y_y}$ [S1, unofficial].

**Algorithm Steps:**

1.  **Tetrahedron Construction:** For every triangle of the covering of each face of the solid, construct a tetrahedron by joining the triangle vertices ($A, B, C$) with the origin $O$ [S1, unofficial].

2.  **Sweeping Plane Determination:** Determine the sweeping direction by finding the maximum absolute value of the coordinates of the normal vector of the base triangle. The method assumes rasterization with respect to the Y-direction. If the maximum is negative, invert the tetrahedron with respect to the Y coordinate; at the end, invert the resulting voxel coordinates $(x_i, y_i, z_i)$ to $(x_i, -y_i, z_i)$ [S1, unofficial].

3.  **Initialization:**
    - Initialize the voxel matrix (P-buffer) to 0 [S1, unofficial].
    - Sort vertices $A, B, C$ by their Y-coordinate. Let $A$ be the vertex with the maximum Y.
    - Set current $y = A_y$.
    - Initialize triangle vertices for the current slice: $A_i = A$, $B_i = B$, $C_i = C$.
    - Compute initial slopes: $m_B = m_{AB}$, $m_C = m_{AC}$ [S1, unofficial].

4.  **Iterative Rasterization (Sweep):**
    While $y > 0$:
    - **Update Edge Vertices:**
        - If $B_y = y$, set $B_i = B$ and $m_B = m_{OB}$. Else, set $m_B = m_{AB}$ and $B_i = A_i + (B - A) \cdot m_B$.
        - If $C_y = y$, set $C_i = C$ and $m_C = m_{OC}$. Else, set $m_C = m_{AC}$ and $C_i = A_i + (C - A) \cdot m_C$.
    - **Rasterize Primary Triangle:** Call `Rasterize2D(A_i, B_i, C_i)` to flip presence values for voxels inside the triangle slice at current $y$ [S1, unofficial].
    - **Handle Quadrilateral Slice:** If $B_y > y > C_y$, the intersection is a quadrilateral. Compute the intersection point $D_i = B + (C - B) \cdot m_{BC}$. Call `Rasterize2D(B_i, D_i, C_i)` to handle the second triangle of the quadrilateral [S1, unofficial].
    - **Decrement and Update:**
        - Decrement $y$.
        - If $y < B_y$, update $m_B = m_{OB}$.
        - If $y < C_y$, update $m_C = m_{OC}$.
        - Update $A_i, B_i, C_i$ with the corresponding incremental slopes for the next $y$-slice [S1, unofficial].

5.  **Finalization:** Transfer all positions from the P-buffer with presence value equal to 1 to the framebuffer, applying a color function that returns the corresponding color of the solid at that point [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simplicial_covering_voxelization(solid_triangles, origin, grid_resolution):
    # Initialize P-buffer with 0s
    p_buffer = zeros(grid_resolution, dtype=bool)
    
    for triangle in solid_triangles:
        A, B, C = triangle.vertices
        
        # Step 2: Determine sweeping direction (assumed Y for this implementation)
        # If normal's max abs coord is negative, invert tetrahedron Y and final voxels
        # For simplicity, assume standard orientation or handle inversion post-process
        
        # Step 3: Sort vertices by Y
        vertices = sorted([A, B, C], key=lambda v: v.y, reverse=True)
        A, B, C = vertices[0], vertices[1], vertices[2]
        
        y = A.y
        Ai = A
        Bi = B
        Ci = C
        
        # Compute initial slopes
        # Slope m_XY = (X - Y) / (X.y - Y.y)
        m_AB = (B - A) / (B.y - A.y)
        m_AC = (C - A) / (C.y - A.y)
        m_BC = (C - B) / (C.y - B.y)
        m_OA = (A - origin) / (A.y - origin.y)
        m_OB = (B - origin) / (B.y - origin.y)
        m_OC = (C - origin) / (C.y - origin.y)
        
        mB = m_AB
        mC = m_AC
        
        # Step 4: Iterative Rasterization
        while y > 0:
            # Update Edge Vertices
            if B.y == y:
                Bi = B
                mB = m_OB
            else:
                mB = m_AB
                Bi = Ai + (B - A) * mB
            
            if C.y == y:
                Ci = C
                mC = m_OC
            else:
                mC = m_AC
                Ci = Ai + (C - A) * mC
            
            # Rasterize Primary Triangle
            rasterize_2d_triangle(p_buffer, Ai, Bi, Ci, y)
            
            # Handle Quadrilateral Slice
            if B.y > y > C.y:
                # Intersection point Di on edge BC
                Di = B + (C - B) * m_BC
                rasterize_2d_triangle(p_buffer, Bi, Di, Ci, y)
            
            # Decrement and Update
            y -= 1
            
            if y < B.y:
                mB = m_OB
            if y < C.y:
                mC = m_OC
            
            # Update Ai, Bi, Ci with incremental slopes for next slice
            # Note: Ai is always updated using m_OA relative to origin logic or fixed A?
            # The source implies Ai updates via m_OA if A is the top vertex connected to origin
            # However, standard scanline updates edges. 
            # Source says: "For the vertex A, the parameter mOA will always be used."
            # But Ai is the intersection of OA with plane y.
            # Incremental update: x_{i+1} = x_i + m_OA.x
            Ai = Ai + m_OA
            
            # Bi and Ci updates are handled by the slope checks above or incremental
            # The source algorithm updates Bi/Ci based on slope changes at vertices
            # For continuous update between vertices:
            Bi = Bi + mB
            Ci = Ci + mC

    return p_buffer

def rasterize_2d_triangle(p_buffer, v1, v2, v3, y):
    """
    Fills the 2D triangle in the x-z plane at current y.
    Flips the P-buffer value at (xi, yi, zi) for any pixel inside.
    """
    # Sort vertices by X for scanline fill
    verts = sorted([v1, v2, v3], key=lambda v: v.x)
    v1, v2, v3 = verts[0], verts[1], verts[2]
    
    # Compute slopes for edges
    if v1.x == v2.x:
        m12 = 0
    else:
        m12 = (v2.z - v1.z) / (v2.x - v1.x)
        
    if v2.x == v3.x:
        m23 = 0
    else:
        m23 = (v3.z - v2.z) / (v3.x - v2.x)
        
    if v1.x == v3.x:
        m13 = 0
    else:
        m13 = (v3.z - v1.z) / (v3.x - v1.x)
        
    # Scanline fill
    x = v1.x
    z_left = v1.z
    z_right = v1.z
    
    # Upper triangle (v1 to v2)
    while x < v2.x:
        # Flip bits for z from z_left to z_right
        for z in range(int(z_left), int(z_right) + 1):
            if p_buffer.is_valid(x, y, z):
                p_buffer[x, y, z] = not p_buffer[x, y, z]
        x += 1
        z_left += m12
        z_right += m13
        
    # Lower triangle (v2 to v3)
    while x <= v3.x:
        for z in range(int(z_left), int(z_right) + 1):
            if p_buffer.is_valid(x, y, z):
                p_buffer[x, y, z] = not p_buffer[x, y, z]
        x += 1
        z_left += m23
        z_right += m13
```

**Parameters:**
- **P-buffer:** Stores presence values as a single bit (0 or 1) [S1, unofficial].
- **Color:** For homogeneous solids, all voxels have the same color. For heterogeneous solids, color depends on solid properties at the point [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Quadrilateral Slices:** When $B_y > y_i > C_y$, the slice is a quadrilateral, not a triangle. This is handled by decomposing it into two triangles using the intersection point $D_i$ on edge $BC$ [S1, unofficial].
- **Aliasing:** The main problem is aliasing, appearing by using integer arithmetic on floating objects. This minimizes but does not eliminate the effect [S1, unofficial].
- **Boundary-Only:** To obtain boundary-only rasterization, consider only spans between $B_i$ and $C_i$ (or $D_i$ and $C_i$ when $B_y > y > C_y$) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of faces of the solid and the resolution. Performance tests on a Pentium III 1GHz, 768 MB RAM showed that for a solid with 100,250 vertices and 202,520 faces at resolution 512, the method took 24.635 seconds, while Sramek's method took 237.601 seconds [S1, unofficial]. For boundary-only rasterization at 512 resolution, it took 4.715 seconds vs Sramek's 244.893 seconds [S1, unofficial].
- **Memory Complexity:** Requires a P-buffer of size proportional to the voxel grid resolution.
- **Strengths:** Obtains a complete voxelization of the solid in one pass. Simple and robust, suitable for graphic hardware implementation as it does not use additional data structures or complex operations [S1, unofficial].
- **Limitations:** Aliasing effects due to integer arithmetic. Performance depends on the number of faces, unlike Sramek's method which depends only on resolution and volume [S1, unofficial].

## Background
The method was presented in a paper titled "Voxelization of solids using simplicial coverings" [S1, unofficial]. It is designed for 3D displays and voxelization of solids delimited by planar faces, including those with holes, manifold or non-manifold structures [S1, unofficial].

## Sources
[S1] Voxelization of solids using simplicial coverings, http://wscg.zcu.cz/wscg2004/Papers_2004_Short/J53.pdf, fetched 2026-10-26 (unofficial)
