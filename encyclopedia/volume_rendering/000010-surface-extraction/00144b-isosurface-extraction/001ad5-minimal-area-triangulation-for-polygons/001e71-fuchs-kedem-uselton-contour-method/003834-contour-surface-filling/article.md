# Contour Surface Filling

Also called: Color-filled contour plot, Color-filling a contour surface, Triangle partitioning for contour surfaces

## Overview
Contour Surface Filling is a surface rendering technique that visualizes 3D vector fields by partitioning triangles on coordinate-parallel planes into polygons based on linearly interpolated vector magnitudes and filling them with colors mapped to contour levels [S1, unofficial]. It differs from wireframe or arrow-based vector visualizations by providing a continuous color-coded magnitude representation across the surface, building on the geometric partitioning logic of the [Fuchs-Kedem-Uselton Contour Method](1780-fuchs-kedem-uselton-contour-method.md) but specifically applied to pre-defined planar surfaces rather than implicit isosurfaces [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D grid of vector data where each grid point stores coordinates and a vector magnitude, and a surface defined as a plane parallel to one of the coordinate axes (xy, yz, or zx) [S1, unofficial].
*   **Output:** A rasterized image where each triangle on the surface is filled with colors corresponding to the vector magnitude at that location [S1, unofficial].

**Data Structures**
*   `vector`: An array storing vector data where the magnitude for vertex $i$ is accessed at index `4*i + 3` [S1, unofficial].
*   `indc`: An array mapping vertex indices to contour level indices [S1, unofficial].
*   `Contours[NC]`: An array of length $NC$ storing an increasing sequence of vector magnitude values that define the upper limits of the contour levels [S1, unofficial].
*   `Colors[NC][3]`: An array of size $NC \times 3$ storing the RGB components for each contour level [S1, unofficial].
*   `vertex[5][3]`: A working space array of size $5 \times 3$ to store the vertices of the partitioned polygons, allocated because the algorithm guarantees that no polygon generated has more than 5 vertices [S1, unofficial].

**Number of Triangles**
The number of triangles to be processed depends on the grid dimensions ($n_x, n_y, n_z$) and the plane orientation [S1, unofficial]:
*   **xy-plane:** $2(n_x-1)(n_y-1)$
*   **yz-plane:** $2(n_y-1)(n_z-1)$
*   **zx-plane:** $2(n_z-1)(n_x-1)$

**Algorithm Steps**
The algorithm processes each triangle individually [S1, unofficial]. Let the triangle vertices be $p_1, p_2, p_3$ with coordinates $(x_1, y_1, z_1), (x_2, y_2, z_2), (x_3, y_3, z_3)$ and magnitudes $m_1, m_2, m_3$ [S1, unofficial].

1.  **Permutation:** Permute vertex indices $i_1, i_2, i_3$ such that $indc[i_1] \le indc[i_2]$ and $indc[i_1] \le indc[i_3]$ [S1, unofficial].
2.  **Initialization:**
    *   Set $ic_1 = indc[i_1]$, $ic_2 = indc[i_2]$, $ic_3 = indc[i_3]$ [S1, unofficial].
    *   Retrieve magnitudes: $m_1 = vector[4*i_1+3]$, $m_2 = vector[4*i_2+3]$, $m_3 = vector[4*i_3+3]$ [S1, unofficial].
    *   Retrieve coordinates for $p_1, p_2, p_3$ [S1, unofficial].
    *   Initialize vertex counter $n = 0$ [S1, unofficial].
    *   Set `vertex[0]` = $(x_1, y_1, z_1)$ [S1, unofficial].
3.  **First Loop (ic from $ic_1$ to $ic_2 - 1$):**
    *   Let $m = Contours[ic]$ [S1, unofficial].
    *   Calculate intersection point $P(x_p, y_p, z_p)$ on edge $p_1-p_2$:
        $$s = \frac{m - m_1}{m_2 - m_1}$$
        $$x_p = x_1 + s(x_2 - x_1), \quad y_p = y_1 + s(y_2 - y_1), \quad z_p = z_1 + s(z_2 - z_1)$$
    *   Calculate intersection point $Q(x_q, y_q, z_q)$:
        *   If $ic < ic_3$: $Q$ is on edge $p_1-p_3$.
            $$s = \frac{m - m_1}{m_3 - m_1}$$
            $$x_q = x_1 + s(x_3 - x_1), \quad y_q = y_1 + s(y_3 - y_1), \quad z_q = z_1 + s(z_3 - z_1)$$
        *   If $ic \ge ic_3$: $Q$ is on edge $p_2-p_3$.
            $$s = \frac{m - m_3}{m_2 - m_3}$$
            $$x_q = x_3 + s(x_2 - x_3), \quad y_q = y_3 + s(y_2 - y_3), \quad z_q = z_3 + s(z_2 - z_3)$$
            *(Note: The source text contains a typo in the formula for $x_q, y_q, z_q$ in this branch, listing $x_2-x_1$ etc. The standard linear interpolation on edge $p_2-p_3$ starting from $p_3$ uses $x_2-x_3$. Implementers must correct this [G].)*
    *   Add $P$ and $Q$ to the vertex array: `vertex[n++]` = $P$, `vertex[n++]` = $Q$ [S1, unofficial].
    *   If $ic == ic_3$, add vertex $p_3$: `vertex[n++]` = $(x_3, y_3, z_3)$ [S1, unofficial].
    *   Fill polygon using `vertex` array with $n$ vertices and color `Colors[ic]` [S1, unofficial].
    *   Reset $n = 2$ [S1, unofficial].
    *   Set `vertex[0]` = $Q$, `vertex[1]` = $P$ [S1, unofficial].
4.  **Post-First Loop:**
    *   Add vertex $p_2$: `vertex[n++]` = $(x_2, y_2, z_2)$ [S1, unofficial].
5.  **Second Loop (if $ic_3 \ge ic_2$):**
    *   Loop $ic$ from $ic_2$ to $ic_3 - 1$:
        *   Let $m = Contours[ic]$ [S1, unofficial].
        *   Calculate $P$ on edge $p_2-p_3$:
            $$s = \frac{m - m_2}{m_3 - m_2}$$
            $$x_p = x_2 + s(x_3 - x_2), \quad y_p = y_2 + s(y_3 - y_2), \quad z_p = z_2 + s(z_3 - z_2)$$
        *   Calculate $Q$ on edge $p_1-p_3$:
            $$s = \frac{m - m_1}{m_3 - m_1}$$
            $$x_q = x_1 + s(x_3 - x_1), \quad y_q = y_1 + s(y_3 - y_1), \quad z_q = z_1 + s(z_3 - z_1)$$
        *   Add $P$ and $Q$ to `vertex` [S1, unofficial].
        *   Fill polygon with color `Colors[ic]` [S1, unofficial].
        *   Reset $n = 2$ [S1, unofficial].
        *   Set `vertex[0]` = $Q$, `vertex[1]` = $P$ [S1, unofficial].
    *   Add vertex $p_3$: `vertex[n++]` = $(x_3, y_3, z_3)$ [S1, unofficial].
    *   Set $ic = ic_3$ [S1, unofficial].
6.  **Final Fill:**
    *   Fill the final polygon using `vertex` array with $n$ vertices and color `Colors[ic]` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fill_triangle(i1, i2, i3, vector, indc, contours, colors, get_coords):
    # 1. Permute so indc[i1] is minimum
    if indc[i1] > indc[i2]:
        i1, i2 = i2, i1
    if indc[i1] > indc[i3]:
        i1, i3 = i3, i1
    if indc[i2] > indc[i3]:
        i2, i3 = i3, i2

    ic1 = indc[i1]
    ic2 = indc[i2]
    ic3 = indc[i3]

    m1 = vector[4 * i1 + 3]
    m2 = vector[4 * i2 + 3]
    m3 = vector[4 * i3 + 3]

    x1, y1, z1 = get_coords(i1)
    x2, y2, z2 = get_coords(i2)
    x3, y3, z3 = get_coords(i3)

    n = 0
    vertex = [[0.0, 0.0, 0.0] for _ in range(5)]
    vertex[0] = [x1, y1, z1]

    # First loop: ic from ic1 to ic2 - 1
    for ic in range(ic1, ic2):
        m = contours[ic]
        
        # Intersection P on edge p1-p2
        s = (m - m1) / (m2 - m1)
        xp = x1 + s * (x2 - x1)
        yp = y1 + s * (y2 - y1)
        zp = z1 + s * (z2 - z1)
        
        # Intersection Q
        if ic < ic3:
            # Q on edge p1-p3
            s = (m - m1) / (m3 - m1)
            xq = x1 + s * (x3 - x1)
            yq = y1 + s * (y3 - y1)
            zq = z1 + s * (z3 - z1)
        else:
            # Q on edge p2-p3
            # Corrected from source typo: uses (x2 - x3) instead of (x2 - x1)
            s = (m - m3) / (m2 - m3)
            xq = x3 + s * (x2 - x3)
            yq = y3 + s * (y2 - y3)
            zq = z3 + s * (z2 - z3)
            
        vertex[n] = [xp, yp, zp]
        n += 1
        vertex[n] = [xq, yq, zq]
        n += 1
        
        if ic == ic3:
            vertex[n] = [x3, y3, z3]
            n += 1
            
        # Rasterize polygon with n vertices using color colors[ic]
        # fill_polygon(vertex[:n], colors[ic])
        
        n = 2
        vertex[0] = [xq, yq, zq]
        vertex[1] = [xp, yp, zp]

    # Add vertex 2
    vertex[n] = [x2, y2, z2]
    n += 1

    # Second loop: if ic3 >= ic2
    if ic3 >= ic2:
        for ic in range(ic2, ic3):
            m = contours[ic]
            
            # P on edge p2-p3
            s = (m - m2) / (m3 - m2)
            xp = x2 + s * (x3 - x2)
            yp = y2 + s * (y3 - y2)
            zp = z2 + s * (z3 - z2)
            
            # Q on edge p1-p3
            s = (m - m1) / (m3 - m1)
            xq = x1 + s * (x3 - x1)
            yq = y1 + s * (y3 - y1)
            zq = z1 + s * (z3 - z1)
            
            vertex[n] = [xp, yp, zp]
            n += 1
            vertex[n] = [xq, yq, zq]
            n += 1
            
            # Rasterize polygon with n vertices using color colors[ic]
            # fill_polygon(vertex[:n], colors[ic])
            
            n = 2
            vertex[0] = [xq, yq, zq]
            vertex[1] = [xp, yp, zp]
            
        vertex[n] = [x3, y3, z3]
        n += 1
        ic = ic3

    # Final fill
    # Rasterize polygon with n vertices using color colors[ic]
    # fill_polygon(vertex[:n], colors[ic])
```

**Parameters**
*   **NC:** Number of contour levels, which determines the size of the `Colors` and `Contours` arrays [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero:** If $m_2 = m_1$, $m_3 = m_1$, or $m_3 = m_2$, the interpolation factor $s$ involves division by zero, which occurs when vector magnitudes at vertices are identical [G].
*   **Source Typo:** The source text for the case where $Q$ is on edge $p_2-p_3$ provides the formula `xq = x3 + s*(x2 – x1)`. This is geometrically incorrect for linear interpolation along the segment $p_2-p_3$; it should be `x2 - x3` [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles on the surface, scaling as $O(n_x n_y + n_y n_z + n_z n_x)$ for rendering all three orthogonal planes [S1, unofficial].
*   **Memory Complexity:** $O(NC)$ for the contour and color tables, plus $O(1)$ auxiliary space per triangle due to the fixed-size `vertex` array [S1, unofficial].
*   **Strengths:** Simple implementation; provides intuitive color-coded magnitude visualization; efficient rasterization of planar surfaces [S1, unofficial].
*   **When to use:** When visualizing vector field magnitudes on specific orthogonal cross-sections where color coding is preferred over arrows or streamlines [S1, unofficial].

## Background
The method was described in the context of visualizing 3D vector fields as part of a "color-filled contour plot" feature [S1, unofficial]. It was presented in a report discussing the visualization of surfaces and 3D vector fields, utilizing linear interpolation for magnitude mapping and pre-stored RGB tables for coloring [S1, unofficial]. The technique is distinct from the Fuchs-Kedem-Uselton method, which focuses on the geometric extraction of the isosurface itself, whereas this method assumes the surface geometry (triangles) is already defined and focuses on the attribute-based coloring and partitioning [S1, unofficial].

## Sources
[S1] Visualization of Surfaces and 3D Vector Fields, https://digital.library.unt.edu/ark:/67531/metadc3235/?q=vector field volume rendering, fetched 2026-10-28 (unofficial)
