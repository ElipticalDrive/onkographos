# Perlin Gradient and Interpolation Functions

Also called: Fade, Grad, Lerp

## Overview
This article defines the core mathematical primitives—gradient selection, fading, and interpolation—used to construct Perlin noise [S1, unofficial][S2][S3, unofficial]. It solves the problem of generating continuous, differentiable scalar fields from discrete lattice points, which is essential for procedural volume data [G]. It differs from [Simplex Noise](978-simplex-noise.md) by relying on an axis-aligned cubic lattice and standard trilinear interpolation of gradient dot products, whereas Simplex Noise uses a simplex grid to reduce directional artifacts [G].

## How it works
The method combines three components: a gradient vector generator, a smoothing (fade) function, and linear interpolation [S1, unofficial].

**Inputs and Outputs**
- Input: Query point $\mathbf{P} = (x, y, z)$, a permutation table $P$, and gradient vectors $\mathbf{g}$ [S1, unofficial].
- Output: Scalar noise value $n$ [S1, unofficial].

**Data Structures**
- **Permutation Table**: An array used to map integer coordinates to gradient indices, avoiding modulo operations [S1, unofficial].
- **Gradient Selection Logic**: Uses `hash & 15` to pick one of 12 gradient directions, mapping to x, y, or z with sign flips based on bits 0 and 1 [S1, unofficial].

**Steps**
1. **Integer and Fractional Parts**: Compute integer coordinates $\mathbf{I} = \lfloor \mathbf{P} \rfloor$ and fractional coordinates $\mathbf{F} = \mathbf{P} - \mathbf{I}$ [G].
2. **Gradient Selection**: For each of the 8 corners of the unit cell containing $\mathbf{P}$, compute a hash index using the permutation table. Map this index to a gradient vector using `hash & 15` [S1, unofficial].
3. **Dot Product Contribution**: For each corner $i$, compute the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$. Compute the contribution $n_i = \mathbf{g}_i \cdot \mathbf{d}_i$ [G].
4. **Smoothing (Fade)**: Apply the fade function $f(t)$ to each component of $\mathbf{F}$ to obtain smooth interpolation weights $u_x, u_y, u_z$. Formula: $f(t) = 6t^5 - 15t^4 + 10t^3$ [S1, S3, unofficial].
5. **Interpolation**: Combine the 8 contributions using trilinear interpolation weighted by $u_x, u_y, u_z$ [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fade(t):
    """Quintic smoothing function for C1 continuity."""
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0)

def lerp(t, a, b):
    """Linear interpolation."""
    return a + t * (b - a)

def grad(hash_val, x, y, z):
    """
    Selects a gradient vector and computes the dot product with (x, y, z).
    Uses 12 gradient vectors from the face diagonals of a cube.
    """
    h = hash_val & 15
    u = x if h < 8 else y
    v = y if h < 4 else (x if h == 12 or h == 14 else z)
    if h & 1:
        u = -u
    if h & 2:
        v = -v
    return u + v

def perlin_noise(p, perm_table):
    """
    Computes 3D Perlin noise.
    p: tuple (x, y, z)
    perm_table: list of 256 integers
    """
    x, y, z = p
    xi, yi, zi = int(math.floor(x)), int(math.floor(y)), int(math.floor(z))
    xf, yf, zf = x - xi, y - yi, z - zi

    u = fade(xf)
    v = fade(yf)
    w = fade(zf)

    # Hashing corners (assuming 256-entry table masked)
    A = perm_table[xi & 255] + yi
    AA = perm_table[A & 255] + zi
    AB = perm_table[(A + 1) & 255] + zi
    B = perm_table[(xi + 1) & 255] + yi
    BA = perm_table[B & 255] + zi
    BB = perm_table[(B + 1) & 255] + zi

    # Gradient contributions
    # Bottom face (z)
    g_aaa = grad(perm_table[AA & 255], xf, yf, zf)
    g_baa = grad(perm_table[BA & 255], xf - 1, yf, zf)
    g_aba = grad(perm_table[AB & 255], xf, yf - 1, zf)
    g_bba = grad(perm_table[BB & 255], xf - 1, yf - 1, zf)

    # Top face (z+1)
    g_aa = grad(perm_table[(AA + 1) & 255], xf, yf, zf - 1)
    g_ba = grad(perm_table[(BA + 1) & 255], xf - 1, yf, zf - 1)
    g_ab = grad(perm_table[(AB + 1) & 255], xf, yf - 1, zf - 1)
    g_bb = grad(perm_table[(BB + 1) & 255], xf - 1, yf - 1, zf - 1)

    # Interpolation
    q_aa = lerp(w, g_aaa, g_aa)
    q_ba = lerp(w, g_baa, g_ba)
    q_ab = lerp(w, g_aba, g_ab)
    q_bb = lerp(w, g_bba, g_bb)

    p_aa = lerp(v, q_aa, q_ab)
    p_ba = lerp(v, q_ba, q_bb)

    return lerp(u, p_aa, p_ba)
```

**Parameters**
- Fade function: Quintic $6t^5 - 15t^4 + 10t^3$ [S1, S3, unofficial].
- Permutation table size: Not given in the sources.

**Edge Cases and Pitfalls**
- **Boundary Precision**: Using floating-point coordinates near integer boundaries can cause discontinuities if the integer part is not computed consistently (e.g., using `floor` vs `int` truncation for negative numbers). Always use `floor` or bitwise AND with mask for positive domains [G].
- **Gradient Selection**: The mapping from hash to gradient must be uniform. The standard method uses `hash & 15` to select from 12 vectors, discarding 4 indices or mapping them to existing vectors [S1, unofficial].

## Subtopics

### Quintic Noise
Noise using quintic interpolation ($6t^5 - 15t^4 + 10t^3$) to achieve $C^2$ continuity, eliminating visible grid artifacts in derivatives compared to Hermite's $C^1$ [S2]. It computes analytical derivatives alongside the value, useful for physics-based volume effects [S2].

### Improved Smoothing Function
The replacement of the original Perlin smoothing function $f(x) = 3x^2 - 2x^3$ with $f(x) = 6x^5 - 15x^4 + 10x^3$. The new function improves the curve of the noise and achieves better visual results [S3, unofficial].

### [3D Perlin Noise](11323-3d-perlin-noise.md)
3D Perlin Noise generates a continuous scalar value for 3D space using gradient vectors at lattice points, interpolated via cubic smoothing functions. Not given in the sources how it compares to sibling methods.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Produces smooth, continuous noise with $C^1$ continuity (or $C^2$ with quintic derivatives). Simple to implement and highly parallelizable [G].
- **Limitations**: Directional artifacts (grid lines) are visible in the output due to the axis-aligned lattice structure. Computational cost is higher than Simplex Noise in higher dimensions due to the exponential growth of cell corners ($2^D$) [S3, unofficial].
- **When to use**: When $C^1$ continuity is sufficient and implementation simplicity is prioritized over artifact reduction. Ideal for 3D procedural textures where directional bias is masked by other scene elements [G].

## Background
The replacement of the original Perlin smoothing function $f(x) = 3x^2 - 2x^3$ with $f(x) = 6x^5 - 15x^4 + 10x^3$ was documented to improve the curve and achieve better visual results [S3, unofficial]. It remains a foundational technique in procedural generation for volume rendering, terrain synthesis, and texture mapping [G].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
