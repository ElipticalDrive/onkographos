# Simplex Noise 2D

Also called: SNoise, 2D Simplex

## Overview
Simplex Noise 2D is a gradient-based procedural noise function that generates smooth, continuous scalar fields using a skewed triangular grid, fixed gradient vectors, and a permutation lookup table [S3, S5, unofficial]. It addresses the visual anisotropy and high computational cost of higher-dimensional Perlin noise by replacing the axis-aligned hypercube lattice with a simplex grid [S1, unofficial]. The method computes contributions from three surrounding simplex corners instead of four or eight, eliminating directional artifacts [S1, S3, unofficial].

## How it works
**Inputs:** A 2D coordinate vector $\mathbf{v} = (x, y)$. [S3, S5, unofficial]
**Outputs:** A scalar noise value $N \in [-1, 1]$ (normalized). [S3, S5, unofficial]

**Data Structures:**
1. **Permutation Table (`perm`):** An array of 512 integers (indices 0–255 repeated twice) used to map grid coordinates to gradient indices via bitwise wrapping. [S3, S5, unofficial]
2. **Gradient Array (`grad3`):** An array of 12 gradient vectors indexed 0–11, used for 2D and 3D dot products. [S3, S5, unofficial]

**Constants:**
* $F_2 = \frac{\sqrt{3}-1}{2} \approx 0.36602540378$ (2D skew factor) [S3, unofficial]
* $G_2 = \frac{3-\sqrt{3}}{6} \approx 0.21132486540$ (2D unskew factor) [S3, unofficial]
* $C = (F_2, G_2)$ [S3, S5, unofficial]
* $D = (0, 0.5, 1, 2)$ [S3, S5, unofficial]
* $MOD = 289$ (Modulo base for permutation table) [S3, S5, unofficial]
* $SCALE = 70.0$ (Final scaling factor) [S3, S5, unofficial]

**Numbered Steps:**
1. **Skew Input:** Apply a non-uniform scaling to the input coordinates to map the square input domain onto a skewed simplex grid. Calculate $s = (x + y) \times F_2$. Determine integer cell coordinates $i = \lfloor x + s \rfloor$ and $j = \lfloor y + s \rfloor$. [S3, S5, unofficial]
2. **Unskew Cell Origin:** Calculate $t = (i + j) \times G_2$. Calculate unskewed cell origin $X_0 = i - t$, $Y_0 = j - t$. [S3, S5, unofficial]
3. **Relative Position:** Calculate distances from origin $x_0 = x - X_0$, $y_0 = y - Y_0$. [S3, S5, unofficial]
4. **Simplex Selection:** Determine which half of the simplex cell the point lies in. If $x_0 > y_0$, select lower triangle with offsets $(i_1, j_1) = (1, 0)$ and $(i_2, j_2) = (1, 1)$. Else, select upper triangle with offsets $(i_1, j_1) = (0, 1)$ and $(i_2, j_2) = (1, 1)$. [S3, S5, unofficial]
5. **Corner Offsets:** Calculate offsets for the middle corner $(x_1, y_1)$ and last corner $(x_2, y_2)$ in unskewed space: $x_1 = x_0 - i_1 + G_2$, $y_1 = y_0 - j_1 + G_2$, $x_2 = x_0 - i_2 + 2G_2$, $y_2 = y_0 - j_2 + 2G_2$. [S3, S5, unofficial]
6. **Gradient Lookup:** Hash cell coordinates to get gradient indices $g_0, g_1, g_2$ using the permutation table: $g_0 = \text{perm}[\text{perm}[i] + j] \pmod{12}$, $g_1 = \text{perm}[\text{perm}[i+i_1] + j+j_1] \pmod{12}$, $g_2 = \text{perm}[\text{perm}[i+i_2] + j+j_2] \pmod{12}$. [S3, S5, unofficial]
7. **Contribution Calculation:** For each corner $k \in \{0, 1, 2\}$, calculate $t_k = 0.5 - x_k^2 - y_k^2$. If $t_k < 0$, contribution $n_k = 0$. Else, $n_k = t_k^2 \times t_k^2 \times \text{dot}(\text{grad3}[g_k], x_k, y_k)$. [S3, S5, unofficial]
8. **Final Result:** Sum contributions and scale by $70.0$. Result is in $[-1, 1]$. [S3, S5, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
F2 = (math.sqrt(3.0) - 1.0) / 2.0
G2 = (3.0 - math.sqrt(3.0)) / 6.0
SCALE = 70.0

# Permutation table (256 values repeated to 512)
PERM = list(range(256)) * 2

# 12 gradient vectors for 2D/3D
GRAD3 = [
    (1, 1), (-1, 1), (1, -1), (-1, -1),
    (1, 0), (-1, 0), (0, 1), (0, -1),
    (1, 1), (-1, 1), (1, -1), (-1, -1)
]

def permute(x):
    return PERM[x & 255]

def simplex_noise_2d(x, y):
    # 1. Skew input
    s = (x + y) * F2
    i = math.floor(x + s)
    j = math.floor(y + s)
    
    # 2. Unskew cell origin
    t = (i + j) * G2
    X0 = i - t
    Y0 = j - t
    
    # 3. Relative position
    x0 = x - X0
    y0 = y - Y0
    
    # 4. Simplex selection
    if x0 > y0:
        i1, j1 = 1, 0
        i2, j2 = 1, 1
    else:
        i1, j1 = 0, 1
        i2, j2 = 1, 1
        
    # 5. Corner offsets in unskewed space
    x1 = x0 - i1 + G2
    y1 = y0 - j1 + G2
    x2 = x0 - i2 + 2 * G2
    y2 = y0 - j2 + 2 * G2
    
    # 6. Gradient lookup
    g0 = permute(permute(i) + j) % 12
    g1 = permute(permute(i + i1) + j + j1) % 12
    g2 = permute(permute(i + i2) + j + j2) % 12
    
    # 7. Contributions
    n0 = 0.5 - x0*x0 - y0*y0
    if n0 < 0:
        contrib0 = 0.0
    else:
        n0 *= n0
        contrib0 = n0 * n0 * (GRAD3[g0][0]*x0 + GRAD3[g0][1]*y0)
        
    n1 = 0.5 - x1*x1 - y1*y1
    if n1 < 0:
        contrib1 = 0.0
    else:
        n1 *= n1
        contrib1 = n1 * n1 * (GRAD3[g1][0]*x1 + GRAD3[g1][1]*y1)
        
    n2 = 0.5 - x2*x2 - y2*y2
    if n2 < 0:
        contrib2 = 0.0
    else:
        n2 *= n2
        contrib2 = n2 * n2 * (GRAD3[g2][0]*x2 + GRAD3[g2][1]*y2)
        
    # 8. Final result
    return SCALE * (contrib0 + contrib1 + contrib2)
```

**Edge Cases and Pitfalls:**
* **Precision:** Near the boundaries of the simplex cells, floating-point precision errors can cause discontinuities. Using double precision for intermediate calculations is recommended [G].
* **Gradient Selection:** The choice of gradient vectors affects the visual quality. The 12-vector set is standard for 2D/3D implementations to ensure isotropy. Using fewer gradients requires careful indexing to avoid repetition artifacts [G].
* **Normalization:** The constant $70.0$ ensures the output range is approximately $[-1, 1]$. Other implementations may use different normalizers depending on the gradient set and cutoff. Ensure consistency with the chosen gradient set [S3, S5, unofficial].

## Subtopics

### [2D Octaved Noise with Memoization](4530-2d-octaved-noise-with-memoization.md)
Generates complex, self-similar noise patterns by summing multiple layers (octaves) of a base 2D gradient noise function, such as Simplex Noise 2D, with decreasing amplitude and increasing frequency. It solves the problem of repetitive or computationally expensive procedural texture generation by caching previously computed noise values for specific coordinate-octave combinations. Unlike standard octave noise which recomputes base noise for every sample, this method uses a cache to avoid redundant calculations, significantly improving performance for static or slowly varying scenes.

## Operating characteristics
* **Time Complexity:** $O(1)$ per sample. The number of operations is constant regardless of the input coordinate, involving a few arithmetic operations, a floor, and a few table lookups. [S1, unofficial]
* **Memory Complexity:** $O(1)$ if the permutation table is pre-computed and stored in a small array (256 or 512 integers). No large data structures are required. [S3, S5, unofficial]
* **Measured Performance:** Significantly faster than classic Perlin Noise in 2D because it requires only 3 gradient evaluations instead of 4, and the simplex grid is more efficient in higher dimensions. [S1, unofficial]
* **Strengths:** Eliminates directional artifacts present in integer-lattice noise. Smooth, continuous derivatives. Efficient computation. [S1, S3, unofficial]
* **Limitations:** The skewed grid makes it slightly more complex to implement than Perlin Noise. The normalization constant is implementation-specific. [S3, S5, unofficial]
* **When to Use:** Ideal for procedural terrain generation, texture synthesis, and volume density fields where directional bias is undesirable and performance is critical. [S1, unofficial]
* **When Not to Use:** If strict reproducibility with classic Perlin Noise is required, or if the specific artifact profile of Perlin Noise is desired for artistic reasons. [G]

## Background
Simplex Noise was introduced by Ken Perlin to address shortcomings of the original Perlin noise, specifically visual anisotropy and computational cost in higher dimensions [S1, unofficial]. The 2D variant is a specific case of the general simplex noise algorithm, adapted for 2D space using a skewed triangular grid. It is widely used in computer graphics for procedural generation [S3, S5, unofficial]. The implementation details in [S3, S5, unofficial] reflect common GLSL and C++ implementations found in open-source projects.

## Sources
[S1] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
[S2] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S3] Rydgel/Vokuselu, https://github.com/Rydgel/Vokuselu, fetched 2026-10-27 (unofficial)
[S4] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
[S5] FredrikErikJohansson/Proccraft, https://github.com/FredrikErikJohansson/Proccraft, fetched 2026-10-27 (unofficial)
