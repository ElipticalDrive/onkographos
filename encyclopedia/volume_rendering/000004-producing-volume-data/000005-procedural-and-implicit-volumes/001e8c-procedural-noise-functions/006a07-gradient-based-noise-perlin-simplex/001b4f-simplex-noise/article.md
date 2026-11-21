# Simplex Noise

Also called: 3D Simplex Noise, SNoise, Simplex Noise 3D

## Overview
Simplex Noise is a gradient-based procedural noise function that generates smooth, continuous scalar fields by mapping input coordinates to a skewed simplex lattice (triangles in 2D, tetrahedra in 3D, pentachora in 4D) and summing weighted dot products of gradient vectors [S1][S2, S4, S6, unofficial]. It solves the problem of directional artifacts and high computational complexity inherent in the original integer-lattice Perlin Noise by utilizing a simplex grid, which provides a more even distribution of grid points and fewer gradient contributions per sample [S1][S2, unofficial][S11]. Unlike standard Perlin Noise which uses a square grid and requires four gradient evaluations in 2D, Simplex Noise 2D uses a triangular grid and requires only three gradient evaluations, lowering computational complexity and eliminating undesired directional artifacts [S2, unofficial][S11]. It scales to higher dimensions ($n$) with complexity $O(n^2)$ compared to $O(n 2^n)$ for classic noise, making it easier to implement in hardware and suitable for higher-dimensional volume rendering [S11].

## How it works
**Inputs and Outputs:**
- Input: Query point $\mathbf{P} = (x, y, z)$ in 3D space (or $\mathbf{P} = (x, y)$ in 2D).
- Output: Scalar noise value $N \in [-1, 1]$.

**Data Structures:**
1.  **Permutation Table (`perm`)**: An array of 512 integers, constructed by doubling a 256-element permutation table `p` (containing a shuffled sequence of 0–255) to avoid index wrapping modulo arithmetic [S2, S7, S13, unofficial].
    -   Layout: `perm[i] = p[i & 255]` for $i \in [0, 511]$.
    -   Typical `p` initialization: A shuffled array of 0–255, e.g., `[151, 160, 137, 91, ...]` [S2, S7, unofficial].
2.  **Gradient Vectors (`grad3`)**: An array of 12 3D vectors representing the face diagonals of a cube [S2, S7, S13, unofficial].
    -   Layout: `grad3[k]` for $k \in [0, 11]$.
    -   Values:
        ```python
        grad3 = [
            [1, 1, 0], [-1, 1, 0], [1, -1, 0], [-1, -1, 0],
            [1, 0, 1], [-1, 0, 1], [1, 0, -1], [-1, 0, -1],
            [0, 1, 1], [0, -1, 1], [0, 1, -1], [0, -1, -1]
        ]
        ```
    -   Note: Some implementations normalize these or map permutation indices to a 7x7 octahedron grid to derive gradients [S6, S9, unofficial]. The standard implementation uses the fixed diagonals above [S2, S7, unofficial].

**Parameters:**
-   $F_3 = 1/3$: Skew factor for 3D input space [S2, S7, S10, unofficial].
-   $G_3 = 1/6$: Unskew factor for 3D output space [S2, S7, S10, unofficial].
-   Scaling factor: $32.0$ to map result to $[-1, 1]$ [S2, S7, S8, unofficial].

**Algorithm Steps (3D):**

1.  **Skew Input Space**:
    Calculate the skew factor $s$ and determine the integer simplex cell corner $(i, j, k)$ containing $\mathbf{P}$:
    $$ s = (x + y + z) \cdot F_3 $$
    $$ i = \lfloor x + s \rfloor $$
    $$ j = \lfloor y + s \rfloor $$
    $$ k = \lfloor z + s \rfloor $$

2.  **Unskew Cell Origin**:
    Calculate the unskew factor $t$ and the origin of the simplex cell $(X_0, Y_0, Z_0)$ in original coordinates:
    $$ t = (i + j + k) \cdot G_3 $$
    $$ X_0 = i - t $$
    $$ Y_0 = j - t $$
    $$ Z_0 = k - t $$

3.  **Relative Coordinates**:
    Calculate the vector $\mathbf{x}_0$ from the cell origin to the query point:
    $$ x_0 = x - X_0 $$
    $$ y_0 = y - Y_0 $$
    $$ z_0 = z - Z_0 $$

4.  **Determine Simplex Corners**:
    Identify the offsets $(i_1, j_1, k_1)$ and $(i_2, j_2, k_2)$ for the second and third corners of the tetrahedron based on the relative position of $(x_0, y_0, z_0)$ within the base simplex [S2, S7, unofficial].
    ```python
    if x0 >= y0:
        if y0 >= z0:
            i1, j1, k1 = 1, 0, 0
            i2, j2, k2 = 1, 1, 0
        elif x0 >= z0:
            i1, j1, k1 = 1, 0, 0
            i2, j2, k2 = 1, 1, 1
        else:
            i1, j1, k1 = 0, 0, 1
            i2, j2, k2 = 1, 1, 1
    else:
        if y0 < z0:
            i1, j1, k1 = 0, 0, 1
            i2, j2, k2 = 0, 1, 1
        elif x0 < z0:
            i1, j1, k1 = 0, 1, 0
            i2, j2, k2 = 0, 1, 1
        else:
            i1, j1, k1 = 0, 1, 0
            i2, j2, k2 = 1, 1, 1
    ```

5.  **Compute Corner Offsets**:
    Calculate the unskewed relative coordinates for the remaining three corners of the simplex [S2, S7, unofficial]:
    $$ x_1 = x_0 - i_1 + G_3, \quad y_1 = y_0 - j_1 + G_3, \quad z_1 = z_0 - k_1 + G_3 $$
    $$ x_2 = x_0 - i_2 + 2G_3, \quad y_2 = y_0 - j_2 + 2G_3, \quad z_2 = z_0 - k_2 + 2G_3 $$
    $$ x_3 = x_0 - 1 + 3G_3, \quad y_3 = y_0 - 1 + 3G_3, \quad z_3 = z_0 - 1 + 3G_3 $$

6.  **Hash Gradient Indices**:
    Map the integer coordinates of the four corners to gradient indices using the permutation table [S2, S7, unofficial]:
    ```python
    ii = i & 255
    jj = j & 255
    kk = k & 255
    gi0 = perm[ii + perm[jj + perm[kk]]] % 12
    gi1 = perm[ii + i1 + perm[jj + j1 + perm[kk + k1]]] % 12
    gi2 = perm[ii + i2 + perm[jj + j2 + perm[kk + k2]]] % 12
    gi3 = perm[ii + 1 + perm[jj + 1 + perm[kk + 1]]] % 12
    ```

7.  **Calculate Contributions**:
    For each corner $k \in \{0, 1, 2, 3\}$ with relative vector $(x_k, y_k, z_k)$ and gradient index $g_k$:
    $$ t_k = 0.6 - x_k^2 - y_k^2 - z_k^2 $$
    If $t_k < 0$, contribution $n_k = 0$.
    Else:
    $$ n_k = t_k^2 \cdot t_k^2 \cdot (g_{k,x} x_k + g_{k,y} y_k + g_{k,z} z_k) $$
    where $(g_{k,x}, g_{k,y}, g_{k,z})$ is the gradient vector from `grad3[g_k]` [S2, S7, S8, unofficial].

8.  **Final Result**:
    $$ N = 32.0 \cdot (n_0 + n_1 + n_2 + n_3) $$

**Edge Cases and Pitfalls:**
-   **Index Wrapping**: The permutation table is doubled to 512 entries to allow direct indexing without modulo on the table index itself, only on the gradient index (modulo 12) [S2, S7, unofficial].
-   **Precision**: Floating-point precision issues can arise near cell boundaries; using double precision for intermediate calculations is recommended for high-quality rendering [G].
-   **Patent**: Simplex Noise is patented and cannot be used in commercial implementations without license [S14, unofficial].

## Subtopics

### [Simplex Noise 2D](4155-simplex-noise-2d.md)
Simplex Noise 2D is a gradient-based procedural noise function for 2D space that generates smooth, continuous scalar fields by using a skewed grid, fixed gradient vectors, and a lookup table. It solves the problem of directional artifacts and high computational complexity inherent in the original integer-lattice Perlin Noise by utilizing a simplex (triangular) grid, which provides more even distribution of grid points and fewer gradient contributions per sample. Unlike standard Perlin Noise which uses a square grid and requires four gradient evaluations in 2D, Simplex Noise 2D uses a triangular grid and requires only three gradient evaluations, lowering computational complexity and eliminating undesired directional artifacts. It builds on [Simplex Noise](978-simplex-noise.md) by specializing the algorithm for two dimensions.

### [Simplex-based Encoding](9582-simplex-based-encoding.md)
Simplex-based encoding is a method for representing graphics primitives (such as images, Signed Distance Fields, and NeRFs) using simplex-based structures instead of traditional grid-based structures. It solves the problem of exponential computational overhead ($2^n$ vertices) inherent in grid-based representations by reducing the number of vertices per cell to linear scaling ($n+1$ vertices) with dimensionality. It sets itself apart from its siblings, such as [Simplex Noise](978-simplex-noise.md) which generates scalar fields, by using the simplex topology specifically for efficient feature storage and interpolation in high-dimensional encoding spaces, often combined with hash tables and neural networks.

### [Simplex Noise 3D Gradient](3496-simplex-noise-3d-gradient.md)
Computes 3D Simplex Noise and its gradient (derivative) simultaneously. It performs the same skewing, corner identification, and surflet coordinate calculations as `SimplexNoise3D`. It computes contribution weights $t_k = \max(0.6 - \mathbf{x}_k \cdot \mathbf{x}_k, 0)$ and gradient vectors $\mathbf{g}_k$ [G]. The derivative is calculated as $\mathbf{d} = -8 \sum t_k^2 \mathbf{x}_k (\mathbf{g}_k \cdot \mathbf{x}_k) + \sum t_k^4 \mathbf{g}_k$ [G].

### Simplex Cell Location
The process of locating the specific simplex cell containing a point $\mathbf{x}$ within the unit hypercube by sorting the coordinates of $\mathbf{x}$ and determining the permutation of vertices [S15, unofficial]. It sorts the entries of the skewed input point $\mathbf{x}$ in descending order to obtain $x_1, \dots, x_n$, records the original indices, and constructs the vertex path from $\mathbf{0}$ to $\mathbf{1}$ by adding 1 to the base vertex at each sorted index [S15, unofficial]. Bubble sort is used for small $n$ [S15, unofficial]. Sorting must preserve original indices to correctly map sorted coordinate values back to vertex offsets [S15, unofficial].

### Simplex-based Coordinate Skewing
An affine transformation that maps a sampled point $\mathbf{x}$ in $\mathbb{R}^n$ to a skewed coordinate $\mathbf{x}'$ by multiplying it with a constant matrix [S15, unofficial]. This transforms the unit $n$-hypercube into an $n$-parallelpiped, reducing distortion in simplex cells and avoiding axis-aligned artifacts [S15, unofficial]. The formula is $\mathbf{x}' = \mathbf{M} \mathbf{x}$, where $M_{ii} = 1 + F_n$ and $M_{ij} = 1$ for $i \neq j$, with $F_n = \frac{\sqrt{n+1}-1}{n}$ [S15, unofficial]. For $n=2$, this creates equilateral triangles; for $n=3$, it creates tetrahedrons with congruent isosceles faces [S15, unofficial]. Without skewing, cells are $n$-orthoschemes, causing axis-aligned artifacts [S15, unofficial].

## Operating characteristics
-   **Time Complexity**: $O(n^2)$ for $n$ dimensions, compared to $O(n 2^n)$ for classic Perlin noise [S11]. In 3D, it requires 3 gradient evaluations (vs 8 for Perlin) and fewer multiplications [S11].
-   **Memory Complexity**: Requires a 512-entry permutation table and 12 gradient vectors for 3D [S2, S7, unofficial].
-   **Performance**: Scales efficiently to higher dimensions (4D, 5D) and is easier to implement in hardware [S11].
-   **Strengths**: Eliminates directional artifacts present in cubic grid noise; more uniform distribution of grid points; lower computational cost in higher dimensions [S1, S11].
-   **Limitations**: Patented, restricting commercial use [S14, unofficial]; slightly more complex implementation than Perlin due to simplex traversal logic [S11].
-   **When to Use**: When directional artifacts are unacceptable, when higher-dimensional noise (4D+) is needed, or when computational efficiency in hardware is critical [S11].
-   **When Not to Use**: In commercial products where patent licensing is not feasible [S14, unofficial].

## Background
Simplex Noise was introduced by Stefan Gustavson, optimized by Peter Eastman, with a better rank ordering method from 2012 [S2, unofficial]. It was designed to address the limitations of Perlin Noise, specifically the directional artifacts and high computational cost in higher dimensions [S1, S11].

## Sources
[S1] A Survey of Procedural Noise Functions, https://doi.org/10.1111/j.1467-8659.2010.01827.x, fetched 2026-10-27
[S2] Show HN: Basic voxel engine in WebGL/JavaScript with full source, https://github.com/Lallassu/vox2, fetched 2026-10-26 (unofficial)
[S3] Pseudorandom Noise for Real-Time Volumetric Rendering of Fire in a Production System, https://doi.org/10.2312/vg/vg-pbg08/129-136, fetched 2026-10-26
[S4] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S5] SebLague/Godot-Marching-Cubes, https://github.com/SebLague/Godot-Marching-Cubes, fetched 2026-10-26 (unofficial)
[S6] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
[S7] tcgj/terrain-generator, https://github.com/tcgj/terrain-generator, fetched 2026-10-27 (unofficial)
[S8] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S9] billiam-dev/Isosurface-Generation, https://github.com/billiam-dev/Isosurface-Generation, fetched 2026-10-27 (unofficial)
[S10] jshrake/lexov, https://github.com/jshrake/lexov, fetched 2026-10-27 (unofficial)
[S11] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
[S12] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S13] Markyparky56/dualmc-compute, https://github.com/Markyparky56/dualmc-compute, fetched 2026-10-27 (unofficial)
[S14] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S15] Efficient Encoding of Graphics Primitives with Simplex-based Structures, https://doi.org/10.48550/arxiv.2311.15439, fetched 2026-10-26 (unofficial)
