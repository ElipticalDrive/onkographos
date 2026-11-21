# GPU-Optimized Perlin Noise (BBS)

Also called: Modified Noise, Blum-Blum-Shub Perlin Noise

## Overview
GPU-Optimized Perlin Noise (BBS), also known as Modified Noise, is a procedural noise function designed for efficient direct computation on GPUs by eliminating permutation table lookups [S1, unofficial]. It replaces the hash lookup with a computable Blum-Blum-Shub (BBS) generator and selects gradients from hypercube corners, enabling dimension reducibility to optimize 3D noise via 2D texture lookups [S1, unofficial]. It differs from [Improved Perlin Noise](17170-improved-perlin-noise.md) by using corner gradients instead of unit sphere vectors and from [3D Perlin Noise](11323-3d-perlin-noise.md) by its specific GPU-oriented hashing and texture optimization strategies [S1, unofficial].

## How it works
**Inputs:** A 3D point $p = (p_x, p_y, p_z)$.
**Outputs:** A scalar noise value $N(p) \in [-1, 1]$.

**Data Structures:**
*   **Modulus $M$:** A composite number $M = pq$ where $p$ and $q$ are large primes [S1, unofficial].
*   **Textures (Optional Optimization):**
    *   `tex_2d`: Stores 2D noise values $N_{2D}(x, y)$.
    *   `tex_zgrad`: Stores z-gradient values $\frac{\partial N}{\partial z}$ at integer $z$ slices.
    *   Alternatively, a single 2-channel texture storing both.

**Parameters:**
*   $M$: Modulus for BBS generator. Not given in the sources.
*   Precision mode: "Exact" (5-instruction) or "Inexact" (4-instruction) modulo.

**Steps:**

1.  **Integer and Fractional Parts:**
    Compute integer coordinates $i = \lfloor p \rfloor$ and fractional coordinates $f = p - i$.
    Let $i_x, i_y, i_z$ be the integer components and $f_x, f_y, f_z$ be the fractional components.

2.  **Hash Construction:**
    Define the hash function using the Blum-Blum-Shub generator:
    $$ \text{hash}(x) = x^2 \bmod M $$
    Construct multidimensional hashes for lattice corners. For a 3D point, the hash for a corner $(x, y, z)$ is constructed recursively:
    $$ \text{hash3}(x, y, z) = \text{hash}(\text{hash}(\text{hash}(x+1) + y+1) + z+1) $$
    Note: The source uses $p_i + 1$ in the hash construction, implying the hash is computed on the integer lattice indices offset by 1 [S1, unofficial].

3.  **Gradient Selection:**
    Gradients are selected from the 8 corners of the unit hypercube surrounding $p$. For a corner defined by bits $j, k, l \in \{0, 1\}$, the gradient vector $g_{jkl}$ has components $\pm 1$:
    $$ g_{jkl} = [(-1)^j, (-1)^k, (-1)^l] $$
    The dot product with the fractional offset $f = p - i$ simplifies to additions and subtractions:
    $$ \text{grad3D}(f) = \pm f_x \pm f_y \pm f_z $$
    The signs are determined by the corner's bit indices [S1, unofficial].

4.  **Dimension Reducibility Optimization (3D):**
    To reduce texture lookups, 3D noise is computed using 2D noise and z-gradients.
    Compute the hash for the current and next integer $z$-slice:
    $$ h_0 = \text{hash}(i_z + 1) $$
    $$ h_1 = \text{hash}(i_z + 2) $$
    Index the 2D textures using the $x, y$ coordinates and the $z$-hash:
    $$ c_0 = (i_x, i_y, h_0) $$
    $$ c_1 = (i_x, i_y, h_1) $$
    Retrieve 2D noise values $n_0 = \text{tex\_2d}(c_0)$, $n_1 = \text{tex\_2d}(c_1)$ and z-gradients $g_0 = \text{tex\_zgrad}(c_0)$, $g_1 = \text{tex\_zgrad}(c_1)$ [S1, unofficial].

5.  **Interpolation:**
    Perform trilinear interpolation. First, interpolate in $z$ using the retrieved values:
    $$ n_z = \text{lerp}(f_z, n_0, n_1) $$
    $$ g_z = \text{lerp}(f_z, g_0, g_1) $$
    Then, interpolate in $x$ and $y$ using the standard Perlin fading and gradient dot products for the 2D slices, combining them with the $z$ component. The final formula provided in the source for separate textures is:
    $$ N(p) = \text{lerp}(f_z, \text{lerp}(f_x, \text{lerp}(f_y, \text{grad}(c_0), \text{grad}(c_0')), \dots), \dots) $$
    *Note: The source provides a specific final formula for the optimized case:*
    $$ N(p) = \text{lerp}(f_z, \text{lerp}(f_x, \text{lerp}(f_y, \text{grad}(c_0), \text{grad}(c_0')), \text{lerp}(f_y, \text{grad}(c_1), \text{grad}(c_1'))), \dots) $$
    Actually, the source formula (19) is:
    $$ \text{lerp}(f_z, \text{lerp}(f_x, \text{lerp}(f_y, \text{grad}(c_0), \text{grad}(c_0')), \text{lerp}(f_y, \text{grad}(c_1), \text{grad}(c_1'))), \dots) $$
    Wait, the source text for formula 19 is:
    `flerp( pzf, mNoise2( c0 ) + pzf * zgrad( c0 ), mNoise2( c1 ) + pzf * zgrad( c1 ) )`
    This implies the 2D noise and gradient are pre-combined or accessed such that the final step is a linear interpolation between the two z-slices, where each slice value includes the z-gradient contribution.
    Specifically:
    $$ V_0 = N_{2D}(c_0) + f_z \cdot \frac{\partial N}{\partial z}(c_0) $$
    $$ V_1 = N_{2D}(c_1) + f_z \cdot \frac{\partial N}{\partial z}(c_1) $$
    $$ N(p) = \text{lerp}(f_z, V_0, V_1) $$
    Where $N_{2D}$ and $\frac{\partial N}{\partial z}$ are computed using the 2D hash and gradient logic on $c_0, c_1$ [S1, unofficial].

6.  **Modulo Implementation:**
    *   **Exact (5-instruction):** $i \bmod M = i - \lfloor i/M \rfloor \cdot M$. Used for 2D noise to ensure periodicity at $M$ [S1, unofficial].
    *   **Inexact (4-instruction):** $i \bmod M = \text{frac}(i/M) \cdot M$. Used for 3D+ where precision loss is less critical [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bbs_mod(x, M):
    # Inexact 4-instruction version for 3D
    return (x / M - floor(x / M)) * M

def bbs_hash(x, M):
    return bbs_mod(x * x, M)

def hash3(x, y, z, M):
    h = bbs_hash(x + 1, M)
    h = bbs_hash(h + y + 1, M)
    return bbs_hash(h + z + 1, M)

def grad3d(fx, fy, fz, corner_bits):
    # corner_bits is a tuple (j, k, l) in {0, 1}
    sx = 1 if corner_bits[0] == 0 else -1
    sy = 1 if corner_bits[1] == 0 else -1
    sz = 1 if corner_bits[2] == 0 else -1
    return sx * fx + sy * fy + sz * fz

def noise_3d(p, M):
    ix, iy, iz = floor(p)
    fx, fy, fz = p - (ix, iy, iz)
    
    # Compute hashes for z-slices
    h0 = hash3(ix, iy, iz, M)
    h1 = hash3(ix, iy, iz + 1, M)
    
    # Access 2D noise and z-gradient for the two z-slices
    # Note: In a pure implementation without textures, this would be computed recursively.
    # The source describes using textures for optimization.
    # Here we assume a pure computation for completeness, but the source emphasizes texture use.
    # For the sake of the "How it works" section based on the source's optimization:
    
    # c0 and c1 are indices for 2D texture lookup
    c0 = (ix, iy, h0)
    c1 = (ix, iy, h1)
    
    # Retrieve pre-computed 2D noise and z-gradient
    n0 = texture_lookup_2d(c0)
    g0 = texture_lookup_zgrad(c0)
    n1 = texture_lookup_2d(c1)
    g1 = texture_lookup_zgrad(c1)
    
    # Combine z-component
    v0 = n0 + fz * g0
    v1 = n1 + fz * g1
    
    return lerp(fz, v0, v1)
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The BBS hash involves squaring and modulo, which is more expensive than a simple table lookup but avoids memory latency [S1, unofficial].
*   **Memory Complexity:** $O(1)$ if purely computed. $O(1)$ with texture optimization (storing 2D slices) [S1, unofficial].
*   **Performance:** The 4-instruction modulo is faster but inexact. The 5-instruction modulo is exact but slower. Texture lookups accelerate 3D noise by reducing dependent lookups [S1, unofficial].
*   **Strengths:** No permutation table required; suitable for GPUs with limited constant memory; dimension reducibility reduces texture fetches [S1, unofficial].
*   **Limitations:** Axis-aligned clumping is visible in 2D results; the inexact modulo causes non-repeating noise at modulus $M$ in lower dimensions [S1, unofficial].
*   **When to use:** When GPU constant memory is constrained and a permutation table cannot be stored; for 3D/4D turbulence where clumping is less noticeable [S1, unofficial].
*   **When not to use:** When exact periodicity at a specific modulus is required in 2D; when visual quality in 2D is critical due to clumping [S1, unofficial].

## Background
Modified Noise was introduced in a thesis on real-time volumetric rendering of fire in a production system [S1, unofficial]. It modifies Perlin's improved noise to be more suitable for GPU implementation by removing table lookups and using the Blum-Blum-Shub generator [S1, unofficial].

## Sources
[S1] Real-Time Volumetric Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-thesis/YVanzine_thesis.pdf, fetched 2026-10-27 (unofficial)
