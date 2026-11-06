# Conservative Voxelization via Depth Range

Also called: Depth range voxelization

## Overview
Conservative Voxelization via Depth Range is a GPU-based rasterization technique that converts 3D triangle meshes into a binary voxel grid by computing the depth range of a projected triangle over each fragment rectangle and encoding occupancy bits into a texture [S1, unofficial]. It differs from standard rasterization by ensuring that any voxel intersecting the triangle is marked as occupied, avoiding the "missing voxel" artifacts common in non-conservative methods, and differs from slicemap-based approaches by explicitly calculating geometric intersection bounds rather than relying on bitwise blending of discrete depth slices [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D triangle mesh defined by vertices $P_i = (x_i, y_i, z_i)$ for $i \in \{1, 2, 3\}$, and a fragment rectangle $R$ in screen space defined by $x_{min}, x_{max}, y_{min}, y_{max}$ [S1, unofficial].
*   **Output:** A bitmask representing the occupancy of voxels $(i, j, k)$ for the fragment's column, where '1' indicates intersection and '0' indicates no intersection [S1, unofficial].

**Data Structures:**
*   **Lookup Table Texture:** A 2D texture storing precomputed bitmasks for all $(z^*_{min}, z^*_{max})$ pairs, where $z^* = z \times 32$ [S1, unofficial].
    *   Standard mode (32 slices): Size $32 \times 32$. Format RGBA8. Each texel stores a 32-bit mask [S1, unofficial].
    *   MRT mode (128 slices): Size $512 \times 128$. Format RGBA8. Each texel stores 32 bits. Four consecutive texels store a 128-bit mask [S1, unofficial].
*   **Fragment Rectangle ($R$):** The axis-aligned bounding box of the fragment in screen space, defined by $x_{min}, x_{max}, y_{min}, y_{max}$ [S1, unofficial].

**Parameters:**
*   $\epsilon_I$: Threshold for detecting parallel lines and degenerate edges [S1, unofficial].
*   $\epsilon_S$: Threshold for detecting degenerate triangles based on projected area $S$ [S1, unofficial].
*   $\epsilon_m$: Machine error constant used to initialize enlarged width/height $x_E, y_E$ [S1, unofficial].
*   $\epsilon_{float}$: Floating point error constant ($10^{-5}$) added to the computed depth range to address limited GPU precision [S1, unofficial].

**Algorithm Steps:**

1.  **Vertex Processing (Partial Derivatives):**
    For each vertex $P_i$, compute the partial derivatives of barycentric coordinates and depth in the vertex processing stage [S1, unofficial]. Output $C^* = C(P_i) - \Delta \mathbf{C}$ as a 3D vector [S1, unofficial].
    *   Compute signed area $S$ of the projected triangle:
        $$ S = x_1 y_2 - x_2 y_1 + x_2 y_3 - x_3 y_2 + x_3 y_1 - x_1 y_3 $$
    *   Compute partial derivatives of barycentric coordinates $\mathbf{C}$:
        $$ \frac{\partial \mathbf{C}}{\partial x} = \frac{(y_2 - y_3, y_3 - y_1, y_1 - y_2)}{S} $$
        $$ \frac{\partial \mathbf{C}}{\partial y} = \frac{(x_3 - x_2, x_1 - x_3, x_2 - x_1)}{S} $$
    *   Compute partial derivatives of depth $z$:
        $$ \frac{\partial z}{\partial x} = (z_1, z_2, z_3) \cdot \frac{\partial \mathbf{C}}{\partial x} $$
        $$ \frac{\partial z}{\partial y} = (z_1, z_2, z_3) \cdot \frac{\partial \mathbf{C}}{\partial y} $$

2.  **Intersection Computation:**
    Compute intersection points $Q$ between each edge of the triangle $T_p$ and each edge of the fragment rectangle $R$ by calculating line intersections [S1, unofficial].
    *   For an edge $P_i P_j$ and a horizontal rectangle edge (e.g., top or bottom), calculate coefficients:
        $$ a_{ij}^y = \frac{1}{y_j - y_i}, \quad b_{ij}^y = \frac{-y_i}{y_j - y_i} $$
    *   Calculate parameter $t$ for intersection with $y_{max}$ (or $y_{min}$):
        $$ t = a_{ij}^y \cdot y_{max} + b_{ij}^y $$
    *   Calculate intersection point $Q$:
        $$ Q = \mathcal{L}(P_i, P_j, t) $$
        where $\mathcal{L}$ denotes linear interpolation [S1, unofficial].
    *   Calculate $z(Q) = \mathcal{L}(z_i, z_j, t)$ [S1, unofficial].
    *   Check if $Q$ lies on both the triangle edge and the rectangle edge. If true, keep $z(Q)$; otherwise, assign $z(Q)$ a huge value (infinity) [S1, unofficial].
    *   Repeat for all 12 combinations (3 edges $\times$ 4 rectangle edges) [S1, unofficial].

3.  **Depth Range Determination:**
    *   Compute the minimal depth value among the 12 intersection points to determine $z_{min}$ [S1, unofficial].
    *   Compute the maximal depth value to determine $z_{max}$ [S1, unofficial].
    *   Apply optimization: Use GPU 4D vector processing to compute intersections of one triangle edge with four rectangle edges simultaneously. Compare 12 depth values using four comparisons (two 4D vectors, one 2D vector, one scalar) [S1, unofficial].

4.  **Special Case Handling (Robustness):**
    *   **Parallel Lines:** If $|\frac{y_j - y_i}{x_j - x_i}| < \epsilon_I$, the edge is nearly horizontal [S1, unofficial]. Set $a_{ij}^y = 0, b_{ij}^y = 2$ to discard intersections with top/bottom edges [S1, unofficial]. Enlarge rectangle height by $\epsilon_I$ [S1, unofficial]. Test intersection with left/right edges by checking $y_{min} - \epsilon_I \le y \le y_{max} + \epsilon_I$ [S1, unofficial].
    *   **Degenerate Edges:** If $|x_j - x_i| < \epsilon_I$ and $|y_j - y_i| < \epsilon_I$, assume both vertices are inside the fragment [S1, unofficial]. Check if the third vertex $P_k$ is inside $R$ using the enlarged bounds [S1, unofficial].
    *   **Degenerate Triangles:** If $|S| < \epsilon_S$, barycentric coordinates are unreliable [S1, unofficial]. Set $C^* = (-1, -1, -1)$ [S1, unofficial]. Enlarge rectangle width and height by $\sqrt{2}h$, where $h$ is the minimal height of the triangle [S1, unofficial].

5.  **Voxel Encoding:**
    *   Enlarge the depth range by a small constant value ($10^{-5}$) to address limited GPU precision [S1, unofficial].
    *   Map depth range to voxel indices: $z^*_{min} = z_{min} \times 32$, $z^*_{max} = z_{max} \times 32$ [S1, unofficial].
    *   Perform a texture lookup in the Lookup Table using $(z^*_{min}, z^*_{max})$ to retrieve the bitmask [S1, unofficial].
    *   Write the bitmask to the fragment color buffer. If using MRT, write to multiple targets to cover 128 slices [S1, unofficial].

6.  **Conservative Rasterization Integration:**
    *   Enlarge each triangle by half a pixel to form a bounding polygon [S1, unofficial].
    *   Draw the optimal bounding triangle and cull redundant fragments to improve efficiency [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def conservative_voxelization(vertices, rect, epsilon_I, epsilon_S, epsilon_m, epsilon_float, lookup_table):
    """
    Computes conservative depth range and encodes voxels.
    vertices: list of 3 tuples (x, y, z)
    rect: dict with keys x_min, x_max, y_min, y_max
    epsilon_I: threshold for parallel lines/degenerate edges
    epsilon_S: threshold for degenerate triangles
    epsilon_m: machine error constant
    epsilon_float: floating point error constant (1e-5)
    lookup_table: 2D array for bitmask lookup
    Returns: bitmask
    """
    x1, y1, z1 = vertices[0]
    x2, y2, z2 = vertices[1]
    x3, y3, z3 = vertices[2]
    
    # 1. Compute Signed Area S
    S = (x1 * y2 - x2 * y1 + x2 * y3 - x3 * y2 + x3 * y1 - x1 * y3)
    
    # 2. Handle Degenerate Triangle
    if abs(S) < epsilon_S:
        # Calculate minimal height h of the triangle
        h = min(
            ((x2-x1)**2 + (y2-y1)**2)**0.5,
            ((x3-x2)**2 + (y3-y2)**2)**0.5,
            ((x1-x3)**2 + (y1-y3)**2)**0.5
        )
        # Enlarge rectangle bounds by sqrt(2)*h
        rect['x_min'] -= (2**0.5) * h
        rect['x_max'] += (2**0.5) * h
        rect['y_min'] -= (2**0.5) * h
        rect['y_max'] += (2**0.5) * h
        # C* = (-1, -1, -1) is handled by skipping derivative computation
    else:
        # Compute partial derivatives for barycentric coordinates and depth
        dCdx = ((y2 - y3) / S, (y3 - y1) / S, (y1 - y2) / S)
        dCdy = ((x3 - x2) / S, (x1 - x3) / S, (x2 - x1) / S)
        dzdx = z1 * dCdx[0] + z2 * dCdx[1] + z3 * dCdx[2]
        dzdy = z1 * dCdy[0] + z2 * dCdy[1] + z3 * dCdy[2]
        
    # 3. Initialize enlarged rectangle dimensions with machine error
    xE = epsilon_m
    yE = epsilon_m
    
    depths = []
    edges = [(vertices[0], vertices[1]), (vertices[1], vertices[2]), (vertices[2], vertices[0])]
    
    for p1, p2 in edges:
        xi, yi, zi = p1
        xj, yj, zj = p2
        dx = xj - xi
        dy = yj - yi
        
        # Handle parallel lines (nearly horizontal edge)
        if abs(dx) > 1e-9 and abs(dy / dx) < epsilon_I:
            ay = 0.0
            by = 2.0
            yE = max(yE, epsilon_I)
            # Discard top/bottom intersections, test left/right with enlarged bounds
            if abs(dx) > epsilon_I:
                ax = 1.0 / dx
                bx = -xi / dx
                for x_val in [rect['x_min'], rect['x_max']]:
                    t = ax * x_val + bx
                    if 0 <= t <= 1:
                        Qz = zi + t * (zj - zi)
                        depths.append(Qz)
            continue
        else:
            ay = 1.0 / dy
            by = -yi / dy
            # Intersect with top/bottom edges
            for y_val in [rect['y_min'], rect['y_max']]:
                t = ay * y_val + by
                if 0 <= t <= 1:
                    Qz = zi + t * (zj - zi)
                    depths.append(Qz)
                    
        # Intersect with left/right edges
        if abs(dx) > epsilon_I:
            ax = 1.0 / dx
            bx = -xi / dx
            for x_val in [rect['x_min'], rect['x_max']]:
                t = ax * x_val + bx
                if 0 <= t <= 1:
                    Qz = zi + t * (zj - zi)
                    depths.append(Qz)
        else:
            # Vertical edge
            if rect['x_min'] - epsilon_I <= xi <= rect['x_max'] + epsilon_I:
                depths.append(zi)
                depths.append(zj)
                
    # 4. Determine depth range
    if not depths:
        z_min = float('-inf')
        z_max = float('inf')
    else:
        z_min = min(depths)
        z_max = max(depths)
        
    # 5. Enlarge depth range for precision
    z_min -= epsilon_float
    z_max += epsilon_float
        
    # 6. Encode voxels
    z_min_idx = int(z_min * 32)
    z_max_idx = int(z_max * 32)
    z_min_idx = max(0, min(z_min_idx, 31))
    z_max_idx = max(0, min(z_max_idx, 31))
    
    bitmask = lookup_table[z_min_idx][z_max_idx]
    return bitmask
```

## Subtopics

### Slicemap Boundary Voxelization
A technique to represent a binary 3D grid of voxels using a single 2D texture, where the $k$-th bit of the RGBA representation of pixel $(i, j)$ encodes the value of voxel $(i, j, k)$ [S2, unofficial]. It renders the scene and, for each fragment, if its depth lies in the $k$-th slice, outputs a color having only the $k$-th bit activated [S2, unofficial]. It uses a bitwise OR blending mode to combine all fragments projecting to the same pixel, allowing tracking of all fragments in a single pass rather than discarding all but the closest [S2, unofficial]. Unlike Conservative Voxelization via Depth Range, it does not compute geometric intersection bounds but relies on depth quantization and bitwise operations [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of triangles, as each triangle is processed in a single pass (or few passes for MRT). The intersection computation is constant time per triangle (12 intersections) [S1, unofficial].
*   **Memory Complexity:** $O(res_x \times res_y \times \frac{res_z}{32})$ for the output texture. The lookup table is small ($32 \times 32$ or $512 \times 128$ texels) [S1, unofficial].
*   **Performance:** The method leverages GPU native 4D vector processing for intersection calculations, significantly accelerating the depth range computation [S1, unofficial]. MRT support allows processing 128 slices in a single pass, reducing the number of rendering passes required for high-resolution volumes [S1, unofficial].
*   **Accuracy:** Theoretically accurate; misses no voxels. The number of redundant voxels is no more than 0.1% of the total number [S1, unofficial].
*   **Limitations:** Requires hardware support for MRT and bitwise blending operations. The lookup table size grows with the number of slices if not using MRT. Parallel line and degenerate triangle cases require careful handling to maintain correctness [S1, unofficial].

## Background
The method was introduced in a paper on conservative voxelization, which proposed computing the depth range of a projected triangle over a fragment rectangle to encode occupancy bits into a texture [S1, unofficial]. It builds upon the concept of slicemaps, where a 3D grid is represented by a 2D texture using bits of RGBA channels [S2, unofficial]. The approach addresses the limitations of pre-computation based approaches that use switch-case structures and costly intersection point location computations, offering a more hardware-friendly implementation using native GPU vector operations [S1, unofficial].

## Sources
[S1] Conservative voxelization, https://wo1fsea.github.io/files/About_NavMesh/P1_Conservative_voxelization.pdf, fetched 2026-10-27 (unofficial)
[S2] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
