# Perlin Gradient and Interpolation Functions

Also called: Fade, Grad, Lerp

## Overview
This article defines the core mathematical primitives—gradient selection, fading, and linear interpolation—used to construct Perlin noise. It solves the problem of generating smooth, continuous scalar fields from discrete lattice points by ensuring $C^1$ continuity (continuous first derivatives) across cell boundaries. It differs from [Simplex Noise](978-simplex-noise.md) by using a hypercubic lattice and standard linear interpolation, and from [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md) by focusing on the fundamental 3D/2D building blocks rather than higher-dimensional optimizations.

## How it works
**Inputs:**
- Query point $\mathbf{P} = (x, y, z)$ (floating point).
- Permutation table $P$ of size 512 (or 256 repeated), containing integers $0..255$.
- Gradient set $\mathcal{G}$ (typically 12 vectors for 3D).

**Outputs:**
- Scalar noise value $n \in [-1, 1]$ (approximate range depending on gradient normalization).

**Data Structures:**
- `perm`: Array of 512 `uint8_t` or `int`. `perm[i] = i % 256` shuffled.
- `grads`: Array of 12 `vec3` or `float[12][3]`.

**Steps:**

1.  **Integer and Fractional Parts:**
    Compute the integer lattice coordinates $\mathbf{I} = (\lfloor x \rfloor, \lfloor y \rfloor, \lfloor z \rfloor)$ and the fractional coordinates $\mathbf{F} = (x - \lfloor x \rfloor, y - \lfloor y \rfloor, z - \lfloor z \rfloor)$.

2.  **Hashing:**
    Map lattice corners to gradient indices. For a corner $(X, Y, Z)$, compute:
    $$
    \begin{aligned}
    A &= P[X] + Y \\
    AA &= P[A] + Z \\
    AB &= P[A + 1] + Z \\
    B &= P[X + 1] + Y \\
    BA &= P[B] + Z \\
    BB &= P[B + 1] + Z
    \end{aligned}
    $$
    The 8 gradient indices are $AA, BA, BB, AB, \dots$ (specifically $AA, BA, BB, AB, A+B+Z, \dots$). In 2D, indices are $P[P[X]+Y]$ and $P[P[X+1]+Y]$.

3.  **Gradient Selection (`Grad`):**
    For each index $h$, select a gradient vector $\mathbf{g}$ from the predefined set. In the standard 3D implementation [S1, unofficial], the gradient is selected using `h & 15` (modulo 16) to pick one of 12 directions. The mapping [S1, unofficial] is:
    ```python
    def grad(hash_val, x, y, z):
        h = hash_val & 15
        u = x if h < 8 else y
        v = y if h < 4 else (x if h == 12 or h == 14 else z)
        return (u if (h & 1) == 0 else -u) + (v if (h & 2) == 0 else -v)
    ```
    This computes the dot product $\mathbf{g} \cdot \mathbf{d}$ directly, where $\mathbf{d} = \mathbf{F} - \mathbf{0}$ (distance from lattice corner to query point). Note: Some implementations normalize gradients to unit length; others do not. The source [S1, unofficial] does not normalize.

4.  **Fade Function (`Fade`):**
    Apply the quintic smoothing function to each component of $\mathbf{F}$ to ensure $C^1$ continuity [S3, unofficial].
    $$
    f(t) = 6t^5 - 15t^4 + 10t^3
    $$
    Or in Horner's form [S1, unofficial]:
    $$
    f(t) = t \cdot t \cdot t \cdot (t \cdot (t \cdot 6 - 15) + 10)
    $$
    Let $\mathbf{u} = (f(F_x), f(F_y), f(F_z))$.

5.  **Interpolation (`Lerp`):**
    Combine the dot product contributions from the 8 corners using trilinear interpolation weighted by $\mathbf{u}$ [S1, unofficial].
    $$
    \text{lerp}(a, b, t) = a + (b - a)t
    $$
    The final noise value is:
    $$
    \begin{aligned}
    n &= \text{lerp}(
        \text{lerp}(
            \text{lerp}(grad(idx_{000}, F_x, F_y, F_z), grad(idx_{100}, F_x-1, F_y, F_z), u_x),
            \text{lerp}(
                grad(idx_{010}, F_x, F_y-1, F_z),
                grad(idx_{110}, F_x-1, F_y-1, F_z),
                u_x
            ),
            u_y
        ),
        \text{lerp}(
            \text{lerp}(
                grad(idx_{001}, F_x, F_y, F_z-1),
                grad(idx_{101}, F_x-1, F_y, F_z-1),
                u_x
            ),
            \text{lerp}(
                grad(idx_{011}, F_x, F_y-1, F_z-1),
                grad(idx_{111}, F_x-1, F_y-1, F_z-1),
                u_x
            ),
            u_y
        ),
        u_z
    )
    \end{aligned}
    $$

**Parameters:**
- Permutation table size: 512 (to avoid boundary wrap issues with simple modulo).
- Gradient set: 12 vectors (face diagonals of a cube).

**Edge Cases:**
- **Precision:** Floating point errors in `fract()` or large coordinates can cause discontinuities. Use `modf` or `floor` carefully.
- **Boundaries:** The permutation table must be large enough to handle coordinate wrapping if periodicity is required.

## Subtopics

### Quintic Noise
Uses the quintic interpolation function $6t^5 - 15t^4 + 10t^3$ to achieve $C^2$ continuity (continuous second derivatives), eliminating visible grid artifacts in derivatives compared to the standard $C^1$ Hermite interpolation [S2]. It requires computing the analytical derivative of the fade function: $du = 30f^2(f(f-2)+1)$ [S2]. The value is computed as $k_0 + k_1 u_x + k_2 u_y + k_3 u_x u_y$ in 2D, with derivatives scaled by $du$ [S2].

### Improved Smoothing Function
Replaces the original Perlin smoothing function $f(x) = 3x^2 - 2x^3$ with the quintic $f(x) = 6x^5 - 15x^4 + 10x^3$ [S3, unofficial]. This change improves the curve of the function and yields better visual results by providing smoother transitions [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. Requires 8 gradient lookups, 8 dot products, and 7 linear interpolations (or equivalent polynomial evaluations).
- **Memory Complexity:** $O(1)$ auxiliary space, plus the permutation table (256 or 512 bytes).
- **Strengths:** Simple to implement; widely supported; good visual quality for procedural textures.
- **Limitations:** Grid-aligned artifacts (shearing) are visible in derivatives; computational cost grows exponentially with dimensions ($2^D$ corners).
- **When to use:** Standard procedural noise for volumes, textures, and terrain where $C^1$ continuity is sufficient.
- **When not to use:** When higher dimensional scalability is needed (use [Simplex Noise](978-simplex-noise.md)) or when $C^2$ continuity is required for smooth shading derivatives (use [Quintic Noise](#quintic-noise)).

## Background
Ken Perlin introduced the noise function in 1983 for the movie *Tron* [S3, unofficial]. The original implementation used cubic Hermite interpolation ($3x^2 - 2x^3$). The "Improved Perlin Noise" using the quintic fade function was published by Perlin in 2002 to address visual artifacts in the original [S3, unofficial]. The gradient selection method using `hash & 15` is a common implementation detail found in many unofficial sources and tutorials [S1, unofficial].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
