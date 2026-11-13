# Stochastic Jittering

Also called: Ray Jittering, Jittered Ray Sampling, Stochastic Sample Jittering

## Overview
Stochastic jittering is a sampling technique used in direct volume ray casting to mitigate banding and quantization artifacts caused by regular, uniformly spaced sample positions [S1, unofficial][S6, unofficial]. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by perturbing sample locations with random offsets, transforming structured aliasing into less visually noticeable noise [S1, unofficial][S6, unofficial]. It is particularly effective when using low sample counts or nearest-neighbor interpolation, where regular grid patterns are most visible [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Base step size $\Delta t$.
- A pre-computed jitter texture (2D noise texture).
- Pixel coordinates $(u, v)$ or screen-space position.
- Maximum noise factor $\alpha$ (optional, controls jitter magnitude).

**Outputs:**
- Perturbed sample positions along the ray.

**Data Structures:**
- **Jitter Texture:** A 2D texture storing noise values.
  - Dimensions: $N \times N$ (e.g., $64 \times 64$) [S5][S7, unofficial].
  - Format: Single channel (e.g., Red) or grayscale [S4, unofficial].
  - Filtering: `GL_NEAREST` (or equivalent) to preserve sharp noise patterns without interpolation [S3, unofficial].
  - Wrapping: `GL_REPEAT` to allow seamless tiling [S3, unofficial].
  - Source: Often blue noise for better perceptual quality [S7, unofficial].

**Steps:**
1. **Initialize Ray:** Start ray marching from the near plane or volume entry point. Let $t_{start}$ be the initial parameter along the ray [S6, unofficial].
2. **Loop for Samples:** For each sample index $n = 0, 1, 2, \dots$:
   a. **Compute Base Position:** Calculate the regular sample position:
      $$ t_{base} = t_{start} + n \cdot \Delta t $$
      $$ \mathbf{p}_{base} = \mathbf{o} + t_{base} \cdot \mathbf{d} $$
   b. **Fetch Jitter Value:**
      i. Compute texture coordinates from pixel coordinates:
         $$ u_{tex} = u \pmod N $$
         $$ v_{tex} = v \pmod N $$
      ii. Sample the jitter texture to get a value $j \in [0, 1]$. If using blue noise, $j$ is drawn from a blue noise distribution [S7, unofficial].
   c. **Apply Jitter:**
      i. Scale the jitter by the step size and optionally by a maximum noise factor $\alpha$:
         $$ \delta t = j \cdot \Delta t \cdot \alpha $$
         If $\alpha$ is not used, $\delta t = j \cdot \Delta t$ [S4, unofficial][S5].
      ii. Offset the sample position:
          $$ t_{sample} = t_{base} + \delta t $$
          $$ \mathbf{p}_{sample} = \mathbf{o} + t_{sample} \cdot \mathbf{d} $$
      *Note:* Some implementations add the jitter to the start of the segment rather than the center, effectively picking a random point within the segment $[t_{base}, t_{base} + \Delta t]$ [S1, unofficial][S6, unofficial]. The formula $t_{sample} = t_{start} + (n + j) \cdot \Delta t$ achieves this [S6, unofficial].
   d. **Sample Volume:** Trilinearly or nearest-neighbor sample the volume data at $\mathbf{p}_{sample}$ to get color and opacity.
   e. **Composite:** Accumulate color and opacity using front-to-back or back-to-front compositing.
   f. **Advance:** If the ray exits the volume or opacity threshold is reached, break. Otherwise, continue to next $n$.

**Parameters:**
- **Jitter Texture Size ($N$):** Typical value: 64 [S5]. Larger sizes provide more unique noise patterns per frame but increase memory bandwidth [S8, book].
- **Max Noise Factor ($\alpha$):** Typical value: 0.75 [S5]. Controls the maximum displacement relative to step size. A value of 1.0 allows jitter across the entire step [S1, unofficial].
- **Noise Type:** Blue noise is preferred over white noise because it is tileable and easier to blur out, reducing visible dither patterns [S7, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stochastic_jittered_ray_cast(ray_origin, ray_dir, volume_data, step_size, jitter_texture, pixel_uv, max_noise_factor=1.0):
    """
    Casts a ray through a volume using stochastic jittering to reduce banding.
    
    Args:
        ray_origin: vec3, origin of the ray.
        ray_dir: vec3, normalized direction of the ray.
        volume_data: function, samples volume at (x,y,z) returning (color, opacity).
        step_size: float, base distance between samples.
        jitter_texture: function, samples 2D noise texture at (u,v) returning float in [0,1].
        pixel_uv: vec2, screen-space coordinates of the pixel.
        max_noise_factor: float, multiplier for jitter magnitude (default 1.0).
        
    Returns:
        vec4, final accumulated color with alpha.
    """
    # Assume ray enters volume at t_near and exits at t_far
    t_near = compute_ray_volume_intersection(ray_origin, ray_dir).t_near
    t_far = compute_ray_volume_intersection(ray_origin, ray_dir).t_far
    
    accumulated_color = vec4(0.0)
    accumulated_opacity = 0.0
    transmission = 1.0
    
    # Get jitter value based on pixel coordinates
    # Use modulo to handle texture wrapping if needed, or assume texture is large enough
    u = pixel_uv.x % jitter_texture.width
    v = pixel_uv.y % jitter_texture.height
    jitter_val = jitter_texture.sample(u, v)
    
    # Scale jitter by step size and max noise factor
    jitter_offset = jitter_val * step_size * max_noise_factor
    
    t = t_near
    sample_index = 0
    
    while t < t_far and transmission > 0.01: # Break threshold
        # Calculate sample position with jitter
        # Method 1: Offset from regular center position
        # t_regular = t_near + step_size * (sample_index + 0.5)
        # t_sample = t_regular + jitter_offset - (step_size * 0.5) # Adjust to keep within segment?
        
        # Method 2: Random point within segment [t, t + step_size]
        # This is equivalent to t = t_near + step_size * (sample_index + rand())
        t_sample = t_near + step_size * (sample_index + jitter_val)
        
        sample_pos = ray_origin + t_sample * ray_dir
        
        # Sample volume
        color, opacity = volume_data(sample_pos)
        
        # Front-to-back compositing
        alpha = opacity * transmission
        accumulated_color += color * alpha
        accumulated_opacity += alpha
        transmission -= alpha
        
        sample_index += 1
        t = t_near + sample_index * step_size # Advance base position
        
    return vec4(accumulated_color, accumulated_opacity)
```

**Edge Cases and Pitfalls:**
- **Banding vs. Noise:** Jittering replaces visible banding artifacts with noise. While visually more pleasing, it introduces noise that may require further reduction techniques (e.g., temporal filtering) [S1, unofficial][S6, unofficial].
- **Precision:** At high sampling rates, the benefit of jittering vanishes as the regular sampling is sufficient to capture volume details [S2, unofficial].
- **Isosurface Errors:** Without refinement, isosurface rendering errors are inevitable if sample distance is not reduced; jittering makes these artifacts less visible but does not eliminate them [S5].
- **Texture Bandwidth:** Using a large jitter texture can increase memory bandwidth requirements [S8, book].

## Subtopics

### [Slice Artifact Reduction](1628-slice-artifact-reduction.md)
Slice Artifact Reduction is a stochastic sampling technique used in direct volume ray casting to mitigate "wood grain" artifacts—visible slice boundaries caused by regular ray start positions and discrete voxel grids. It differs from standard ray casting by randomizing the initial ray origin along the viewing direction and performing a localized "solidity check" to verify that a sampled voxel is truly part of the volume rather than an edge artifact. This method is particularly effective when using a low number of ray casting steps or when nearest-neighbor interpolation is employed, where regular sampling patterns are most visible.

### [Temporal Filtering and Jittering](2762-temporal-filtering-and-jittering.md)
Temporal Filtering and Jittering is a technique that smooths out noise and improves perceived volume resolution without extra raymarching steps. It addresses spatial banding artifacts caused by coarse slice resolution by combining sub-voxel depth jittering with temporal reprojection. It distinguishes itself from static spatial filters by leveraging camera motion vectors to blend frames, and from pure spatial sampling methods by accumulating jittered samples over time to improve perceived resolution without additional raymarching steps.

### [Random Jitter with Reprojection](2782-random-jitter-with-reprojection.md)
Random Jitter with Reprojection is a technique to reduce banding artifacts caused by low sample counts in volume rendering by introducing stochastic jitter during ray marching and subsequently reducing the resulting noise via temporal accumulation. It distinguishes itself from pure spatial jittering methods by leveraging temporal coherence, projecting a pseudo-hit position from the current frame into the previous frame's screen space to blend the current result with a history buffer, thereby allowing for lower per-frame sample counts while maintaining visual stability.

### [Ray Casting with Jittered Sampling](4151-ray-casting-with-jittered-sampling.md)
Ray Casting with Jittered Sampling is a direct volume rendering technique that casts rays through a 3D voxel grid to accumulate color and opacity along the ray path. It distinguishes itself from uniform ray sampling by perturbing sample positions with a pre-computed jitter texture to break up regular grid patterns and reduce aliasing artifacts. Unlike stochastic methods, it uses deterministic offsets stored in a 2D texture rather than runtime random number generation [G].

### Low-Resolution Stochastic Sampling
An extension where sampling occurs at a lower resolution to reduce texture bandwidth requirements, combined with varying the sample pattern with stochastic sampling to reduce regular pattern artifacts where sampling density is reduced [S8, book].

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. Jittering adds negligible overhead (texture fetch and addition) compared to the volume sampling itself.
- **Memory Complexity:** $O(N^2)$ for the jitter texture, where $N$ is the texture dimension (e.g., $64 \times 64$ floats = 16 KB) [S5].
- **Measured Performance:** On typical hardware, jittering has minimal impact on frame rate. The benefit is visual quality rather than speed [S2, unofficial].
- **Scaling:** The benefit of ray jittering vanishes with higher sampling rates (e.g., $s \ge 2.0$) [S2, unofficial]. It is most effective at low sampling rates (e.g., $s = 0.25, 0.5$) [S2, unofficial].
- **Strengths:** Effectively reduces banding and wood-grain artifacts; simple to implement; works well with nearest-neighbor interpolation.
- **Limitations:** Introduces noise; does not improve accuracy, only visual quality; ineffective at high sample counts.
- **When to Use:** When using low sample counts, nearest-neighbor interpolation, or when banding artifacts are prominent.
- **When Not to Use:** When high sample counts are available, or when noise is unacceptable without temporal filtering.

## Background
Stochastic jittering originated as a technique to address quantization errors in volume rendering. It is implemented in libraries such as VTK (Visualization Toolkit) under the name "Noise" [S5]. The technique is widely used in real-time volume rendering to improve visual quality without significant performance cost [S1, unofficial][S6, unofficial]. Blue noise textures for jittering are commonly sourced from libraries like 'free-blue-noise-textures' [S7, unofficial].

## Sources
[S1] "Volume Rendering for Developers" Group Read Part 1, https://www.youtube.com/watch?v=mZzMVD4LQvk, fetched 2026-10-26 (unofficial)
[S2] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
[S3] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
[S4] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S5] Adding Refined Isosurface Rendering and Shadow Mapping to vtkGPUVolumeRayCastMapper, https://insight-journal.org/pdfs/880.pdf, fetched 2026-10-27
[S6] Ray Marching: Getting it Right!, https://www.scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers//ray-marching-get-it-right.html, fetched 2026-10-28 (unofficial)
[S7] Efficient volumetric clouds - Resources - three.js forum, https://discourse.threejs.org/t/efficient-volumetric-clouds/66067, fetched 2026-10-29 (unofficial)
[S8] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
