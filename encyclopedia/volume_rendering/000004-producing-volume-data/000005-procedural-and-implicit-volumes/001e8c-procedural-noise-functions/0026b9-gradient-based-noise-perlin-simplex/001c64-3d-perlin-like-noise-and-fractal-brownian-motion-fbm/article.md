# 3D Perlin-like Noise and Fractal Brownian Motion (FBM)

Also called: 3D Noise, 3D Perlin-like Noise, Fractal Brownian Motion, Fractional Brownian Motion, Volume Noise Source, Volumetric Noise

## Overview
3D Perlin-like Noise generates smooth, pseudo-random scalar values on a 3D unit lattice using gradient interpolation, while Fractal Brownian Motion (FBM) constructs multi-scale stochastic patterns by summing multiple octaves of this base noise with increasing frequency and decreasing amplitude [S1, unofficial][S2, unofficial][S3][S4, unofficial]. It solves the problem of creating natural-looking, continuous volumetric density fields (such as clouds, fog, or terrain) without the tiling artifacts or discrete steps of value noise. It is distinguished from [Simplex Noise](978-simplex-noise.md) by its use of a cubic lattice and trilinear interpolation, and from [Improved Perlin Noise (4D)](3723-improved-perlin-noise-4d.md) by its specific 3D gradient set and interpolation order [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
*   Query point $\mathbf{p} = (x, y, z) \in \mathbb{R}^3$.
*   Base noise function $N(\mathbf{p})$ (e.g., from [Gradient-based Noise](3860-gradient-based-noise-perlin-simplex.md)).
*   FBM parameters: number of octaves $n$, persistence $\alpha$ (amplitude decay), lacunarity $\lambda$ (frequency gain), and optionally a base frequency $f_0$ and phase shift $\mathbf{s}$.

**Outputs:**
*   Scalar value $B_n(\mathbf{p}) \in \mathbb{R}$, typically normalized to $[-1, 1]$ or $[0, 1]$.

**Data Structures:**
*   **Permutation Table:** An array `perm` of size 256 (or 512 to handle overflow) containing a shuffled permutation of integers $0 \dots 255$. Used to map integer coordinates to gradient indices [S6, unofficial][S11, unofficial][S12, unofficial].
*   **Gradient Vectors:** 256 pre-computed 3D vectors $\mathbf{g}_i$, typically normalized face diagonals of a cube (e.g., $(\pm 1, \pm 1, 0)$ permutations) or random unit vectors [S1, unofficial][S2, unofficial][S12, unofficial].

**Algorithm Steps:**

1.  **Base Noise Evaluation ($N(\mathbf{p})$):**
    *   Compute integer lattice coordinates: $\mathbf{i} = \lfloor \mathbf{p} \rfloor$ and fractional offset $\mathbf{f} = \mathbf{p} - \mathbf{i}$ [S2, unofficial][S4, unofficial].
    *   Compute fade function $u = f(t) = 6t^5 - 15t^4 + 10t^3$ for each component of $\mathbf{f}$ to ensure $C^1$ continuity [S1, unofficial][S17, unofficial].
    *   Hash the 8 corner coordinates of the unit cube containing $\mathbf{p}$ to gradient indices using the permutation table: $h_{ijk} = \text{perm}[(\mathbf{i}_x + i) \oplus \text{perm}[(\mathbf{i}_y + j) \oplus \text{perm}[\mathbf{i}_z + k]]] \pmod{256}$ for $i,j,k \in \{0,1\}$ [S11, unofficial][S12, unofficial].
    *   Retrieve gradient vectors $\mathbf{g}_{ijk}$ and compute dot products with distance vectors $\mathbf{d}_{ijk} = \mathbf{p} - (\mathbf{i} + (i,j,k))$: $n_{ijk} = \mathbf{g}_{ijk} \cdot \mathbf{d}_{ijk}$ [S1, unofficial][S12, unofficial].
    *   Perform trilinear interpolation of the 8 corner values using the fade weights $u_x, u_y, u_z$ [S1, unofficial][S2, unofficial].

2.  **FBM Summation:**
    *   Initialize `sum = 0.0`, `amplitude = 0.5` (or 1.0 depending on normalization), `frequency = f_0`, and `position = p` [S5, unofficial][S8, unofficial].
    *   For each octave $i$ from $0$ to $n-1$:
        *   Evaluate base noise: $val = N(\text{position} \times \text{frequency})$ [S8, unofficial].
        *   Add weighted contribution: `sum += val * amplitude` [S5, unofficial][S8, unofficial].
        *   Update parameters for next octave:
            *   `frequency *= lacunarity` (typically 2.0) [S7, unofficial][S8, unofficial].
            *   `amplitude *= persistence` (typically 0.5) [S7, unofficial][S8, unofficial].
            *   `position *= lacunarity` (or apply a random shift $\mathbf{s}$ to avoid periodicity: $\text{position} = \text{position} \times \lambda + \mathbf{s}$) [S5, unofficial][S16, unofficial].
    *   Return `sum`.

**Formulas:**
The standard FBM definition with lacunarity parameter $\beta$ (where $\beta=1$ corresponds to $1/f$ noise) is [S3]:
$$ B_n(\mathbf{p}) = \sum_{i=1}^n o_i(\mathbf{p}), \quad o_i(\mathbf{p}) = \frac{noise(2^{i-1}\mathbf{p})}{2^{\beta(i-1)}} $$
In implementation, this maps to:
$$ \text{sum} = \sum_{i=0}^{n-1} \text{persistence}^i \cdot N(\lambda^i \mathbf{p}) $$
where $\lambda$ is the lacunarity and $\text{persistence} = 2^{-\beta}$ [S3][S6, unofficial].

**Parameters:**
*   **Octaves ($n$):** Number of layers. Typical values: 4–7 [S8, unofficial][S12, unofficial][S16, unofficial].
*   **Persistence ($\alpha$):** Amplitude decay per octave. Default: 0.5 [S6, unofficial][S7, unofficial].
*   **Lacunarity ($\lambda$):** Frequency gain per octave. Default: 2.0 [S7, unofficial][S20].
*   **Seed:** Integer used to initialize the permutation table [S19, unofficial][S20].

**Edge Cases and Pitfalls:**
*   **Single Octave Limit:** Some implementations (e.g., `fbm3` in [S1, unofficial]) only return $0.5 \times N(\mathbf{p})$, lacking multi-scale detail [S1, unofficial][S11, unofficial].
*   **Periodicity:** Without a random shift $\mathbf{s}$ or non-integer lacunarity, FBM exhibits visible repetition along axes [S5, unofficial][S16, unofficial].
*   **Hashing Artifacts:** Using trigonometric hash functions (e.g., `sin`) can cause architectural differences between GPUs; integer-based permutation tables are preferred for consistency [S17, unofficial].
*   **Division by Zero:** Vector normalization or division operations must guard against zero vectors [S12, unofficial].
*   **Boundary Conditions:** When sampling outside defined bounds (e.g., cloud height limits), density should be explicitly clamped to 0 [S16, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fade(t):
    # Quintic smoothstep for C1 continuity
    return 6 * t**5 - 15 * t**4 + 10 * t**3

def lerp(t, a, b):
    return a + t * (b - a)

def grad(hash_val, x, y, z):
    # Retrieve gradient vector g from permutation table based on hash_val
    # Compute dot product g . (x, y, z)
    # Note: Implementation depends on specific gradient set (e.g., 12 face diagonals)
    pass

def perlin_noise(x, y, z):
    # 1. Integer and fractional parts
    xi = int(floor(x))
    yi = int(floor(y))
    zi = int(floor(z))
    xf = x - floor(x)
    yf = y - floor(y)
    zf = z - floor(z)
    
    # 2. Fade curves
    u = fade(xf)
    v = fade(yf)
    w = fade(zf)
    
    # 3. Hashing corners
    # Assuming perm table and grad function are available
    A = perm[xi] + yi
    AA = perm[A] + zi
    AB = perm[A + 1] + zi
    B = perm[xi + 1] + yi
    BA = perm[B] + zi
    BB = perm[B + 1] + zi
    
    # 4. Gradient contributions
    x1 = lerp(u, grad(perm[AA], xf, yf, zf), grad(perm[BA], xf - 1, yf, zf))
    x2 = lerp(u, grad(perm[AB], xf, yf - 1, zf), grad(perm[BB], xf - 1, yf - 1, zf))
    
    # 5. Interpolation
    y1 = lerp(v, x1, x2)
    y2 = lerp(v, 
              lerp(u, grad(perm[AA + 1], xf, yf, zf - 1), grad(perm[BA + 1], xf - 1, yf, zf - 1)),
              lerp(u, grad(perm[AB + 1], xf, yf - 1, zf - 1), grad(perm[BB + 1], xf - 1, yf - 1, zf - 1)))
              
    return lerp(w, y1, y2)

def fbm(x, y, z, octaves=4, persistence=0.5, lacunarity=2.0):
    total = 0.0
    amplitude = 0.5
    frequency = 1.0
    position_x = x
    position_y = y
    position_z = z
    
    for i in range(octaves):
        total += perlin_noise(position_x * frequency, 
                              position_y * frequency, 
                              position_z * frequency) * amplitude
        
        position_x *= lacunarity
        position_y *= lacunarity
        position_z *= lacunarity
        amplitude *= persistence
        
    return total
```

## Subtopics

### [Noise Field Generator](1302-noise-field-generator.md)
Produces a 3D scalar field by summing octaves of Simplex/Perlin noise with vertical gradient offsets. Chosen for realistic organic structures like clouds/terrain. Distinguished by vertical gradient integration and scalar summation vs. vector fields.

### [Global Cloud Cover with Curl Noise](1711-global-cloud-cover-with-curl-noise.md)
Generates planet-wide cloud patterns using mixed Perlin/Worley noise on a sphere. Chosen for large-scale atmospheric coherence. Distinguished by spherical mapping and divergence-free vector fields.

### [3D Perlin-Worley Noise Texture Generation](2355-3d-perlin-worley-noise-texture-generation.md)
Combines low-frequency Perlin for mass with inverted Worley noise for cellular erosion. Chosen for puffy cumulus structures. Distinguished by secondary Worley layer for detail.

### [Voxel World Generation with Perlin Noise](2556-voxel-world-generation-with-perlin-noise.md)
Maps fBm thresholds to discrete voxel materials (stone, dirt). Chosen for terrain generation. Distinguished by discrete material mapping and neighborhood checks.

### [Ridged Multifractal](4249-ridged-multifractal.md)
Generates sharp ridges using absolute value transformation and feedback. Chosen for terrain ridges/lightning. Distinguished by non-linear kernel and multiplicative feedback.

### 3D Noise Generation with Box Filtering
Generates a 3D tiling noise volume by applying a 3×3×3 box blur to hide trilinear interpolation artifacts. Chosen for seamless procedural volumes. Uses a seeded PRNG to initialize random values, averages the 3×3×3 neighborhood for each voxel, and wraps coordinates modulo dimension for tiling [S22, unofficial].

### FBM BVH Construction
Optimizes sphere tracing of FBM by presorting octaves by range and using analytic bounds. Chosen for efficient ray-marching of large FBM fields. Requires no memory storage for bounds; all octaves share the same Lipschitz constant when $\beta=1$ [S3].

### High Frequency Surface Noise Remapping
Remaps noise values using a range defined by low-frequency noise to introduce high-frequency detail on cloud surfaces. Chosen for realistic cloud edge detail. Uses a remapping function to clamp and scale noise based on a base density [S21, unofficial].

### Derivative Erosion FBM
Accumulates derivatives across octaves to dampen amplitude via `1/(1+|grad|^2)`. Chosen for realistic erosion patterns where steep slopes remain smooth. Updates frequency, amplitude, and position rotation per octave [S23].

### Tileable Volume Noise Generation
Generates four specific noise textures (Perlin-Worley FBM, Worley FBM, Curl Noise, Weather Data) for volumetric clouds. Chosen for pre-computed runtime efficiency. Uses specific resolutions (e.g., 128³ for Perlin-Worley) and RGBA channel mappings [S24, unofficial].

### Perlin Noise Erosion
Erodes Signed Distance Fields (SDF) by adding negative Perlin noise values to the distance. Chosen for surface weathering. Scales noise by `noise_scale` and applies `noise_amplitude` only when noise is negative [S25, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(n \cdot k)$ where $n$ is the number of octaves and $k$ is the cost of the base noise evaluation (constant for fixed lattice). Linear in octaves.
*   **Memory Complexity:** $O(1)$ for procedural evaluation (only permutation table stored, typically 256–512 integers). Pre-computed volumes require $O(D^3)$ for dimension $D$ [S15, unofficial].
*   **Performance:** GPU-friendly due to lack of branching and memory access patterns. Pre-computed 3D textures can cause runtime freezes if generated on-the-fly [S17, unofficial][S20].
*   **Strengths:** Continuous, smooth, natural-looking detail; easy to tune via persistence/lacunarity; tileable with proper hashing [S3][S22, unofficial].
*   **Limitations:** Single-octave variants lack detail [S1, unofficial]; periodicity if no shift is used [S5, unofficial]; hash function portability issues on GPUs [S17, unofficial]; division by zero risks in gradient normalization [S12, unofficial].
*   **When to use:** Procedural textures, cloud/fog density, terrain heightmaps.
*   **When not to use:** When strict tiling without blur is required (use box-filtered variant [S22, unofficial]) or when sharp, non-smooth features are needed (use Ridged Multifractal [S25, unofficial]).

## Background
Originated from Ken Perlin's 1985 paper "Improving Noise" [S1, unofficial]. FBM was introduced as a method to combine multiple frequencies of noise to create fractal-like patterns [S3]. It is widely used in computer graphics for stochastic pattern generation across many scales [S3]. Variants include value noise (using scalar interpolation) and gradient noise (using dot products) [S1, unofficial][S2, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] Minecraft Voxel DDA Raycast - Shadertoy, https://www.shadertoy.com/view/WccyWl, fetched 2026-10-26 (unofficial)
[S3] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S4] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S5] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S6] Y-o-p/fog, https://github.com/Y-o-p/fog, fetched 2026-10-26 (unofficial)
[S7] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S8] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S9] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S10] SebLague/Godot-Marching-Cubes, https://github.com/SebLague/Godot-Marching-Cubes, fetched 2026-10-26 (unofficial)
[S11] aparis69/MarchingCubeCpp, https://github.com/aparis69/MarchingCubeCpp, fetched 2026-10-26 (unofficial)
[S12] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S13] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
[S14] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S15] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S16] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
[S17] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S18] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S19] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S20] Volume Noise Sources, https://docs.occasoftware.com/buto/volume-noise-sources, fetched 2026-10-27
[S21] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
[S22] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S23] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S24] [volumetric-clouds-v2][RenderEngine] 1:1 复刻 NadirRoGue/RenderEngine 体积云, https://github.com/ruochenhua/AetherEngine/issues/139, fetched 2026-10-28 (unofficial)
[S25] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
