# Simplex Noise 2D

Also called: 2D OpenSimplex Noise, 3D Perlin Noise (Improved), Improved Noise, Perlin Noise (Improved), SNoise, Simplex Noise, 2D Simplex

## Overview
Simplex Noise 2D is a gradient-based procedural noise function for 2D space that generates smooth, continuous scalar fields using a skewed grid (simplex lattice), fixed gradient vectors, and a lookup table [S2, unofficial]. It solves the problem of directional artifacts and high computational complexity inherent in integer-lattice noise (like classic Perlin Noise) by using a triangular grid that requires fewer gradient evaluations and avoids axis-aligned bias [S4]. Unlike its sibling [Simplex-based Encoding](9582-simplex-based-encoding.md), which focuses on data compression or spatial indexing, Simplex Noise 2D is a generative function used directly as a density field $f(x,y)$ for volume rendering or texture synthesis [S3, unofficial].

## How it works
**Inputs:** A 2D coordinate vector $\mathbf{v} = (x, y)$.
**Outputs:** A scalar noise value $N \in [-1, 1]$ (normalized).

**Data Structures:**
1.  **Permutation Table (`perm`):** An array of 512 integers (indices 0–255 repeated twice) used to map grid coordinates to gradient indices. The base table contains a permutation of 0–255.
2.  **Gradient Lookup:** In the 2D implementation described in [S2, unofficial], gradients are often stored in a flat array `gradients2D` of size 512 (or derived from the permutation table), containing pre-defined 2D vectors. The sources [S2, unofficial] imply a lookup via a hash index derived from cell coordinates.

**Constants:**
*   $C_1 = \frac{\sqrt{3} - 1}{2}$ (Stretch factor for input)
*   $C_2 = \frac{3 - \sqrt{3}}{6}$ (Squish factor for output)
*   $G_1 = \frac{1}{\sqrt{3}}$ (Inverse of stretch factor, often used in alternative derivations, but [S2, unofficial] uses explicit constants)
*   $NORM_{2D} = \frac{1}{47.0}$ (Normalization constant)
*   $MOD_{289} = 289$ (Modulo base for permutation table)
*   $SCALE = 42.0$ (Final scaling factor)

**Numbered Steps:**

1.  **Stretch Input:**
    Apply a non-uniform scaling to the input coordinates to map the square input domain onto a skewed simplex grid.
    $$
    \begin{aligned}
    s &= (x + y) \times C_1 \\
    x_s &= x + s \\
    y_s &= y + s
    \end{aligned}
    $$
    Where $C_1 = \frac{\sqrt{3}-1}{2} \approx 0.36602540378$.

2.  **Find Cell Origin:**
    Determine the integer coordinates of the simplex triangle containing the input point.
    $$
    \begin{aligned}
    i_x &= \lfloor x_s \rfloor \\
    i_y &= \lfloor y_s \rfloor
    \end{aligned}
    $$
    Calculate the relative position of the input point within the skewed cell (distance from the cell origin in the skewed space).
    $$
    \begin{aligned}
    x_0 &= x_s - i_x \\
    y_0 &= y_s - i_y
    \end{aligned}
    $$

3.  **Determine Simplex Corners:**
    Identify the two other corners of the simplex triangle within the cell. The simplex grid in 2D consists of triangles. Based on the relative position $(x_0, y_0)$, determine which half of the square cell the point lies in.
    *   If $x_0 + y_0 < 1$: The point is in the lower-left triangle. The corners are $(0,0)$, $(1,0)$, and $(0,1)$.
    *   If $x_0 + y_0 \ge 1$: The point is in the upper-right triangle. The corners are $(1,1)$, $(1,0)$, and $(0,1)$.
    
    The sources [S1, S3, unofficial] describe a generalized 3D approach using step functions to determine offsets $i_1, i_2$. For 2D, this simplifies to determining the offsets for the second and third corners relative to the first corner $(0,0)$ in skewed space.
    
    Let the offsets for the three corners in skewed space be:
    *   Corner 0: $(0, 0)$
    *   Corner 1: $(1, 0)$ if $x_0+y_0 < 1$ else $(0, 1)$
    *   Corner 2: $(0, 1)$ if $x_0+y_0 < 1$ else $(1, 0)$
    
    *Note: [S2, unofficial] uses a hash-based lookup for gradients, implying a specific permutation mapping. We follow the standard geometric derivation which is consistent with the "skewed grid" description.*

4.  **Squish Output Coordinates:**
    Map the skewed coordinates back to the Euclidean space to calculate distances to the corners.
    $$
    \begin{aligned}
    t &= (i_x + i_y) \times C_2 \\
    x_{sq} &= i_x - t \\
    y_{sq} &= i_y - t
    \end{aligned}
    $$
    Calculate the vector from each corner to the original input point $(x, y)$ in Euclidean space.
    $$
    \begin{aligned}
    dx_0 &= x - x_{sq} \\
    dy_0 &= y - y_{sq}
    \end{aligned}
    $$
    Repeat for Corner 1 and Corner 2. Let the relative vectors be $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$.

5.  **Hash and Gradient Lookup:**
    Compute a hash index for each corner to retrieve the corresponding gradient vector.
    $$
    \text{hash} = \text{perm}[\text{perm}[i_x] + i_y] \pmod{256}
    $$
    (Note: [S2, unofficial] uses a more complex hash involving bit shifts for a linked-list lookup, but standard Simplex uses direct permutation indexing. We use the standard permutation indexing as it is the canonical implementation referenced by [S4] and implied by the "permutation table" in [S3, unofficial]).
    
    Retrieve gradient vector $\mathbf{g}_k$ for corner $k$. In 2D, gradients are typically chosen from a set of 4 or 8 vectors. [S2, unofficial] implies a lookup from `gradients2D`. A common set is $\{(1,1), (-1,1), (1,-1), (-1,-1)\}$ or normalized variants.

6.  **Calculate Contribution:**
    For each corner $k \in \{0, 1, 2\}$:
    *   Calculate the squared distance from the corner to the input point: $d_k^2 = dx_k^2 + dy_k^2$.
    *   Calculate the attenuation factor:
        $$
        n_k = \max(0.5 - d_k^2, 0.0)
        $$
        (Note: [S3, unofficial] uses $0.6$ for 3D. For 2D, the cutoff is typically $0.5$ because the simplex radius is smaller. [S2, unofficial] uses `2 - dx*dx - dy*dy` which suggests a different normalization or cutoff. However, standard Simplex 2D uses $0.5$. We stick to the standard $0.5$ cutoff for 2D simplex as per [S4]'s reference to Olano et al.).
    *   Square the attenuation twice to create a smooth falloff:
        $$
        n_k = n_k \times n_k
        $$
    *   Calculate the dot product with the gradient:
        $$
        \text{contrib}_k = n_k \times n_k \times (\mathbf{g}_k \cdot \mathbf{v}_k)
        $$

7.  **Accumulate and Normalize:**
    Sum the contributions:
    $$
    N_{raw} = \sum_{k=0}^{2} \text{contrib}_k
    $$
    Apply the normalization constant:
    $$
    N = N_{raw} \times NORM_{2D}
    $$
    Where $NORM_{2D} = 1.0 / 47.0$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
C1 = (math.sqrt(3.0) - 1.0) / 2.0
C2 = (3.0 - math.sqrt(3.0)) / 6.0
NORM_2D = 1.0 / 47.0

# Permutation table (example, typically 0-255 shuffled)
# In practice, this is a global or pre-computed array
PERM = list(range(256)) * 2 

# Gradient vectors for 2D (4 vectors)
GRADIENTS_2D = [
    (1, 1), (-1, 1), (1, -1), (-1, -1)
]

def permute(x):
    return PERM[x & 255]

def simplex_noise_2d(x, y):
    # 1. Stretch input
    s = (x + y) * C1
    x_s = x + s
    y_s = y + s
    
    # 2. Find cell origin
    i_x = math.floor(x_s)
    i_y = math.floor(y_s)
    
    # Relative position in skewed space
    x_0 = x_s - i_x
    y_0 = y_s - i_y
    
    # 3. Determine simplex corners
    # If x0 + y0 < 1, we are in the lower triangle (0,0)-(1,0)-(0,1)
    # Else, we are in the upper triangle (1,1)-(1,0)-(0,1)
    
    if x_0 + y_0 < 1:
        i1_x, i1_y = 1, 0
        i2_x, i2_y = 0, 1
    else:
        i1_x, i1_y = 0, 1
        i2_x, i2_y = 1, 0
        
    # 4. Squish output coordinates
    t = (i_x + i_y) * C2
    x_sq = i_x - t
    y_sq = i_y - t
    
    # Vectors from corners to input point
    dx_0 = x - x_sq
    dy_0 = y - y_sq
    
    dx_1 = dx_0 - i1_x + C2 # Offset for corner 1 in Euclidean space
    dy_1 = dy_0 - i1_y + C2
    
    dx_2 = dx_0 - i2_x + C2 # Offset for corner 2 in Euclidean space
    dy_2 = dy_0 - i2_y + C2
    
    # 5. Hash and Gradient Lookup
    # Calculate hash indices for the three corners
    h0 = permute(permute(i_x) + i_y)
    h1 = permute(permute(i_x + i1_x) + i_y + i1_y)
    h2 = permute(permute(i_x + i2_x) + i_y + i2_y)
    
    # Select gradients
    g0 = GRADIENTS_2D[h0 % 4]
    g1 = GRADIENTS_2D[h1 % 4]
    g2 = GRADIENTS_2D[h2 % 4]
    
    # 6. Calculate contributions
    # Attenuation cutoff is 0.5 for 2D
    n0 = 0.5 - (dx_0*dx_0 + dy_0*dy_0)
    n1 = 0.5 - (dx_1*dx_1 + dy_1*dy_1)
    n2 = 0.5 - (dx_2*dx_2 + dy_2*dy_2)
    
    # If attenuation is negative, contribution is 0
    if n0 < 0:
        contrib0 = 0.0
    else:
        n0 *= n0
        contrib0 = n0 * n0 * (g0[0]*dx_0 + g0[1]*dy_0)
        
    if n1 < 0:
        contrib1 = 0.0
    else:
        n1 *= n1
        contrib1 = n1 * n1 * (g1[0]*dx_1 + g1[1]*dy_1)
        
    if n2 < 0:
        contrib2 = 0.0
    else:
        n2 *= n2
        contrib2 = n2 * n2 * (g2[0]*dx_2 + g2[1]*dy_2)
        
    # 7. Accumulate and Normalize
    raw = contrib0 + contrib1 + contrib2
    return raw * NORM_2D
```

**Edge Cases and Pitfalls:**
*   **Precision:** Near the boundaries of the simplex cells, floating-point precision errors can cause discontinuities. Using double precision for intermediate calculations is recommended [G].
*   **Gradient Selection:** The choice of gradient vectors affects the visual quality. The 4-vector set $\{(1,1), (-1,1), (1,-1), (-1,-1)\}$ is standard for 2D. Using more gradients (e.g., 8) can reduce directional artifacts further but requires a larger permutation table or different indexing [G].
*   **Normalization:** The constant $1/47.0$ is specific to the implementation in [S2, unofficial]. Other implementations may use different normalizers (e.g., $1/0.6$ or $1/0.5$) depending on the gradient set and cutoff. Ensure consistency with the chosen gradient set.

## Subtopics

### [2D Octaved Noise with Memoization](4530-2d-octaved-noise-with-memoization.md)
Generates complex, self-similar noise patterns by summing multiple layers (octaves) of a base 2D gradient noise function, such as Simplex Noise 2D, with decreasing amplitude and increasing frequency. It solves the problem of repetitive or computationally expensive procedural texture generation by caching previously computed noise values for specific coordinate-octave combinations. Unlike standard octave noise which recomputes base noise for every sample, this method uses a cache to avoid redundant calculations, significantly improving performance for static or slowly varying scenes.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The number of operations is constant regardless of the input coordinate, involving a few arithmetic operations, a floor, and a few table lookups.
*   **Memory Complexity:** $O(1)$ if the permutation table is pre-computed and stored in a small array (256 or 512 integers). No large data structures are required.
*   **Measured Performance:** Significantly faster than classic Perlin Noise in 2D because it requires only 3 gradient evaluations instead of 4, and the simplex grid is more efficient in higher dimensions. [S4] notes that simplex noise lowers computational complexity.
*   **Strengths:** Eliminates directional artifacts present in integer-lattice noise. Smooth, continuous derivatives. Efficient computation.
*   **Limitations:** The skewed grid makes it slightly more complex to implement than Perlin Noise. The normalization constant is implementation-specific.
*   **When to Use:** Ideal for procedural terrain generation, texture synthesis, and volume density fields where directional bias is undesirable and performance is critical.
*   **When Not to Use:** If strict reproducibility with classic Perlin Noise is required, or if the specific artifact profile of Perlin Noise is desired for artistic reasons.

## Background
Simplex Noise was introduced by Ken Perlin in 2001 as an improvement over his original Perlin Noise [S4]. It was designed to reduce computational complexity and eliminate directional artifacts. The 2D variant is a specific case of the general simplex noise algorithm, adapted for 2D space. It is widely used in computer graphics for procedural generation [S3, unofficial]. The implementation details in [S2, unofficial] and [S1, unofficial] reflect common GLSL and C++ implementations found in open-source projects.

## Sources
[S1] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S2] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S3] FredrikErikJohansson/Proccraft, https://github.com/FredrikErikJohansson/Proccraft, fetched 2026-10-27 (unofficial)
[S4] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
