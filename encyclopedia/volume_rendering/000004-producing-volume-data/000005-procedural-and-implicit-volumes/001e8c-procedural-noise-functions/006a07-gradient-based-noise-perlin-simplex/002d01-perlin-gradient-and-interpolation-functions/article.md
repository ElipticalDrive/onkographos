# Perlin Gradient and Interpolation Functions

Also called: Fade, Grad, Lerp

## Overview
This article defines the core mathematical primitives—gradient selection, fading (smoothing), and linear interpolation—used to construct gradient-based noise functions like Perlin noise. It solves the problem of generating continuous, differentiable scalar fields from discrete lattice points by ensuring $C^1$ continuity (smooth first derivatives) to eliminate grid-aligned artifacts. It differs from sibling methods primarily in its specific choice of the quintic fade polynomial and the 12-gradient vector set for 3D, whereas Simplex noise uses a different lattice structure and interpolation geometry [S1, S3, unofficial].

## How it works
**Inputs:**
- Query point $\mathbf{P} = (x, y, z)$ in continuous space.
- A permutation table $P$ of size 512 (or 256 repeated) containing integers $0..255$ to map coordinates to gradient indices [S1, S3, unofficial].

**Outputs:**
- A scalar noise value $n \in [-1, 1]$ (normalized).
- Optionally, the gradient vector $\mathbf{g}$ and distance vector $\mathbf{d}$ for derivative calculation.

**Data Structures:**
- **Permutation Table:** An array of 256 (or 512) integers. Used to avoid modulo operations and ensure uniform distribution of gradient indices [4211-perlin-noise-2d.md].
- **Gradient Set (3D):** 12 vectors corresponding to the face diagonals of a cube: $(\pm 1, \pm 1, 0)$ and permutations thereof [S1, unofficial].

**Steps:**

1.  **Integer and Fractional Decomposition:**
    Compute the integer lattice coordinates $\mathbf{I} = (\lfloor x \rfloor, \lfloor y \rfloor, \lfloor z \rfloor)$ and the fractional coordinates $\mathbf{F} = (x - \lfloor x \rfloor, y - \lfloor y \rfloor, z - \lfloor z \rfloor)$. Let $f_x, f_y, f_z$ be the components of $\mathbf{F}$ [S1, unofficial].

2.  **Gradient Indexing:**
    Map the integer coordinates to gradient indices using the permutation table. For a corner at $(i, j, k)$, the index is computed as:
    $$ \text{idx} = P[P[P[i] + j] + k] \pmod{12} $$
    (Assuming a 3D implementation with 12 gradients) [S1, unofficial].

3.  **Gradient Vector Selection:**
    Decode the index into a gradient vector $\mathbf{g} = (g_x, g_y, g_z)$. The standard 3D set uses bits of the index to select axes and signs [S1, unofficial]:
    - Let $h = \text{idx} \ \& \ 15$ (using a 4-bit mask for the 12 gradients).
    - $u = (h < 8) ? x : y$
    - $v = (h < 4) ? y : (h == 12 \lor h == 14) ? x : z$
    - $\mathbf{g} = ((h \ \& \ 1) == 0 ? u : -u) + ((h \ \& \ 2) == 0 ? v : -v)$
    This yields one of the 12 face diagonals [S1, unofficial].

4.  **Dot Product Contribution:**
    For each of the 8 corners of the unit cell containing $\mathbf{P}$, compute the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$ and the contribution:
    $$ n_i = \mathbf{g}_i \cdot \mathbf{d}_i $$
    [Parent]

5.  **Fade (Smoothing) Function:**
    Apply the quintic fade function to each fractional coordinate to ensure $C^1$ continuity. The standard improved fade function is [S1, S3, unofficial]:
    $$ f(t) = 6t^5 - 15t^4 + 10t^3 $$
    Alternatively written as:
    $$ f(t) = t^3 (t (t \cdot 6 - 15) + 10) $$
    [S1, unofficial]
    *Note:* The original Perlin noise used $f(t) = 3t^2 - 2t^3$, but the quintic version is preferred for better visual results and smoother derivatives [S3, unofficial].

6.  **Interpolation:**
    Combine the 8 contributions using trilinear interpolation weighted by the fade values $u_x = f(f_x)$, $u_y = f(f_y)$, $u_z = f(f_z)$ [1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md]:
    $$ n = \text{lerp}(\text{lerp}(\text{lerp}(n_{000}, n_{100}, u_x), \text{lerp}(n_{010}, n_{110}, u_x), u_y), \text{lerp}(\text{lerp}(n_{001}, n_{101}, u_x), \text{lerp}(n_{011}, n_{111}, u_x), u_x), u_z) $$
    Where $\text{lerp}(a, b, t) = a + (b - a)t$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fade(t):
    """Quintic fade function for C1 continuity."""
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
    return ((-u) if (h & 1) != 0 else u) + ((-v) if (h & 2) != 0 else v)

def perlin_gradient_noise(x, y, z, perm_table):
    """
    Compute Perlin noise value at (x, y, z).
    perm_table: list of 256 or 512 integers.
    """
    # 1. Integer and fractional parts
    X = int(x) & 255
    Y = int(y) & 255
    Z = int(z) & 255
    
    x -= int(x)
    y -= int(y)
    z -= int(z)
    
    # 2. Fade function
    u = fade(x)
    v = fade(y)
    w = fade(z)
    
    # 3. Hashing corners
    A = perm_table[X] + Y
    AA = perm_table[A] + Z
    AB = perm_table[A + 1] + Z
    B = perm_table[X + 1] + Y
    BA = perm_table[B] + Z
    BB = perm_table[B + 1] + Z
    
    # 4. Gradient contributions
    # Corner 000
    g000 = grad(perm_table[AA], x, y, z)
    # Corner 100
    g100 = grad(perm_table[BA], x - 1, y, z)
    # Corner 010
    g010 = grad(perm_table[AB], x, y - 1, z)
    # Corner 110
    g110 = grad(perm_table[BB], x - 1, y - 1, z)
    # Corner 001
    g001 = grad(perm_table[AA + 1], x, y, z - 1)
    # Corner 101
    g101 = grad(perm_table[BA + 1], x - 1, y, z - 1)
    # Corner 011
    g011 = grad(perm_table[AB + 1], x, y - 1, z - 1)
    # Corner 111
    g111 = grad(perm_table[BB + 1], x - 1, y - 1, z - 1)
    
    # 5. Interpolation
    x1 = lerp(g000, g100, u)
    x2 = lerp(g010, g110, u)
    y1 = lerp(x1, x2, v)
    
    x1 = lerp(g001, g101, u)
    x2 = lerp(g011, g111, u)
    y2 = lerp(x1, x2, v)
    
    return lerp(y1, y2, w)
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision errors in large coordinate values can cause lattice wrapping issues. Use modulo arithmetic on integer parts only [S1, unofficial].
- **Boundary Conditions:** The fade function $f(t)$ must satisfy $f(0)=0, f(1)=1, f'(0)=0, f'(1)=0$ for $C^1$ continuity. The quintic polynomial satisfies this [S3, unofficial].
- **Gradient Set:** Using only 4 gradients in 3D (axis-aligned) leads to visible grid artifacts. The 12-gradient set is standard for Perlin noise [S1, unofficial].

## Subtopics

### [3D Perlin Noise](11323-3d-perlin-noise.md)
3D Perlin Noise generates a continuous scalar field in three-dimensional space by interpolating gradient vectors assigned to lattice points, producing seamless, tileable textures when wrapping is enabled. It distinguishes itself from its parent method by explicitly supporting 3D coordinate wrapping via modulo arithmetic and utilizing a fifth-order polynomial fade function to eliminate grid-oriented artifacts and derivative discontinuities.

### [Flow Noise](17017-flow-noise.md)
Flow Noise is a procedural noise function designed to generate time-varying, swirling, and advected textures, primarily for physically-based simulations of fluids and gases. It differs from standard Perlin noise by introducing a time-dependent advection step that displaces the sampling coordinates, creating a sense of continuous flow and swirling motion rather than static, frozen noise patterns.

### [Improved Perlin Noise](17170-improved-perlin-noise.md)
Improved Perlin Noise, introduced by Kensler et al. [KKS08], is a lattice gradient noise function that applies three mutually orthogonal improvements to the original Perlin noise [Per85, Per02] to address axial decorrelation, band-limitation, and surface noise quality. It distinguishes itself from standard Perlin noise by replacing the hash and reconstruction kernels with optimized variants that reduce visual artifacts along axes and provide better spectral control.

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
- **Time Complexity:** $O(1)$ per sample, with a constant number of arithmetic operations (approx. 30-40 flops for 3D) [S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, plus $O(N)$ for the permutation table (typically 256 or 512 integers) [S1, unofficial].
- **Strengths:** $C^1$ continuity ensures smooth gradients; widely supported and understood; good for terrain and cloud textures [S3, unofficial].
- **Limitations:** Grid-aligned artifacts (though reduced by the quintic fade) are still present compared to Simplex noise; computational cost is higher than Simplex noise in higher dimensions [S3, unofficial].
- **When to use:** When $C^1$ continuity is required and the scene is 3D or lower; when compatibility with existing Perlin-based pipelines is needed [S3, unofficial].
- **When not to use:** For high-dimensional noise (4D+) where Simplex noise scales better; when axial artifacts are unacceptable without further post-processing [S3, unofficial].

## Background
Perlin noise was introduced by Ken Perlin in 1985 for the movie *Tron* [Per85]. The original 1985 version used a cubic Hermite interpolation with a simpler fade function. In 2002, Perlin published an "improved noise" algorithm that replaced the smoothing function with the quintic polynomial $6t^5 - 15t^4 + 10t^3$ to improve visual quality and continuity [S3, Per02, unofficial]. This variant became the standard for procedural texture generation in computer graphics [S3, unofficial].

## Sources
[S1] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
