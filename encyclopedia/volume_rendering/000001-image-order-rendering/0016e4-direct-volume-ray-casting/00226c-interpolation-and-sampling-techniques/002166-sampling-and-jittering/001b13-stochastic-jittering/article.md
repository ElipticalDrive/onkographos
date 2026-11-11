# Stochastic Jittering

Also called: Ray Jittering, Jittered Ray Sampling, Stochastic Sample Jittering

## Overview
Stochastic jittering is a sampling technique in direct volume ray casting that perturbs sample positions along rays using random noise to mitigate banding and aliasing artifacts caused by regular, undersampled grid structures [S1, unofficial][S2, unofficial][S5][S6, unofficial]. It sets itself apart from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by breaking the periodicity of sample placement, and from [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md) by using a fixed, pre-computed noise texture rather than distributing rays across a pixel footprint, trading visible banding for less objectionable dither patterns [S5][S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, base step size $\Delta t$, pixel coordinates $(u, v)$, noise texture $\mathcal{N}$, maximum noise factor $\alpha$ [S3, unofficial][S4, unofficial][S5].
*   **Output:** Perturbed sample positions $\mathbf{p}_n$ for $n=0, 1, \dots, N-1$ along the ray.

**Data Structures:**
*   **Noise Texture ($\mathcal{N}$):** A 2D texture storing noise values [S3, unofficial].
    *   **Format:** Typically single-channel (Red) [S3, unofficial].
    *   **Size:** $J \times J$ pixels. Typical value is $64 \times 64$ [S5].
    *   **Filtering:** `GL_NEAREST` (or equivalent) for both minification and magnification filters to preserve sharp noise patterns without interpolation [S3, unofficial].
    *   **Wrapping:** `GL_REPEAT` (or equivalent) for both S and T axes to allow seamless tiling [S3, unofficial].
    *   **Content:** Blue noise is preferred because it is intrinsically tileable and easier to blur out than white noise [S7, unofficial].

**Parameters:**
*   **MaxNoiseFactor ($\alpha$):** Controls the maximum magnitude of the jitter offset. Typical value: $0.75$ [S5].
*   **JitterTexSize ($J$):** Dimensions of the noise texture. Typical value: $64$ [S5].
*   **RayStepSize ($\Delta t$):** The base step size used for scaling the jitter magnitude [S3, unofficial][S4, unofficial].

**Algorithm:**
1.  **Pre-computation:** Load or generate a 2D noise texture $\mathcal{N}$ of size $J \times J$. Configure texture sampling to use nearest-neighbor filtering and repeat wrapping [S3, unofficial].
2.  **Ray Initialization:** Compute the base ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for the current pixel [G].
3.  **Jitter Offset Calculation:** Compute texture coordinates $u_{tex} = (u \bmod J) / J$ and $v_{tex} = (v \bmod J) / J$. Sample the red channel of $\mathcal{N}$ to obtain a jitter value $j \in [0, 1]$ [S3, unofficial][S4, unofficial].
4.  **Sample Position Perturbation:** For each sample index $n$, calculate the nominal distance $t_n = t_0 + n \cdot \Delta t$. Offset the position by scaling the jitter value with the step size and maximum noise factor: $\mathbf{p}_n = \mathbf{o} + (t_n + \alpha \cdot \Delta t \cdot j) \cdot \mathbf{d}$ [S3, unofficial][S4, unofficial][S6, unofficial]. This replaces the fixed midpoint placement $t_0 + \Delta t \cdot (n + 0.5)$ with a random position uniformly distributed within the segment [S6, unofficial].
5.  **Volume Sampling:** Sample the volume data at $\mathbf{p}_n$ and accumulate color and opacity using standard volume rendering integration (e.g., front-to-back compositing) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_jittered_sample_positions(ray_origin, ray_dir, t0, num_samples, alpha, delta_t, J, pixel_uv, noise_sampler):
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
        pixel_uv: tuple(float, float), pixel coordinates (u, v)
        noise_sampler: callable(u, v) -> float, samples the pre-computed noise texture
        
    Returns:
        list of vec3, perturbed sample positions
    """
    # 1. Compute texture coordinates from pixel coordinates
    u_tex = (pixel_uv[0] % J) / J
    v_tex = (pixel_uv[1] % J) / J
    
    # 2. Sample noise texture for jitter value
    jitter_val = noise_sampler(u_tex, v_tex)
    
    positions = []
    for n in range(num_samples):
        # 3. Calculate nominal position
        t_nominal = t0 + n * delta_t
        
        # 4. Apply jitter offset
        t_jittered = t_nominal + alpha * delta_t * jitter_val
        
        # 5. Compute 3D position
        pos = ray_origin + t_jittered * ray_dir
        positions.append(pos)
        
    return positions
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors can accumulate if jitter offsets are very small relative to ray origin coordinates [G].
*   **Boundary Conditions:** Ensure noise texture coordinates are correctly wrapped to avoid edge artifacts [S3, unofficial].
*   **Artifact Transformation:** Jittering transforms banding artifacts into noise. Without refinement or sufficient sampling, isosurface rendering errors may still be present, though less visible [S5].
*   **Sampling Rate Dependency:** The benefit of jittering is significant at low sampling rates (e.g., 0.25, 0.5) but vanishes at higher sampling rates [S2, unofficial].

## Subtopics

### [Slice Artifact Reduction](1628-slice-artifact-reduction.md)
Mitigates "wood grain" artifacts by jittering ray start positions and performing a local solidity check. It differs from standard ray casting by randomizing the initial ray origin along the viewing direction and verifying sampled voxels are truly part of the volume rather than edge artifacts. This method is particularly effective when using a low number of ray casting steps or when nearest-neighbor interpolation is employed.

### [Temporal Filtering and Jittering](2762-temporal-filtering-and-jittering.md)
Combines sub-voxel depth jittering with temporal reprojection to smooth noise and improve perceived volume resolution without extra raymarching steps. It distinguishes itself from static spatial filters by leveraging motion vectors to blend frames, and from pure stochastic sampling methods by focusing on temporal accumulation rather than single-sample unbiased estimation.

### Low-Resolution Stochastic Sampling
An extension where sampling occurs at a lower resolution to reduce texture bandwidth requirements, combined with varying the sample pattern with stochastic sampling to reduce regular pattern artifacts [S8, book].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. The noise texture lookup is $O(1)$ [G].
*   **Memory Complexity:** Memory overhead is proportional to the noise texture size $J \times J$ [S5].
*   **Measured Performance:** Provides slight visual improvement at low sampling rates (0.25, 0.5) but the benefit vanishes with higher sampling rates [S2, unofficial].
*   **Strengths:** Reduces visible banding artifacts; simple to implement; low memory overhead [S5].
*   **Limitations:** Replaces banding with noise; noise can be reduced using quasi Monte-Carlo methods but basic implementation uses simple random/blue noise [S6, unofficial]; benefit diminishes at high sampling rates [S2, unofficial].
*   **When to Use:** When undersampling volumes for performance and banding artifacts are visible [S2, unofficial].
*   **When Not to Use:** When high sampling rates are feasible, or when strict deterministic results are required [S2, unofficial].

## Background
Stochastic jittering originated as a technique to mitigate aliasing in volume rendering by introducing controlled noise to sample positions [S4, unofficial]. It is implemented in libraries such as VTK (as `SetNoise`) [S5] and used in various ray casting implementations [S2, unofficial][S3, unofficial]. Blue noise textures for this purpose are available from libraries like 'free-blue-noise-textures' by Calinou [S7, unofficial].

## Sources
[S1] "Volume Rendering for Developers" Group Read Part 1 [2023-03-30], https://www.youtube.com/watch?v=mZzMVD4LQvk, fetched 2026-10-26 (unofficial)
[S2] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
[S3] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
[S4] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S5] Adding Refined Isosurface Rendering and Shadow Mapping to vtkGPUVolumeRayCastMapper, https://insight-journal.org/pdfs/880.pdf, fetched 2026-10-27
[S6] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S7] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S8] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
