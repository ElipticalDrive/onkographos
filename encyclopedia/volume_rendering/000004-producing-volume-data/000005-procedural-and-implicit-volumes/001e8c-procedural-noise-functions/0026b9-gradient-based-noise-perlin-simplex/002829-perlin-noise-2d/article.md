# Perlin Noise 2D

Also called: 2D Perlin Noise, RawPerlin2D

## Overview
Perlin Noise 2D is a gradient noise function that generates smooth, continuous random values by interpolating dot products of gradient vectors and distance vectors at the corners of the unit grid cell containing the query point [S1, unofficial]. It solves the problem of discontinuous or visually artifact-prone noise by ensuring $C^1$ continuity across grid boundaries through a quintic fade function [S1, unofficial]. It sets itself apart from its siblings, such as Simplex Noise, by operating on a square grid topology rather than a simplex lattice, and from 3D variants by restricting the domain to two dimensions [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Two floating-point coordinates $(X, Y)$ representing the query point in 2D space; an integer `Seed` for initializing the permutation table.
- **Output**: A single floating-point noise value $N(X, Y)$, typically in the range $[-1, 1]$ or $[0, 1]$ depending on normalization (standard implementation returns unnormalized dot product sums).

**Data Structures**
- **Permutation Table**: An array of 512 unsigned 8-bit integers (`uint8[512]`). It consists of two copies of a shuffled permutation of integers $0$ to $255$. This doubling avoids modulo operations when accessing indices near the boundary of the $0-255$ range [S1, unofficial].

**Algorithm Steps**
1.  **Initialize Permutation Table**: If not already cached for the given `Seed`, generate a permutation table `P` of size 512.
    - Create an array `p` of size 256 containing integers $0$ to $255$.
    - Use a seeded random number generator to shuffle `p` (e.g., Fisher-Yates shuffle).
    - Copy `p` into `P[0...255]` and `P[256...511]`.
2.  **Grid Coordinates**: Compute the integer coordinates of the cell containing $(X, Y)$:
    $$ XI = \lfloor X \rfloor \pmod{256} $$
    $$ YI = \lfloor Y \rfloor \pmod{256} $$
    *Note: In implementation, this is often done via bitwise AND (`& 255`) if coordinates are non-negative, or by adding the doubled table offset.*
3.  **Fractional Coordinates**: Compute the position of the point within the cell:
    $$ XF = X - \lfloor X \rfloor $$
    $$ YF = Y - \lfloor Y \rfloor $$
4.  **Fade Function**: Apply the quintic fade function to smooth the interpolation weights:
    $$ U = 6XF^5 - 15XF^4 + 10XF^3 $$
    $$ V = 6YF^5 - 15YF^4 + 10YF^3 $$
5.  **Gradient Index Lookup**: Compute the indices for the four corners of the cell using the permutation table. The indices map to gradient vectors:
    $$ AA = P[P[XI] + YI] $$
    $$ AB = P[P[XI] + YI + 1] $$
    $$ BA = P[P[XI + 1] + YI] $$
    $$ BB = P[P[XI + 1] + YI + 1] $$
    *Note: $XI+1$ and $YI+1$ are handled by the doubled table size (512), so no modulo is needed for the lookup.*
6.  **Gradient Dot Products**: Calculate the contribution from each corner. The gradient vector $\mathbf{g}$ is determined by the low 3 bits of the hash index. The contribution is $\mathbf{g} \cdot \mathbf{d}$, where $\mathbf{d}$ is the vector from the corner to the point $(X, Y)$:
    - Corner $(0,0)$ (index $AA$): $\mathbf{d} = (XF, YF)$. Contribution $n_{AA} = \text{Grad}(AA, XF, YF)$.
    - Corner $(1,0)$ (index $BA$): $\mathbf{d} = (XF-1, YF)$. Contribution $n_{BA} = \text{Grad}(BA, XF-1, YF)$.
    - Corner $(0,1)$ (index $AB$): $\mathbf{d} = (XF, YF-1)$. Contribution $n_{AB} = \text{Grad}(AB, XF, YF-1)$.
    - Corner $(1,1)$ (index $BB$): $\mathbf{d} = (XF-1, YF-1)$. Contribution $n_{BB} = \text{Grad}(BB, XF-1, YF-1)$.
7.  **Interpolation**:
    - Interpolate along the X-axis for the top and bottom edges:
      $$ L_1 = \text{Lerp}(U, n_{AA}, n_{BA}) $$
      $$ L_2 = \text{Lerp}(U, n_{AB}, n_{BB}) $$
    - Interpolate along the Y-axis:
      $$ N(X, Y) = \text{Lerp}(V, L_1, L_2) $$

**Formulas**
- **Fade**: $f(t) = 6t^5 - 15t^4 + 10t^3$ [S1, unofficial].
- **Lerp**: $\text{Lerp}(t, a, b) = a + t(b - a)$ [S1, unofficial].
- **Grad**: The function $\text{Grad}(hash, x, y)$ selects a gradient direction based on `hash & 7`:
  - $0$: $x + y$
  - $1$: $x - y$
  - $2$: $-x + y$
  - $3$: $-x - y$
  - $4$: $x$
  - $5$: $-x$
  - $6$: $y$
  - $7$: $-y$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def perlin2d(x: float, y: float, seed: int) -> float:
    # 1. Initialize Permutation Table if needed
    # In a real implementation, this would be a class member or global cache.
    # Here we assume a function get_permutation_table(seed) returns uint8[512].
    p = get_permutation_table(seed)
    
    # 2. Grid Coordinates
    xi = int(x) & 255
    yi = int(y) & 255
    
    # 3. Fractional Coordinates
    xf = x - int(x)
    yf = y - int(y)
    
    # 4. Fade Function
    u = fade(xf)
    v = fade(yf)
    
    # 5. Gradient Index Lookup
    aa = p[p[xi] + yi]
    ab = p[p[xi] + yi + 1]
    ba = p[p[xi + 1] + yi]
    bb = p[p[xi + 1] + yi + 1]
    
    # 6. Gradient Dot Products
    g_aa = grad(aa, xf, yf)
    g_ba = grad(ba, xf - 1.0, yf)
    g_ab = grad(ab, xf, yf - 1.0)
    g_bb = grad(bb, xf - 1.0, yf - 1.0)
    
    # 7. Interpolation
    x_lerp_1 = lerp(u, g_aa, g_ba)
    x_lerp_2 = lerp(u, g_ab, g_bb)
    result = lerp(v, x_lerp_1, x_lerp_2)
    
    return result

def fade(t: float) -> float:
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0)

def lerp(t: float, a: float, b: float) -> float:
    return a + t * (b - a)

def grad(hash_val: int, x: float, y: float) -> float:
    h = hash_val & 7
    u = x if h < 4 else y
    v = y if h < 4 else x
    return (u if (h & 1) == 0 else -u) + (v if (h & 2) == 0 else -v)
```

**Parameters**
- **Seed**: Integer used to initialize the permutation table. Different seeds produce different noise patterns. Typical values are any 32-bit integer.

**Edge Cases and Pitfalls**
- **Boundary Conditions**: The permutation table must be doubled to 512 entries to handle cases where `XI + 1` or `YI + 1` exceeds 255. Without this, indices would wrap incorrectly or require expensive modulo operations [S1, unofficial].
- **Precision**: Using `int(x)` for floor is standard for positive coordinates. For negative coordinates, `floor()` must be used explicitly, as truncation towards zero differs from floor [G].
- **Gradient Selection**: The specific 8-gradient set used here is a common simplification for 2D. It provides good visual quality but is not the only valid set [S1, unofficial].

## Subtopics

### [Fractal Brownian Motion (fBm) with Perlin Noise](4253-fractal-brownian-motion-fbm-with-perlin-noise.md)
Fractal Brownian Motion (fBm) with Perlin Noise is a procedural technique that generates rougher, more natural-looking noise by summing multiple octaves of a base gradient noise function, specifically Perlin Noise 2D, with decreasing amplitude and increasing frequency. It solves the problem of simple noise appearing too synthetic or lacking fine detail by introducing self-similar detail at multiple scales, mimicking natural phenomena like terrain or clouds. It sets itself apart from standard Perlin noise by producing a hierarchical structure of detail, and from other fBm variants (like those using Worley noise) by retaining the connectedness and billowiness characteristic of gradient-based interpolation.

## Operating characteristics
- **Time Complexity**: $O(1)$ per query. The algorithm involves a fixed number of arithmetic operations and table lookups, independent of the resolution or scene complexity.
- **Memory Complexity**: $O(1)$ auxiliary space for the permutation table (512 bytes).
- **Measured Performance**: On modern CPUs, a single 2D Perlin noise evaluation takes approximately 10-20 nanoseconds, depending on cache hits for the permutation table [S1, unofficial].
- **Scaling**: Performance is constant regardless of the coordinate magnitude, as the modulo operation (or bitwise AND) keeps indices within bounds.
- **Strengths**: Simple to implement; provides smooth, continuous gradients; widely supported and understood.
- **Limitations**: The square grid topology can introduce directional artifacts (anisotropy) along the axes, which is less pronounced in Simplex Noise [S1, unofficial]. The 8-gradient set is a simplification and may not provide the highest possible quality compared to larger gradient sets.

## Background
Perlin Noise was introduced by Ken Perlin in 1985 for the movie *Tron* [S1, unofficial]. The 2D variant described here is the original implementation, often referred to as "Raw Perlin" or "Improved Perlin" (referring to the 1985 improved version with the fade function, distinct from the 2002 "Improved Perlin" which addressed grid alignment artifacts). It is a foundational technique in procedural texture generation and is widely used in game development, particularly for terrain heightmaps and cloud simulations [S1, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
