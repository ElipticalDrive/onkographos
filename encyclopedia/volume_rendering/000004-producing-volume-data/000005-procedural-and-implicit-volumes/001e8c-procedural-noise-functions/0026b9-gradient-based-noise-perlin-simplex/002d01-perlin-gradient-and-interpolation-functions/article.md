# Perlin Gradient and Interpolation Functions

Also called: Fade, Grad, Lerp

## Overview
This article defines the core mathematical primitives—gradient selection, fading, and interpolation—used to construct Perlin noise. It solves the problem of generating continuous, differentiable scalar fields from discrete lattice points, which is essential for procedural volume data. It differs from [Simplex Noise](978-simplex-noise.md) by relying on an axis-aligned cubic lattice and standard trilinear interpolation of gradient dot products, whereas Simplex Noise uses a simplex grid to reduce directional artifacts.

## How it works
The method combines three components: a gradient vector generator, a smoothing (fade) function, and linear interpolation.

**Inputs and Outputs**
- Input: Query point $\mathbf{P} = (x, y, z)$, a permutation table $P$ of size 512 (or 256 repeated), and gradient vectors $\mathbf{g}$.
- Output: Scalar noise value $n \in [-1, 1]$ (normalized) or unnormalized dot product sum.

**Data Structures**
- **Permutation Table**: An array of 256 integers (typically $0..255$), duplicated to 512 to avoid modulo operations during indexing.
- **Gradient Set (3D)**: 12 vectors representing the face diagonals of a cube: $(\pm 1, \pm 1, 0), (\pm 1, 0, \pm 1), (0, \pm 1, \pm 1)$ [S1, unofficial].

**Steps**
1. **Integer and Fractional Parts**: Compute integer coordinates $\mathbf{I} = \lfloor \mathbf{P} \rfloor$ and fractional coordinates $\mathbf{F} = \mathbf{P} - \mathbf{I}$.
2. **Gradient Selection**: For each of the 8 corners of the unit cell containing $\mathbf{P}$, compute a hash index using the permutation table. Map this index to one of the 12 gradient vectors.
   - The hash is computed as $h = P[I_x + (P[I_y + P[I_z]])]$.
   - The gradient vector $\mathbf{g}$ is selected based on $h \pmod{12}$ [S1, unofficial].
3. **Dot Product Contribution**: For each corner $i$, compute the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$ (where $\mathbf{V}_i$ is the corner coordinate). Compute the contribution $n_i = \mathbf{g}_i \cdot \mathbf{d}_i$.
4. **Smoothing (Fade)**: Apply the fade function $f(t)$ to each component of $\mathbf{F}$ to obtain smooth interpolation weights $u_x, u_y, u_z$.
   - Formula: $f(t) = 6t^5 - 15t^4 + 10t^3$ [S3, unofficial].
5. **Interpolation**: Combine the 8 contributions using trilinear interpolation weighted by $u_x, u_y, u_z$.
   - First, interpolate along the x-axis for each y-z pair.
   - Then interpolate along the y-axis.
   - Finally, interpolate along the z-axis.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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
    h = hash_val & 15  # Keep within 0-15
    u = x if h < 8 else y
    v = y if h < 4 else (x if h == 12 or h == 14 else z)
    
    # Apply sign based on the least significant bits
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
    
    # Integer part
    xi = int(x) & 255
    yi = int(y) & 255
    zi = int(z) & 255
    
    # Fractional part
    xf = x - int(x)
    yf = y - int(y)
    zf = z - int(z)
    
    # Fade curves
    u = fade(xf)
    v = fade(yf)
    w = fade(zf)
    
    # Hashing corners
    A = perm_table[xi] + yi
    AA = perm_table[A] + zi
    AB = perm_table[A + 1] + zi
    B = perm_table[xi + 1] + yi
    BA = perm_table[B] + zi
    BB = perm_table[B + 1] + zi
    
    # Gradient contributions
    # Top face (z+1)
    g_aa = grad(perm_table[AA], xf, yf, zf - 1)
    g_ba = grad(perm_table[BA], xf - 1, yf, zf - 1)
    g_ab = grad(perm_table[AB], xf, yf - 1, zf - 1)
    g_bb = grad(perm_table[BB], xf - 1, yf - 1, zf - 1)
    
    # Bottom face (z)
    g_aaa = grad(perm_table[AA], xf, yf, zf)
    g_baa = grad(perm_table[BA], xf - 1, yf, zf)
    g_aba = grad(perm_table[AB], xf, yf - 1, zf)
    g_bba = grad(perm_table[BB], xf - 1, yf - 1, zf)
    
    # Interpolation
    # Along Z
    q_aa = lerp(w, g_aaa, g_aa)
    q_ba = lerp(w, g_baa, g_ba)
    q_ab = lerp(w, g_aba, g_ab)
    q_bb = lerp(w, g_bba, g_bb)
    
    # Along Y
    p_aa = lerp(v, q_aa, q_ab)
    p_ba = lerp(v, q_ba, q_bb)
    
    # Along X
    return lerp(u, p_aa, p_ba)
```

**Parameters**
- Permutation table size: 256 (standard) [S1, unofficial].
- Fade function: Quintic $6t^5 - 15t^4 + 10t^3$ [S3, unofficial].

**Edge Cases and Pitfalls**
- **Boundary Precision**: Using floating-point coordinates near integer boundaries can cause discontinuities if the integer part is not computed consistently (e.g., using `floor` vs `int` truncation for negative numbers). Always use `floor` or bitwise AND with mask for positive domains [G].
- **Gradient Selection**: The mapping from hash to gradient must be uniform. The standard method uses `hash & 15` to select from 12 vectors, discarding 4 indices or mapping them to existing vectors [S1, unofficial].

## Subtopics

### Quintic Noise
Noise using quintic interpolation ($6t^5 - 15t^4 + 10t^3$) to achieve $C^2$ continuity, eliminating visible grid artifacts in derivatives compared to Hermite's $C^1$ [S2]. It computes analytical derivatives alongside the value, useful for physics-based volume effects [S2].

### Improved Smoothing Function
The replacement of the original Perlin smoothing function $f(x) = 3x^2 - 2x^3$ with $f(x) = 6x^5 - 15x^4 + 10x^3$. The new function improves the curve of the noise and achieves better visual results by providing smoother transitions between lattice points [S3, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per sample, constant number of operations (8 gradient lookups, 12 lerp operations, 3 fade evaluations).
- **Memory Complexity**: $O(1)$ auxiliary space, plus $O(N)$ for the permutation table (typically 1KB).
- **Strengths**: Produces smooth, continuous noise with $C^1$ continuity (or $C^2$ with quintic derivatives). Simple to implement and highly parallelizable.
- **Limitations**: Directional artifacts (grid lines) are visible in the output due to the axis-aligned lattice structure. Computational cost is higher than Simplex Noise in higher dimensions due to the exponential growth of cell corners ($2^D$) [S3, unofficial].
- **When to use**: When $C^1$ continuity is sufficient and implementation simplicity is prioritized over artifact reduction. Ideal for 3D procedural textures where directional bias is masked by other scene elements.

## Background
Perlin noise was introduced by Ken Perlin in 1983 for the movie *Tron* [S3, unofficial]. The original smoothing function was $3x^2 - 2x^3$. Perlin later improved the noise in 2002 by introducing the quintic fade function and a better gradient selection method to reduce visual artifacts [S3, unofficial]. It remains a foundational technique in procedural generation for volume rendering, terrain synthesis, and texture mapping.

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
