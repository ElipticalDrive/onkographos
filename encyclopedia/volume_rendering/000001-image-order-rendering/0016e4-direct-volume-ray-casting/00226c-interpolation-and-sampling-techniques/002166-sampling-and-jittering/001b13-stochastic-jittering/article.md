# Stochastic Jittering

Also called: Jittering Sample Positions, Stochastic Noise, Stochastic Sampling

## Overview
Stochastic Jittering is a sampling technique in direct volume ray casting that replaces regularly spaced sample positions with random offsets within each segment to eliminate banding and "wood-grain" artifacts [S1][S2, unofficial]. Unlike [Uniform Ray Sampling](1261-uniform-ray-sampling.md), which produces visible structural artifacts due to aliasing, Stochastic Jittering transforms these artifacts into high-frequency noise that is visually less noticeable [S1][S2, unofficial]. It differs from [Monte Carlo Volume Rendering](2060-monte-carlo-volume-rendering.md) in that it typically applies jitter only to the sample positions along the ray rather than using complex importance sampling or stratification for the entire integral, though it is fundamentally a Monte Carlo approach [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin ($ray\_orig$), ray direction ($ray\_dir$), entry point $t_0$ ($isect.t0$), step size ($step\_size$), sample index ($n$) [S2, unofficial].
- Output: Sample position ($sample\_pos$) [S2, unofficial].

**Data Structures:**
- `Noise`: A boolean flag to enable or disable the technique [S1].
- `MaxNoiseFactor`: A scalar controlling the maximum magnitude of the jitter [S1].
- `NoiseTexture`: A 2D texture of size `NoiseTextureSize` × `NoiseTextureSize` used to generate spatially coherent jitter values [S1].

**Steps:**
1. Generate a random value $r$ uniformly distributed in $[0, 1]$ [S2, unofficial]. In GPU implementations, this is typically sampled from a noise texture using screen coordinates [S1][S3, unofficial].
2. Calculate the sample position $t$ along the ray:
   $$ t = isect.t0 + step\_size \times (n + r) $$
   where $n$ is the integer sample index ($0, 1, 2, \dots$) [S2, unofficial].
3. Compute the 3D sample position:
   $$ sample\_pos = ray\_orig + t \times ray\_dir $$
   [S2, unofficial].

**Parameters:**
- `MaxNoiseFactor`: Typical value $0.75$ [S1]. This scales the random offset $r$ to bound the maximum inserted jitter magnitude [S1]. Standard stochastic sampling uses $r \in [0,1]$ directly, while VTK implementations scale the noise factor [S1][S2, unofficial].
- `NoiseTextureSize`: Typical value $64$ [S1]. Determines the resolution of the pre-generated noise texture [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash2d(x, y):
    """Deterministic pseudo-random value in [0, 1) for screen-space coherence [G]."""
    return (math.sin(x * 12.9898 + y * 78.233) * 43758.5453) % 1.0

def get_sample_position(isect_t0, step_size, n, ray_orig, ray_dir, use_noise, max_noise_factor, frag_coord):
    """
    Calculates the sample position using stochastic jittering.
    
    Args:
        isect_t0: Entry point t0 of the ray into the volume.
        step_size: Distance between regular sample intervals.
        n: Integer sample index (0, 1, 2...).
        ray_orig: Ray origin vector.
        ray_dir: Ray direction vector.
        use_noise: Boolean flag to enable jittering.
        max_noise_factor: Maximum magnitude of the jitter offset.
        frag_coord: Screen-space coordinates (x, y) for texture lookup.
        
    Returns:
        sample_pos: The 3D position of the sample.
    """
    if use_noise:
        # Sample noise texture at screen coordinates, scaled by texture size [S1][S3, unofficial]
        r = hash2d(frag_coord[0], frag_coord[1])
        # Scale by maximum noise factor if the implementation requires bounded jitter [S1]
        r = r * max_noise_factor
        t = isect_t0 + step_size * (n + r)
    else:
        # Standard uniform sampling at the center of the segment [S2, unofficial]
        t = isect_t0 + step_size * (n + 0.5)
    sample_pos = ray_orig + t * ray_dir
    return sample_pos
```

**Edge Cases and Pitfalls:**
- Without refinement, isosurface rendering errors are inevitable if the sample distance is not reduced; jittering only makes these artifacts less visible [S1].
- Replaces banding artifacts with noise. The noise can be reduced using quasi-Monte Carlo methods, but this is not implemented in the basic version [S2, unofficial].
- Precision issues may arise if `step_size` is very small and `isect_t0` is large, potentially causing loss of precision in the addition `isect_t0 + step_size * (n + r)` [G].

## Subtopics

### Slice Artifact Reduction
A variant that combines jittering with a solidity check to reduce "wood grain" artifacts caused by visible slice boundaries [S3, unofficial]. It jitters the ray start position along the view direction by adding `stepVector * texture(Noise, gl_FragCoord.xy / 256).r` [S3, unofficial]. Inside the raycasting loop, if the voxel value $V$ exceeds a threshold, it performs a solidity check: it moves back by half-step ($s = -stepVector \times 0.5$), samples $V$ [S3, unofficial]. If $V > Threshold$, it moves forward by quarter-step ($s *= 0.5$), else backward ($s *= -0.5$), and samples again [S3, unofficial]. Lighting proceeds only if the re-sampled $V$ is still $> Threshold$ [S3, unofficial]. This method is chosen when high-frequency slice boundaries are visible despite adequate sample density [S3, unofficial].

### Temporal Filtering and Jittering
A technique that smooths out noise and improves perceived volume resolution without extra raymarching steps [S4, unofficial]. It applies depth jittering each frame using a sub-voxel offset derived from a pseudo-random sequence (such as blue noise or Halton sequence) to offset ray positions [S4, unofficial]. A temporal reprojection compute pass then blends the current frame’s 3D texture with the previous frame's 3D texture using camera motion vectors [S4, unofficial]. This is chosen for real-time applications where spatial resolution is limited but temporal coherence can be exploited [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample for the jitter calculation, same as uniform sampling [S2, unofficial].
- **Measured Performance:** No specific hardware benchmarks provided in sources [S1][S2, unofficial].
- **Scaling:** Performance is independent of resolution, but visual quality improves with higher `NoiseTextureSize` or better random sequences [S1][S2, unofficial].
- **Strengths:** Eliminates banding artifacts; simple to implement; visually superior to uniform sampling for coarse steps [S1][S2, unofficial].
- **Limitations:** Introduces high-frequency noise; does not reduce isosurface errors without refinement; noise may be distracting in some contexts [S1][S2, unofficial].
- **When to use:** When banding artifacts are visible due to coarse sampling or regular grid alignment [S2, unofficial].
- **When not to use:** When strict quantitative accuracy is required, or when temporal stability is critical without temporal filtering [S2, unofficial][S4, unofficial].

## Background
Stochastic jittering was introduced as a technique to mitigate banding artifacts in volume rendering by adding controlled noise to ray starting points [S1]. It is implemented in the VTK library (`vtkGPUVolumeRayCastMapper2`) under the nomenclature "Noise" to maintain compatibility with the original class [S1]. The method is described as a Monte Carlo technique that samples at non-uniformly spaced locations [S2, unofficial]. It is often used in conjunction with isosurface rendering refinement to hide errors when sample distances are large [S1].

## Sources
[S1] Adding Refined Isosurface Rendering and Shadow Mapping to vtkGPUVolumeRayCastMapper, https://insight-journal.org/pdfs/880.pdf, fetched 2026-10-27
[S2] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S3] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
[S4] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
