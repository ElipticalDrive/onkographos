# Simplex Noise

Also called: 3D Simplex Noise, Gradient Noise, Perlin Noise Implementation, Perlin Simplex Noise, Stefan Gustavson Simplex Noise, snoise

## Overview
Simplex Noise is a gradient-based procedural noise function that generates smooth, continuous scalar fields by evaluating dot products between input coordinates and pseudo-random gradient vectors assigned to simplex cell vertices [S1, S3, S5, unofficial]. It solves the directional artifacts and computational inefficiency of cubic-grid Perlin noise by using a skewed simplex (triangular in 2D, tetrahedral in 3D) grid, which reduces the number of gradient evaluations and dot products required per sample [S1, S3, unofficial][S6]. Unlike its sibling [Simplex Noise 2D](4155-simplex-noise-2d.md) which operates in 2D, this 3D variant extends the simplex grid to tetrahedra to reduce directional artifacts and computational cost compared to cubic-grid noise [S1, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{P} = (x, y, z)$ in 3D space.
*   **Output:** A scalar noise value $N \in [-1, 1]$.

**Data Structures:**
1.  **Permutation Table (`perm`):** An array of 512 integers [S1, unofficial]. It is constructed by doubling a 256-element permutation table `p` (containing a shuffled sequence of 0–255) to avoid modulo arithmetic during index lookup [S1, S7, unofficial].
    *   Layout: `perm[i] = p[i & 255]` for $i \in [0, 511]$.
2.  **Gradient Table (`grad3`):** An array of 12 gradient vectors [S1, unofficial]. Each vector has 3 components (float). The vectors are face diagonals of a cube:
    *   Indices 0-3: $(\pm 1, \pm 1, 0)$
    *   Indices 4-7: $(\pm 1, 0, \pm 1)$
    *   Indices 8-11: $(0, \pm 1, \pm 1)$
    *   Exact values: `[[1,1,0], [-1,1,0], [1,-1,0], [-1,-1,0], [1,0,1], [-1,0,1], [1,0,-1], [-1,0,-1], [0,1,1], [0,-1,1], [0,1,-1], [0,-1,-1]]` [S1, S7, unofficial].

**Parameters:**
*   **Skew Factor ($F_3$):** $1/3$ [S1, S7, unofficial].
*   **Unskew Factor ($G_3$):** $1/6$ [S1, S7, unofficial].
*   **Scaling Constant:** $32.0$ [S1, S4, unofficial].

**Steps:**
1.  **Skew Input Space:** Calculate the skewed coordinate $s = (x + y + z) \times F_3$ [S1, unofficial]. Determine the integer simplex corner containing the origin: $i = \lfloor x + s \rfloor$, $j = \lfloor y + s \rfloor$, $k = \lfloor z + s \rfloor$ [S1, S4, unofficial].
2.  **Unskew Origin:** Calculate the unskew factor $t = (i + j + k) \times G_3$ [S1, unofficial]. Determine the unskewed origin of the cell: $X_0 = i - t$, $Y_0 = j - t$, $Z_0 = k - t$ [S1, S4, unofficial].
3.  **Relative Coordinates:** Calculate the vector from the cell origin to the query point: $x_0 = x - X_0$, $y_0 = y - Y_0$, $z_0 = z - Z_0$ [S1, S4, unofficial].
4.  **Simplex Selection:** Determine the offsets to the other three corners of the tetrahedron based on the relative position of $(x_0, y_0, z_0)$ [S1, S4, unofficial]:
    *   If $x_0 \ge y_0$:
        *   If $y_0 \ge z_0$: $(i_1, j_1, k_1) = (1, 0, 0)$, $(i_2, j_2, k_2) = (1, 1, 1)$ [S1, unofficial].
        *   Else if $x_0 < z_0$: $(i_1, j_1, k_1) = (0, 0, 1)$, $(i_2, j_2, k_2) = (1, 1, 1)$ [S1, unofficial].
        *   Else: $(i_1, j_1, k_1) = (1, 0, 0)$, $(i_2, j_2, k_2) = (1, 0, 1)$ [S1, unofficial].
    *   Else ($x_0 < y_0$):
        *   If $y_0 < z_0$: $(i_1, j_1, k_1) = (0, 0, 1)$, $(i_2, j_2, k_2) = (0, 1, 1)$ [S1, unofficial].
        *   Else if $x_0 < z_0$: $(i_1, j_1, k_1) = (0, 1, 0)$, $(i_2, j_2, k_2) = (0, 1, 1)$ [S1, unofficial].
        *   Else: $(i_1, j_1, k_1) = (0, 1, 0)$, $(i_2, j_2, k_2) = (1, 1, 0)$ [S1, unofficial].
5.  **Corner Offsets:** Calculate the unskewed coordinates for the four corners relative to the origin [S1, S4, unofficial]:
    *   $\mathbf{x}_0 = (x_0, y_0, z_0)$
    *   $\mathbf{x}_1 = (x_0 - i_1 + G_3, y_0 - j_1 + G_3, z_0 - k_1 + G_3)$ [S1, unofficial].
    *   $\mathbf{x}_2 = (x_0 - i_2 + 2G_3, y_0 - j_2 + 2G_3, z_0 - k_2 + 2G_3)$ [S1, unofficial].
    *   $\mathbf{x}_3 = (x_0 - 1 + 3G_3, y_0 - 1 + 3G_3, z_0 - 1 + 3G_3)$ [S1, unofficial].
6.  **Gradient Lookup:** Compute the permutation indices for the four corners using the permutation table [S1, S4, unofficial]:
    *   $ii = i \& 255$, $jj = j \& 255$, $kk = k \& 255$ [S1, unofficial].
    *   $gi_0 = perm[ii + perm[jj + perm[kk]]] \pmod{12}$ [S1, unofficial].
    *   $gi_1 = perm[ii + i_1 + perm[jj + j_1 + perm[kk + k_1]]] \pmod{12}$ [S1, unofficial].
    *   $gi_2 = perm[ii + i_2 + perm[jj + j_2 + perm[kk + k_2]]] \pmod{12}$ [S1, unofficial].
    *   $gi_3 = perm[ii + 1 + perm[jj + 1 + perm[kk + 1]]] \pmod{12}$ [S1, unofficial].
7.  **Contribution Calculation:** For each corner $k \in \{0, 1, 2, 3\}$, calculate the radial falloff factor $t_k = 0.6 - x_k^2 - y_k^2 - z_k^2$ [S1, S4, unofficial].
    *   If $t_k < 0$, the contribution $n_k = 0$ [S1, unofficial].
    *   Else, $n_k = t_k^4 \times \text{dot3}(\text{grad3}[gi_k], x_k, y_k, z_k)$, where $\text{dot3}(\mathbf{g}, \mathbf{x}) = g_x x + g_y y + g_z z$ [S1, S4, unofficial].
8.  **Final Result:** Sum the contributions and scale: $N = 32.0 \times (n_0 + n_1 + n_2 + n_3)$ [S1, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
F3 = 1.0 / 3.0
G3 = 1.0 / 6.0
# PERM is a 512-entry array constructed by doubling a shuffled 0-255 permutation table [S1, unofficial].
# Example initialization: p = [151, 160, 137, ...]; PERM = p + p
PERM = [151, 160, 137, 91, 90, 15, 131, 13, 201, 95, 96, 53, 194, 233, 7, 225, 140, 36, 103, 30, 69, 142, 8, 99, 37, 240, 21, 10, 23, 50, 52, 147, 144, 181, 182, 223, 226, 105, 208, 198, 214, 216, 244, 254, 46, 122, 11, 65, 250, 76, 159, 168, 17, 133, 84, 123, 54, 136, 220, 165, 146, 206, 185, 19, 209, 245, 143, 109, 125, 228, 219, 203, 117, 35, 16, 249, 247, 202, 191, 253, 197, 75, 94, 7, 12, 115, 124, 116, 119, 145, 174, 6, 162, 200, 235, 171, 150, 104, 92, 199, 170, 89, 118, 148, 163, 100, 231, 67, 139, 29, 157, 120, 108, 188, 190, 207, 155, 177, 113, 152, 186, 156, 172, 184, 196, 134, 129, 127, 211, 213, 255, 251, 248, 221, 222, 243, 242, 241, 239, 238, 237, 236, 234, 232, 230, 229, 227, 224, 220, 218, 217, 215, 212, 210, 205, 204, 202, 201, 199, 198, 197, 195, 194, 193, 192, 191, 190, 189, 187, 185, 184, 183, 182, 181, 180, 179, 178, 177, 176, 175, 174, 173, 172, 171, 170, 169, 168, 167, 166, 165, 164, 163, 162, 161, 160, 159, 158, 157, 156, 155, 154, 153, 152, 151, 150, 149, 148, 147, 146, 145, 144, 143, 142, 141, 140, 139, 138, 137, 136, 135, 134, 133, 132, 131, 130, 129, 128, 127, 126, 125, 124, 123, 122, 121, 120, 119, 118, 117, 116, 115, 114, 113, 112, 111, 110, 109, 108, 107, 106, 105, 104, 103, 102, 101, 100, 99, 98, 97, 96, 95, 94, 93, 92, 91, 90, 89, 88, 87, 86, 85, 84, 83, 82, 81, 80, 79, 78, 77, 76, 75, 74, 73, 72, 71, 70, 69, 68, 67, 66, 65, 64, 63, 62, 61, 60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0]

GRAD3 = [
    [1, 1, 0], [-1, 1, 0], [1, -1, 0], [-1, -1, 0],
    [1, 0, 1], [-1, 0, 1], [1, 0, -1], [-1, 0, -1],
    [0, 1, 1], [0, -1, 1], [0, 1, -1], [0, -1, -1]
]

def simplex_noise_3d(x, y, z):
    # 1. Skew input space
    s = (x + y + z) * F3
    i = floor(x + s)
    j = floor(y + s)
    k = floor(z + s)
    
    # 2. Unskew origin
    t = (i + j + k) * G3
    X0 = i - t
    Y0 = j - t
    Z0 = k - t
    
    # 3. Relative coordinates
    x0 = x - X0
    y0 = y - Y0
    z0 = z - Z0
    
    # 4. Simplex selection
    if x0 >= y0:
        if y0 >= z0:
            i1, j1, k1 = 1, 0, 0
            i2, j2, k2 = 1, 1, 1
        elif x0 < z0:
            i1, j1, k1 = 0, 0, 1
            i2, j2, k2 = 1, 1, 1
        else:
            i1, j1, k1 = 1, 0, 0
            i2, j2, k2 = 1, 0, 1
    else:
        if y0 < z0:
            i1, j1, k1 = 0, 0, 1
            i2, j2, k2 = 0, 1, 1
        elif x0 < z0:
            i1, j1, k1 = 0, 1, 0
            i2, j2, k2 = 0, 1, 1
        else:
            i1, j1, k1 = 0, 1, 0
            i2, j2, k2 = 1, 1, 0
            
    # 5. Corner offsets
    x1 = x0 - i1 + G3
    y1 = y0 - j1 + G3
    z1 = z0 - k1 + G3
    
    x2 = x0 - i2 + 2.0 * G3
    y2 = y0 - j2 + 2.0 * G3
    z2 = z0 - k2 + 2.0 * G3
    
    x3 = x0 - 1.0 + 3.0 * G3
    y3 = y0 - 1.0 + 3.0 * G3
    z3 = z0 - 1.0 + 3.0 * G3
    
    # 6. Gradient lookup
    ii = i & 255
    jj = j & 255
    kk = k & 255
    
    gi0 = PERM[ii + PERM[jj + PERM[kk]]] % 12
    gi1 = PERM[ii + i1 + PERM[jj + j1 + PERM[kk + k1]]] % 12
    gi2 = PERM[ii + i2 + PERM[jj + j2 + PERM[kk + k2]]] % 12
    gi3 = PERM[ii + 1 + PERM[jj + 1 + PERM[kk + 1]]] % 12
    
    # 7. Contribution calculation
    n0 = 0.0
    t0 = 0.6 - x0*x0 - y0*y0 - z0*z0
    if t0 > 0:
        t0 *= t0
        n0 = t0 * t0 * (GRAD3[gi0][0]*x0 + GRAD3[gi0][1]*y0 + GRAD3[gi0][2]*z0)
        
    n1 = 0.0
    t1 = 0.6 - x1*x1 - y1*y1 - z1*z1
    if t1 > 0:
        t1 *= t1
        n1 = t1 * t1 * (GRAD3[gi1][0]*x1 + GRAD3[gi1][1]*y1 + GRAD3[gi1][2]*z1)
        
    n2 = 0.0
    t2 = 0.6 - x2*x2 - y2*y2 - z2*z2
    if t2 > 0:
        t2 *= t2
        n2 = t2 * t2 * (GRAD3[gi2][0]*x2 + GRAD3[gi2][1]*y2 + GRAD3[gi2][2]*z2)
        
    n3 = 0.0
    t3 = 0.6 - x3*x3 - y3*y3 - z3*z3
    if t3 > 0:
        t3 *= t3
        n3 = t3 * t3 * (GRAD3[gi3][0]*x3 + GRAD3[gi3][1]*y3 + GRAD3[gi3][2]*z3)
        
    # 8. Final result
    return 32.0 * (n0 + n1 + n2 + n3)
```

## Subtopics

### [Simplex Noise 2D](4155-simplex-noise-2d.md)
Simplex Noise 2D is a gradient-based procedural noise function for 2D space that generates smooth, continuous scalar values. It uses simplex (triangular/hexagonal) grids and fixed gradient vectors to compute noise efficiently.

### [Simplex-based Encoding](9582-simplex-based-encoding.md)
A method for encoding graphics primitives (images, SDFs, NeRFs) using simplex-based structures instead of grid-based structures to reduce computational overhead from exponential scaling ($2^n$ vertices) to linear scaling ($n+1$ vertices) with dimensionality.

### Simplex Noise 3D Gradient
Computes 3D Simplex Noise and its gradient (derivative) simultaneously [S7, unofficial]. It performs the same skewing, corner identification, and surflet coordinate calculations as `SimplexNoise3D` [S7, unofficial]. It computes contribution weights $t_k = \max(0.6 - \text{dot}(\mathbf{surf}_k, \mathbf{surf}_k), 0)$ and gradient vectors $\mathbf{grad}_k$ [S7, unofficial]. The derivative is computed as $\text{deriv} = -8 \times (t_0^2 \mathbf{surf}_0 + t_1^2 \mathbf{surf}_1 + t_2^2 \mathbf{surf}_2 + t_3^2 \mathbf{surf}_3) \times \text{grads} + t_0^4 \mathbf{grad}_0 + t_1^4 \mathbf{grad}_1 + t_2^4 \mathbf{grad}_2 + t_3^4 \mathbf{grad}_3$ [S7, unofficial]. The value is scaled by $46.0$ and the derivative by $7.4$ [S7, unofficial].

### Simplex Cell Location
The process of locating the specific simplex cell containing a point $\mathbf{x}$ within the unit hypercube by sorting the coordinates of $\mathbf{x}$ and determining the permutation of vertices [S9, unofficial]. It sorts the entries of the skewed input point $\mathbf{x}$ in descending order to obtain $x_1, \dots, x_n$, records the original indices, and builds the vertex path from $\mathbf{0}^T$ to $\mathbf{1}^T$ by adding 1 to the base vertex at each sorted index [S9, unofficial]. Bubble sort is used since $n$ is small [S9, unofficial]. Sorting must preserve original indices to correctly map the sorted coordinate values back to the vertex offsets in the hypercube grid [S9, unofficial].

### Simplex-based Coordinate Skewing
An affine transformation that maps a sampled point $\mathbf{x}$ in $\mathbb{R}^n$ to a skewed coordinate $\mathbf{x}'$ by multiplying it with a constant matrix [S9, unofficial]. The formula is $\mathbf{x}' = \begin{bmatrix} 1+F_n & \cdots & 1 \\ \vdots & \ddots & \vdots \\ 1 & \cdots & 1+F_n \end{bmatrix} \mathbf{x}$, where $F_n = \frac{\sqrt{n+1}-1}{n}$ [S9, unofficial]. For $n=2$, this creates equilateral triangles; for $n=3$, it creates tetrahedrons with congruent isosceles faces [S9, unofficial]. Without skewing, cells are $n$-orthoschemes, causing axis-aligned artifacts [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(n^2)$ for $n$ dimensions, compared to $O(n 2^n)$ for classic Perlin noise [S8, unofficial].
*   **Memory Complexity:** $O(1)$ for the algorithm itself, plus $O(2^n)$ for the permutation table (typically 256 or 512 entries) [S1, S7, unofficial].
*   **Measured Performance:** Requires fewer multiplications than classic Perlin noise [S8, unofficial].
*   **Strengths:** Reduced directional artifacts [S1, unofficial], lower computational cost, scales well to higher dimensions (4D, 5D), easier to implement in hardware [S8, unofficial].
*   **Limitations:** Patented, which restricts commercial use [S8, unofficial].
*   **When to use:** When directional artifacts in Perlin noise are visible or when performance is critical in higher dimensions [S8, unofficial].
*   **When not to use:** In commercial products where patent licensing is an issue, or when exact reproducibility of Perlin noise is required [S8, unofficial].

## Background
Simplex Noise was introduced by Stefan Gustavson, optimized by Peter Eastman, with a better rank ordering method from 2012 [S1, unofficial]. It is based on the simplex grid (triangles in 2D, tetrahedra in 3D) to reduce directional artifacts and computational cost compared to Perlin noise [S1, S5, unofficial]. It is used for generating procedural 3D continuous scalar fields, which can serve as the density function $f(x,y,z)$ for isosurface extraction [S4, unofficial].

## Sources
[S1] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S2] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S3] SebLague/Godot-Marching-Cubes, https://github.com/SebLague/Godot-Marching-Cubes, fetched 2026-10-26 (unofficial)
[S4] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S5] billiam-dev/Isosurface-Generation, https://github.com/billiam-dev/Isosurface-Generation, fetched 2026-10-27 (unofficial)
[S6] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S7] Markyparky56/dualmc-compute, https://github.com/Markyparky56/dualmc-compute, fetched 2026-10-27 (unofficial)
[S8] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S9] Efficient Encoding of Graphics Primitives with Simplex-based Structures, https://doi.org/10.48550/arxiv.2311.15439, fetched 2026-10-26 (unofficial)
