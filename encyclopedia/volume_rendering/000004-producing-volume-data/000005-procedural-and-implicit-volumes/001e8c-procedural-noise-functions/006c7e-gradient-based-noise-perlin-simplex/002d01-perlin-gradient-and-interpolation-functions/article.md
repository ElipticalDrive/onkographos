# Perlin Gradient and Interpolation Functions

Also called: Fade, Grad, Lerp

## Overview
This article defines the core mathematical primitives—gradient selection, fading (smoothing), and linear interpolation—used to construct gradient-based noise functions like Perlin noise [S1, S3, unofficial]. It solves the problem of generating continuous scalar fields from discrete lattice points by applying a quintic fade function to ensure smooth transitions and eliminate grid-aligned artifacts [S1, S3, unofficial]. It differs from sibling methods primarily in its specific choice of the quintic fade polynomial and the 12-gradient vector set for 3D, whereas Simplex noise uses a different lattice structure and interpolation geometry [S1, S3, unofficial].

## How it works
**Inputs:**
- Query point $\mathbf{P} = (x, y, z)$ in continuous space [G].
- A permutation table containing integers to map coordinates to gradient indices [G].

**Outputs:**
- A scalar noise value $n$ [G].

**Data Structures:**
- **Permutation Table:** An array of integers used to map lattice coordinates to gradient indices without modulo operations [G].
- **Gradient Set (3D):** 12 vectors corresponding to the face diagonals of a cube, selected via bitwise operations on the hash index [S1, unofficial].

**Steps:**

1.  **Integer and Fractional Decomposition:** Compute the integer lattice coordinates $\mathbf{I} = (\lfloor x \rfloor, \lfloor y \rfloor, \lfloor z \rfloor)$ and the fractional coordinates $\mathbf{F} = (x - \lfloor x \rfloor, y - \lfloor y \rfloor, z - \lfloor z \rfloor)$ [G].
2.  **Gradient Indexing:** Map the integer coordinates to gradient indices using the permutation table [G].
3.  **Gradient Vector Selection:** Decode the index into a gradient vector using bitwise masking. Let $h = \text{idx} \ \& \ 15$. Select components $u$ and $v$ based on $h$, then apply signs based on the lowest two bits: $\mathbf{g} = ((h \ \& \ 1) == 0 ? u : -u) + ((h \ \& \ 2) == 0 ? v : -v)$ [S1, unofficial].
4.  **Dot Product Contribution:** For each of the 8 corners of the unit cell, compute the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$ and the contribution $n_i = \mathbf{g}_i \cdot \mathbf{d}_i$ [Parent].
5.  **Fade (Smoothing) Function:** Apply the quintic fade function to each fractional coordinate: $f(t) = 6t^5 - 15t^4 + 10t^3$, or equivalently $f(t) = t^3(t(t \cdot 6 - 15) + 10)$ [S1, S3, unofficial]. This replaces the original cubic Hermite function $f(t) = 3t^2 - 2t^3$ to improve curve quality and visual results [S3, unofficial].
6.  **Interpolation:** Combine the 8 contributions using trilinear interpolation weighted by the fade values $u_x = f(f_x)$, $u_y = f(f_y)$, $u_z = f(f_z)$, using $\text{lerp}(a, b, t) = a + (b - a)t$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fade(t):
    """Quintic fade function for smooth interpolation."""
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0)

def lerp(a, b, t):
    """Linear interpolation."""
    return a + (b - a) * t

def grad(hash_val, x, y, z):
    """
    Select gradient vector from hash.
    Uses 12 gradients (face diagonals) for 3D.
    """
    h = hash_val & 15  # Mask to 0-15
    u = x if h < 8 else y
    v = y if h < 4 else (x if h == 12 or h == 14 else z)
    return ((-u) if (h & 1) else u) + ((-v) if (h & 2) else v)

def perlin_gradient_noise(x, y, z, perm_table):
    """
    Compute Perlin noise value at (x, y, z).
    perm_table: list of integers (typically 256 or 512 entries).
    """
    # 1. Integer and fractional parts
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    
    # 2. Fade function
    ux, uy, uz = fade(fx), fade(fy), fade(fz)
    
    # 3. Hash corners (standard 256-entry table masking)
    A  = perm_table[ix & 255] + iy
    AA = perm_table[A & 255] + iz
    AB = perm_table[(A + 1) & 255] + iz
    B  = perm_table[(ix + 1) & 255] + iy
    BA = perm_table[B & 255] + iz
    BB = perm_table[(B + 1) & 255] + iz
    
    # 4. Gradient contributions
    n000 = grad(perm_table[AA & 255], fx, fy, fz)
    n100 = grad(perm_table[BA & 255], fx - 1, fy, fz)
    n010 = grad(perm_table[AB & 255], fx, fy - 1, fz)
    n110 = grad(perm_table[BB & 255], fx - 1, fy - 1, fz)
    n001 = grad(perm_table[(AA + 1) & 255], fx, fy, fz - 1)
    n101 = grad(perm_table[(BA + 1) & 255], fx - 1, fy, fz - 1)
    n011 = grad(perm_table[(AB + 1) & 255], fx, fy - 1, fz - 1)
    n111 = grad(perm_table[(BB + 1) & 255], fx - 1, fy - 1, fz - 1)
    
    # 5. Interpolation
    x1 = lerp(n000, n100, ux)
    x2 = lerp(n010, n110, ux)
    y1 = lerp(x1, x2, uy)
    
    x1 = lerp(n001, n101, ux)
    x2 = lerp(n011, n111, ux)
    y2 = lerp(x1, x2, uy)
    
    return lerp(y1, y2, uz)
```

**Edge Cases and Pitfalls:**
- **Precision:** Large coordinate values may cause floating-point precision errors; lattice wrapping should be handled carefully [G].
- **Gradient Set:** Using fewer than 12 gradients in 3D (e.g., only axis-aligned vectors) leads to visible grid artifacts [S1, unofficial].
- **Derivatives:** For analytical derivatives, the fade derivative is $f'(t) = 30t^2(t-1)^2$ [S2].

## Subtopics

### [3D Perlin Noise](11323-3d-perlin-noise.md)
3D Perlin Noise generates a continuous scalar field in three-dimensional space by interpolating gradient vectors assigned to lattice points, producing seamless, tileable textures when wrapping is enabled. It distinguishes itself from its parent method by explicitly supporting 3D coordinate wrapping via modulo arithmetic and utilizing a fifth-order polynomial fade function to eliminate grid-oriented artifacts and derivative discontinuities.

### [Flow Noise](17017-flow-noise.md)
Flow Noise is a procedural noise function designed to generate time-varying, swirling, and advected textures, primarily for physically-based simulations of fluids and gases. It differs from standard Perlin noise by introducing a time-dependent advection step that displaces the sampling coordinates, creating a sense of continuous flow and swirling motion rather than static, frozen noise patterns.

### [Improved Perlin Noise](17170-improved-perlin-noise.md)
Improved Perlin Noise, introduced by Kensler et al. [KKS08], is a lattice gradient noise function that applies three mutually orthogonal improvements to the original Perlin noise [Per85, Per02] to address axial decorrelation, band-limitation, and surface noise quality. It distinguishes itself from standard Perlin noise by replacing the hash and reconstruction kernels with optimized variants that reduce directional artifacts and provide better spectral control.

### [GPU-Optimized Perlin Noise (BBS)](18928-gpu-optimized-perlin-noise-bbs.md)
GPU-Optimized Perlin Noise (BBS), also known as Modified Noise, is a procedural noise function designed for efficient direct computation on GPUs by eliminating permutation table lookups. It replaces the hash lookup with a computable Blum-Blum-Shub (BBS) generator and selects gradients from hypercube corners, enabling dimension reducibility to optimize 3D noise via 2D texture lookups. It differs from [Improved Perlin Noise](17170-improved-perlin-noise.md) by using corner gradients instead of unit sphere vectors and from [3D Perlin Noise](11323-3d-perlin-noise.md) by its specific GPU-oriented hashing and texture optimization strategies.

### Quintic Noise
Noise using quintic interpolation (`6t^5 - 15t^4 + 10t^3`) to achieve C2 continuity, eliminating visible grid artifacts in derivatives compared to Hermite's C1 [S2].
1. Compute `u = f^3 * (f * (f * 6.0 - 15.0) + 10.0)`.
2. Compute derivative `du = 30.0 * f^2 * (f * (f - 2.0) + 1.0)`.
3. Retrieve hashes `a,b,c,d`.
4. Compute coefficients `k0=a, k1=b-a, k2=c-a, k3=a-b-c+d`.
5. Value: `k0 + k1*u.x + k2*u.y + k3*u.x*u.y`.
6. Derivatives: `du * vec2(k1 + k3*u.y, k2 + k3*u.x)` [S2].

### Improved Smoothing Function
The original smoothing function $f(x) = 3x^2 - 2x^3$ was replaced by $f(x) = 6x^5 - 15x^4 + 10x^3$ to improve the curve of the function and achieve better visual results [S3, unofficial].

## Operating characteristics
- **Scaling:** Simplex noise has lower computational complexity and scales to higher dimensions (4D, 5D) with much less computational cost than Perlin noise [S3, unofficial].
- **Strengths:** The quintic fade function improves curve quality and visual results over the original cubic Hermite interpolation [S3, unofficial].
- **Limitations:** Computational cost is higher than Simplex noise in higher dimensions [S3, unofficial].
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** For high-dimensional noise (4D+) where Simplex noise scales better [S3, unofficial].

## Background
Perlin noise was introduced by Ken Perlin in 1985 [S3, unofficial]. The original version used a cubic Hermite interpolation with a simpler fade function. In 2002, Perlin published an "improved noise" algorithm that replaced the smoothing function with the quintic polynomial $6t^5 - 15t^4 + 10x^3$ to improve visual quality and continuity [S3, unofficial].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S4] Procedural Multiplicative Perlin Noise Model for Dust Cloud Generation, https://inria.hal.science/hal-05627012v1, fetched 2026-10-26 (unofficial)
