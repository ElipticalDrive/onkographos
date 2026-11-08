# Stochastic Jittering

Also called: Blue Noise Jittering for Banding Reduction, Jittered Ray Sampling, Jittering Sample Positions, Ray Jitter with Random Noise, Ray Jittering, Stochastic Noise, Stochastic Sampling

## Overview
Stochastic jittering is a sampling technique in direct volume ray casting that perturbs sample positions along rays using random noise to mitigate banding and aliasing artifacts caused by regular, undersampled grid structures [S1, unofficial][S2, unofficial][S5, unofficial][S6, unofficial]. It sets itself apart from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by breaking the periodicity of sample placement, and from [Monte Carlo Volume Rendering](2060-monte-carlo-volume-rendering.md) by typically using a fixed, pre-computed noise texture rather than generating high-variance random numbers per sample, trading visible banding for less objectionable dither patterns [S5, unofficial][S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, base step size $\Delta t$, pixel coordinates $(u, v)$, noise texture $\mathcal{N}$, maximum noise factor $\alpha$.
*   **Output:** Perturbed sample positions $\mathbf{p}_n$ for $n=0, 1, \dots, N-1$ along the ray.

**Data Structures:**
*   **Noise Texture ($\mathcal{N}$):** A 2D texture storing noise values.
    *   **Format:** Typically single-channel (Red) or multi-channel.
    *   **Size:** $W \times H$ pixels. Common size is $64 \times 64$ [S3, unofficial][S4].
    *   **Filtering:** `GL_NEAREST` (or equivalent) for both minification and magnification filters to preserve sharp noise patterns without interpolation [S2, unofficial].
    *   **Wrapping:** `GL_REPEAT` (or equivalent) for both S and T axes to allow seamless tiling [S2, unofficial].
    *   **Content:** Can be white noise or blue noise. Blue noise is preferred because it is intrinsically tileable and easier to blur out than white noise [S6, unofficial].

**Parameters:**
*   **MaxNoiseFactor ($\alpha$):** Controls the maximum magnitude of the jitter offset. Typical value: $0.75$ [S4].
*   **JitterTexSize:** Dimensions of the noise texture. Typical value: $64$ [S4].
*   **RayStepSize ($\Delta t$):** The base step size used for scaling the jitter magnitude [S3, unofficial].

**Algorithm:**
1.  **Pre-computation:** Generate or load a 2D noise texture $\mathcal{N}$ of size $J \times J$. Configure texture sampling to use nearest-neighbor filtering and repeat wrapping [S2, unofficial].
2.  **Ray Initialization:** For each pixel $(u, v)$, compute the base ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
3.  **Jitter Offset Calculation:**
    *   Compute texture coordinates for the noise lookup:
        $$ u_{tex} = \frac{u \pmod J}{J}, \quad v_{tex} = \frac{v \pmod J}{J} $$
    *   Sample the noise texture to get a jitter value $j$:
        $$ j = \mathcal{N}(u_{tex}, v_{tex}) $$
        *Note: In some implementations, $j$ is sampled from the red channel [S3, unofficial]. If using blue noise, $j$ is typically in $[0, 1]$ or $[-0.5, 0.5]$ depending on generation.*
4.  **Sample Position Perturbation:**
    *   For each sample index $n$ along the ray:
        *   Calculate the nominal position:
            $$ t_n = t_0 + n \cdot \Delta t $$
        *   Apply jitter. The method varies by implementation:
            *   **Method A (Position Jitter):** Offset the sample position along the ray direction by a fraction of the step size scaled by the noise factor and the jitter value [S3, unofficial][S5, unofficial]:
                $$ \mathbf{p}_n = \mathbf{o} + (t_n + \alpha \cdot \Delta t \cdot (j - 0.5)) \cdot \mathbf{d} $$
                *Note: [S5, unofficial] describes replacing the midpoint $(n + 0.5)$ with $(n + rand())$, where $rand() \in [0, 1]$. This is equivalent to adding a uniform offset in $[0, \Delta t]$. [S3, unofficial] explicitly scales by $\alpha$ and uses a texture sample.*
            *   **Method B (Origin Jitter):** Offset the ray origin $\mathbf{o}$ by a small vector derived from the noise texture, effectively shifting the entire ray path [S1, unofficial][S4].
5.  **Volume Sampling:** Sample the volume data at $\mathbf{p}_n$ and accumulate color and opacity using standard volume rendering integration (e.g., front-to-back compositing).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computed noise texture N of size JxJ, sampled with nearest neighbor and repeat wrap
# Parameters: alpha (MaxNoiseFactor), delta_t (RayStepSize), J (JitterTexSize)

def compute_jittered_sample_positions(ray_origin, ray_dir, t0, num_samples, alpha, delta_t, J, N):
    """
    Computes sample positions along a ray with stochastic jittering.
    
    Args:
        ray_origin: vec3, origin of the ray
        ray_dir: vec3, normalized direction of the ray
        t0: float, start distance of the ray within the volume
        num_samples: int, number of samples to take
        alpha: float, maximum noise factor (e.g., 0.75)
        delta_t: float, base step size
        J: int, size of the noise texture
        N: texture_sampler, function to sample noise texture at (u, v)
        
    Returns:
        list of vec3, perturbed sample positions
    """
    # 1. Determine jitter value based on pixel coordinates
    # Assuming pixel coordinates (px, py) are available in the caller context
    # For this function, we assume jitter is constant for the whole ray per pixel
    # or varies per sample if texture coordinates are updated.
    # Standard implementation (S3) uses pixel coordinates for a single jitter offset
    # or per-sample if texture coordinates are mapped to sample index.
    # Here we follow S3: jitter is based on pixel coordinates.
    
    # Placeholder: pixel coordinates (px, py) must be passed or available
    # u_tex = (px % J) / J
    # v_tex = (py % J) / J
    # jitter_val = N(u_tex, v_tex)
    
    # For generality, let's assume jitter_val is provided or computed externally
    # If jitter is per-sample, u_tex would involve sample index n
    jitter_val = 0.5 # Placeholder for actual texture lookup
    
    positions = []
    for n in range(num_samples):
        # Nominal position
        t_nominal = t0 + n * delta_t
        
        # Jittered offset
        # S5: t = t0 + step_size * (n + rand())
        # S3: offset = RayStepSize * jitterOffset
        # Combining: offset = alpha * delta_t * (jitter_val - 0.5) 
        # to center the jitter around the nominal position if jitter_val is in [0,1]
        
        jitter_offset = alpha * delta_t * (jitter_val - 0.5)
        
        t_jittered = t_nominal + jitter_offset
        
        # Compute position
        pos = ray_origin + t_jittered * ray_dir
        positions.append(pos)
        
    return positions
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors can accumulate if jitter offsets are very small relative to ray origin coordinates.
*   **Boundary Conditions:** Ensure noise texture coordinates are correctly wrapped to avoid edge artifacts [S2, unofficial].
*   **Artifact Transformation:** Jittering transforms banding artifacts into noise. Without refinement or sufficient sampling, isosurface rendering errors may still be present, though less visible [S4].
*   **Sampling Rate Dependency:** The benefit of jittering is significant at low sampling rates (e.g., 0.25, 0.5) but vanishes at higher sampling rates [S1, unofficial].

## Subtopics

### [Slice Artifact Reduction](1628-slice-artifact-reduction.md)
Mitigates "wood grain" artifacts by jittering ray start positions and performing a local solidity check.

### [Temporal Filtering and Jittering](2762-temporal-filtering-and-jittering.md)
Combines sub-voxel depth jittering with temporal reprojection to smooth noise and improve perceived resolution.

### Low-Resolution Stochastic Sampling
An extension where sampling occurs at a lower resolution to reduce texture bandwidth requirements, combined with varying the sample pattern with stochastic sampling to reduce regular pattern artifacts [S7, book].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. The noise texture lookup is $O(1)$.
*   **Memory Complexity:** $O(J^2)$ for the noise texture, where $J$ is the texture size (e.g., $64 \times 64$) [S4].
*   **Measured Performance:** Provides slight visual improvement at low sampling rates (0.25, 0.5) but the benefit vanishes with higher sampling rates [S1, unofficial].
*   **Strengths:** Reduces visible banding artifacts; simple to implement; low memory overhead.
*   **Limitations:** Replaces banding with noise; noise can be reduced using quasi Monte-Carlo methods but basic implementation uses simple random/blue noise [S5, unofficial]; benefit diminishes at high sampling rates [S1, unofficial].
*   **When to Use:** When undersampling volumes for performance and banding artifacts are visible.
*   **When Not to Use:** When high sampling rates are feasible, or when strict deterministic results are required.

## Background
Stochastic jittering originated as a technique to mitigate aliasing in volume rendering by introducing controlled noise to sample positions [S4]. It is implemented in libraries such as VTK (as `SetNoise`) [S4] and used in various ray casting implementations [S2, unofficial][S3, unofficial]. Blue noise textures for this purpose are available from libraries like 'free-blue-noise-textures' by Calinou [S6, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
[S2] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
[S3] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S4] Adding Refined Isosurface Rendering and Shadow Mapping to vtkGPUVolumeRayCastMapper, https://insight-journal.org/pdfs/880.pdf, fetched 2026-10-27
[S5] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S6] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S7] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
