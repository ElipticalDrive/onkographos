# 3D Perlin-like Noise and Fractal Brownian Motion (FBM)

Also called: Fractal Brownian Motion, Fractional Brownian Motion, 3D Perlin Noise, 3D Perlin-like Noise

## Overview
3D Perlin-like Noise generates smooth pseudo-random scalar values on a 3D unit lattice using hash-derived corner values, Hermite smoothing, and trilinear interpolation [S1, S2, unofficial][S4, unofficial]. Fractal Brownian Motion (FBM) constructs multi-scale fractal noise by summing multiple octaves of a base noise function with scaled frequency and amplitude [S3][S4, unofficial][S5, unofficial]. Together, they solve the problem of generating natural-looking volumetric textures (e.g., clouds, terrain, smoke) that lack the repetitive, uniform appearance of single-layer noise [S3][S4, unofficial]. They differ from [Simplex Noise](978-simplex-noise.md) by using a cubic lattice with trilinear interpolation rather than a simplex grid with tetrahedral interpolation, and from [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md) by operating in 3D space with standard cubic smoothing functions [S1, S2, unofficial][S10, unofficial].

## How it works
**Inputs:**
*   $p = (x, y, z)$: Input 3D coordinate.
*   $n$: Number of octaves (integer, typically 4–7) [S10, unofficial][S14, unofficial].
*   $\beta$: Lacunarity parameter for FBM (typically 1.0) [S3].
*   $L$: Lacunarity multiplier for frequency (typically 2.0) [S6, unofficial].
*   $G$: Gain multiplier for amplitude (typically 0.5) [S6, unofficial].
*   $P$: Permutation table of size 256 (or 512 for overflow safety) containing a shuffled sequence of integers 0–255 [S9, unofficial][S10, unofficial].
*   $V$: Array of 256 random 3D gradient vectors, normalized to unit length [S10, unofficial].

**Outputs:**
*   $B_n(p)$: The FBM noise value, typically in range $[-1, 1]$ or $[0, 1]$ depending on normalization [S3][S5, unofficial].

**Data Structures:**
*   `perm`: `uint8_t[512]` (or `int[256]` with masking). Initialized with a shuffled permutation of $0 \dots 255$ [S9, unofficial][S10, unofficial].
*   `grad`: `vec3[256]`. Pre-computed random unit vectors. `grad[i]` is retrieved via `perm` indexing [S10, unofficial].

**Step 1: Base Noise Evaluation (Perlin-like)** [S7, unofficial].
1.  Compute integer lattice coordinates $i = \lfloor p \rfloor$ and fractional offset $f = p - i$ [S1, S2, S4, S5, S15, unofficial].
2.  Apply Hermite smoothing to $f$: $s = f \cdot f \cdot (3.0 - 2.0 \cdot f)$ [S1, S2, S4, S10, unofficial]. This creates the S-curve weighting function [S10, unofficial].
3.  Retrieve hash indices for the 8 corners of the unit cube containing $p$ [S1, S2, unofficial]. For corner offsets $dx, dy, dz \in \{0, 1\}$, compute indices using the permutation table: $h = P[(i.x + dx) \oplus ((i.y + dy) \ll 8) \oplus ((i.z + dz) \ll 16)]$ [S10, unofficial].
4.  Retrieve gradient vectors $g_{dx,dy,dz} = V[h]$ [S10, unofficial].
5.  Compute dot products between gradient vectors and vectors from corner to point:
    $$ D_{dx,dy,dz} = g_{dx,dy,dz} \cdot (p - (i + (dx, dy, dz))) $$
6.  Perform trilinear interpolation of the 8 dot products using smoothed weights $s_x, s_y, s_z$:
    *   Interpolate along X:
        $$ L_0 = \text{lerp}(D_{0,0,0}, D_{1,0,0}, s_x), \quad L_1 = \text{lerp}(D_{0,1,0}, D_{1,1,0}, s_x) $$
        $$ L_2 = \text{lerp}(D_{0,0,1}, D_{1,0,1}, s_x), \quad L_3 = \text{lerp}(D_{0,1,1}, D_{1,1,1}, s_x) $$
    *   Interpolate along Y:
        $$ M_0 = \text{lerp}(L_0, L_1, s_y), \quad M_1 = \text{lerp}(L_2, L_3, s_y) $$
    *   Interpolate along Z:
        $$ \text{noise}(p) = \text{lerp}(M_0, M_1, s_z) $$
    *   Result is in $[-1, 1]$ [S1, S2, S4, S10, unofficial].

**Step 2: FBM Summation**
1.  Initialize `sum = 0.0`, `freq = 1.0`, `amp = 0.5` [S6, S7, S14, unofficial].
2.  Loop for $i$ from $0$ to $n-1$:
    *   Evaluate base noise: $val = \text{noise}(p \cdot freq)$ [S7, S14, unofficial].
    *   Add to sum: `sum += val * amp` [S7, S14, unofficial].
    *   Update frequency: $freq *= L$ (Lacunarity, typically 2.0) [S6, S7, S14, unofficial].
    *   Update amplitude: $amp *= G$ (Gain, typically 0.5) [S6, S7, S14, unofficial].
    *   *Alternative FBM Definition:* Some sources define octave $i$ as $\frac{noise(2^{i-1}p)}{2^{\beta(i-1)}}$ with $\beta=1$ [S3]. This is equivalent to $L=2, G=0.5$.
3.  Return `sum` [S7, S14, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def perlin_noise(p, perm, grad):
    i = floor(p)
    f = p - i
    s = f * f * (3.0 - 2.0 * f)
    
    u = i.x & 255
    v = i.y & 255
    w = i.z & 255
    
    def get_hash(x, y, z):
        return perm[(x + perm[(y + perm[z & 255] & 255) & 255] & 255) & 255]

    h000 = get_hash(u, v, w)
    h100 = get_hash(u+1, v, w)
    h010 = get_hash(u, v+1, w)
    h110 = get_hash(u+1, v+1, w)
    h001 = get_hash(u, v, w+1)
    h101 = get_hash(u+1, v, w+1)
    h011 = get_hash(u, v+1, w+1)
    h111 = get_hash(u+1, v+1, w+1)
    
    g000 = grad[h000]
    g100 = grad[h100]
    g010 = grad[h010]
    g110 = grad[h110]
    g001 = grad[h001]
    g101 = grad[h101]
    g011 = grad[h011]
    g111 = grad[h111]
    
    d000 = dot(g000, p - (i + vec3(0,0,0)))
    d100 = dot(g100, p - (i + vec3(1,0,0)))
    d010 = dot(g010, p - (i + vec3(0,1,0)))
    d110 = dot(g110, p - (i + vec3(1,1,0)))
    d001 = dot(g001, p - (i + vec3(0,0,1)))
    d101 = dot(g101, p - (i + vec3(1,0,1)))
    d011 = dot(g011, p - (i + vec3(0,1,1)))
    d111 = dot(g111, p - (i + vec3(1,1,1)))
    
    nx00 = lerp(d000, d100, s.x)
    nx10 = lerp(d010, d110, s.x)
    nx01 = lerp(d001, d101, s.x)
    nx11 = lerp(d011, d111, s.x)
    
    nxy0 = lerp(nx00, nx10, s.y)
    nxy1 = lerp(nx01, nx11, s.y)
    
    return lerp(nxy0, nxy1, s.z)

def fbm(p, octaves, perm, grad, lacunarity=2.0, gain=0.5):
    total = 0.0
    frequency = 1.0
    amplitude = 0.5
    
    for i in range(octaves):
        total += perlin_noise(p * frequency, perm, grad) * amplitude
        frequency *= lacunarity
        amplitude *= gain
        
    return total
```

**Edge Cases and Pitfalls:**
*   **Hash Function Portability:** Hash functions using trigonometric functions (e.g., `sin`) can produce different results across GPU architectures [S15, unofficial]. Use integer-based permutation tables for consistency [S5, S9, S10, unofficial].
*   **Division by Zero:** Vector normalization or division operations must guard against zero vectors/norms [S10, unofficial].
*   **Boundary Conditions:** Masking coordinates with `& 255` assumes a permutation table size of 256 [S10, unofficial].
*   **Single Octave Limit:** Some implementations (e.g., `fbm3` in [S1, S2, unofficial]) return only a single octave scaled by 0.5, lacking multi-scale detail [S1, S2, unofficial].

## Subtopics

### [Noise Field Generator](1302-noise-field-generator.md)
The Noise Field Generator produces a 3D scalar field by summing multiple octaves of gradient-based noise, where each octave scales frequency and amplitude to create multi-scale detail. It solves the problem of generating natural-looking volumetric textures (such as soft clouds or terrain density) that lack the repetitive, uniform appearance of single-layer noise. It differs from its parent, [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md), by explicitly defining the multi-octave summation loop, the specific scaling of frequency (lacunarity) and amplitude (gain), and the optional vertical gradient offset used to break symmetry in volumetric data.

### [Global Cloud Cover with Curl Noise](1711-global-cloud-cover-with-curl-noise.md)
Generating global cloud patterns by adding octaves of mixed Perlin and Worley noise, subtracting a threshold, and clamping to create 2D patterns on a spherical surface, restricted between bottom and top heights.

### [3D Perlin-Worley Noise Texture Generation](2355-3d-perlin-worley-noise-texture-generation.md)
This method generates 3D volumetric density maps for procedural textures, specifically targeting cloud-like structures, by combining low-frequency Perlin noise for broad mass with inverted Worley (cellular) noise to carve hollows. It differs from pure Perlin noise by introducing cellular erosion for puffy cumulus appearances and from pure Worley by retaining large-scale continuity. It builds on [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md) by adding a secondary Worley noise layer for detail erosion.

### [Voxel World Generation with Perlin Noise](2556-voxel-world-generation-with-perlin-noise.md)
This method generates a 3D voxel-based world by evaluating fractal Brownian motion (fBm) noise against a threshold to determine voxel existence and material type (stone, dirt, grass). It differs from sibling methods like the [Noise Field Generator](1302-noise-field-generator.md) by explicitly mapping noise values to discrete material layers based on vertical position and local neighborhood checks, rather than producing a continuous scalar field or texture. It builds on [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md) to provide the underlying pseudo-random noise function.

### [Ridged Multifractal](4249-ridged-multifractal.md)
Ridged Multifractal is a fractal noise technique that generates sharp, ridge-like structures by applying a non-linear transformation to the absolute value of base noise, combined with inter-octave feedback. Unlike standard FBM which produces soft, cloud-like volumes, Ridged Multifractal creates high-contrast linear features suitable for terrain ridges, lightning, or brittle fracture patterns. It builds on [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1260-3d-perlin-like-noise-and-fractal-brownian-motion-fbm.md) by replacing the linear summation with a ridge-specific kernel and a multiplicative feedback loop.

### 3D Noise Generation with Box Filtering
Generates a 3D tiling noise volume and applies a 3×3×3 box blur to hide trilinear interpolation grid artifacts, as advised for procedural volume rendering [S19, unofficial].
1. Initialize a 3D array of floats with uniform random values in [0,1] using a seeded PRNG [S19, unofficial].
2. For each voxel $(x,y,z)$, compute the average of its 3×3×3 neighborhood ($dx,dy,dz \in [-1,1]$) [S19, unofficial].
3. Wrap coordinates modulo $d$ (noise dimension) to ensure tiling [S19, unofficial].
4. Convert the averaged float to uint8_t in [0,255] and store in the output buffer [S19, unofficial].
Formula: $v_{out}(x,y,z,c) = \frac{1}{27} \sum_{dx=-1}^{1} \sum_{dy=-1}^{1} \sum_{dz=-1}^{1} v_{in}((x+dx)\%d, (y+dy)\%d, (z+dz)\%d, c) \cdot 255$, where $d$ is the noise dimension, $c$ is the RGBA channel, and $\%$ denotes modulo wrapping for tiling [S19, unofficial].

### FBM BVH Construction
A specific application of the BVH method to Fractional Brownian Motion (FBM), where octaves are presorted by range and have analytic bounds, requiring no memory storage for bounds [S3].
1. Presort octaves by range of values (higher to lower). This provides the optimal evaluation order for FBM [S3].
2. Use the analytic expressions for $\Delta_i$ and $s(i,n)$ to compute BVH bounds. No memory is needed to store them [S3].
3. For FBM with $\beta=1$, all octaves share the same Lipschitz constant. Presorting by range of value corresponds to the optimal pre-sorted order for step size maximization [S3].
Formula: For FBM, the function is the octave $o_i$, and the range is $\Delta_{f_i} = \frac{1}{2^{i-1}}$. The sum of ranges from $i$ to $j$ is $s(i, j) = \sum_{m=i}^j \frac{1}{2^{m-1}} = 2 - \frac{1}{2^{j-1}}$ [S3]. For FBM with Lipschitz noise constant $\lambda_{noise}$, the Lipschitz constant of octave $i$ is $\lambda_{f_i} = \lambda_{noise}$, and the total Lipschitz constant is $\lambda_{B_n} = n \cdot \lambda_{noise}$ [S3].

### High Frequency Surface Noise Remapping
Introducing high frequency noise on cloud surfaces by remapping the noise value using a range defined by low frequency noise, used by Guerilla Games [S18, unofficial].
Code: `float density = clamp(remap(noise, 1 - base, 1.0, 0.0, cap), 0.0, cap);` where `remap` linearly maps `value` from `[original_min, original_max]` to `[new_min, new_max]` [S18, unofficial].

### Derivative Erosion FBM
FBM that accumulates derivatives across octaves. The derivative magnitude dampens amplitude via `1/(1+|grad|^2)`, creating realistic erosion patterns where steep slopes stay smooth [S22].
1. Initialize `value=0.0`, `deriv=vec2(0.0)`, `amplitude=0.5` [S22].
2. Loop `octaves` [S22].
3. Get `n = noisedQ(p * frequency)` [S22].
4. Accumulate `deriv += n.yz` [S22].
5. Add `amplitude * n.x / (1.0 + dot(deriv, deriv))` to `value` [S22].
6. Update `frequency *= 2.0`, `amplitude *= 0.5`, `p = rot * p` [S22].

## Operating characteristics
*   **Time Complexity:** $O(n \cdot k)$ per sample, where $n$ is the number of octaves and $k$ is the cost of the base noise function (constant for fixed lattice size) [S3][S7, S14, unofficial].
*   **Memory Complexity:** $O(P + G)$, where $P$ is the permutation table size (256 or 512 integers) and $G$ is the gradient vector array size (256 vectors) [S5, S9, S10, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Linear with the number of octaves. Higher octaves increase detail but also computational cost [S3][S7, S14, unofficial].
*   **Strengths:** Generates continuous, smooth, pseudo-random fields with multi-scale detail suitable for natural phenomena [S3][S5, unofficial].
*   **Limitations:** Trilinear interpolation can cause grid artifacts (visible as straight lines) which may require box filtering or higher-order interpolation [S19, unofficial]. Hash functions using trigonometry are not portable across all hardware [S15, unofficial]. Single-octave implementations lack fractal detail [S1, S2, unofficial].
*   **When to use:** Procedural texture generation, volumetric cloud/terrain synthesis, and any application requiring smooth, non-repetitive noise fields [S3][S5, unofficial].
*   **When not to use:** When strict portability of hash-based noise is required without permutation tables [S15, unofficial], or when sharp, non-smooth features are needed (use Ridged Multifractal instead) [S18, unofficial].

## Background
Origin: Ken Perlin introduced Perlin noise in 1985 [S17, unofficial]. Fractal Brownian Motion (FBM) was popularized in computer graphics by Perlin and others for generating natural textures [S3]. The method is widely used in computer graphics to create stochastic patterns with details at potentially many different scales [S3]. It is used in voxel world generation, cloud rendering, and terrain synthesis [S1, S14, S15, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Minecraft Voxel DDA Raycast - Shadertoy, https://www.shadertoy.com/view/WccyWl, fetched 2026-10-26 (unofficial)
[S3] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S4] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S5] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S6] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S7] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S8] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S9] aparis69/MarchingCubeCpp, https://github.com/aparis69/MarchingCubeCpp, fetched 2026-10-26 (unofficial)
[S10] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S11] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
[S12] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S13] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S14] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
[S15] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S16] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S17] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S18] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
[S19] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
