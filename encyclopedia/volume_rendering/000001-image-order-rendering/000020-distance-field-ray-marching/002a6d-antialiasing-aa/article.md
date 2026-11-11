# Antialiasing (AA)

Also called: Aliasing Prevention, Procedural Texture Anti-Aliasing, Subpixel Sampling

## Overview
Antialiasing (AA) in distance-field ray marching prevents moiré artifacts and jagged edges caused by high-frequency noise octaves or fine geometric details aliasing against the pixel grid [S2]. It differs from standard supersampling by often integrating frequency-aware logic directly into the procedural evaluation or ray differential calculation, rather than relying solely on brute-force pixel subdivision [S2].

## How it works
### Inputs and Outputs
*   **Inputs**: Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$, pixel coordinates, scene SDF, procedural noise parameters (octaves, frequency), and AA configuration (samples, mode) [G].
*   **Outputs**: Final pixel color, free of high-frequency aliasing artifacts [G].

### Data Structures
*   **Ray Offsets**: A 2D array or vector of $(xx, yy)$ offsets representing subpixel jitter positions within the pixel footprint [G].
*   **LOD State**: Integer count of active noise octaves, derived from pixel size or ray differentials [G].

### Steps

1.  **Procedural Frequency Capping (LOD)**:
    *   Calculate the pixel size in world units, $s_{pixel}$ [S2].
    *   Determine the maximum octave count:
        $$ n_{oct} = \min(N_{max}, \lfloor \log_2(s_{pixel}) \rfloor) $$
        where $N_{max}$ is the global maximum octaves [S2].
    *   Evaluate the noise function using only $n_{oct}$ octaves. This skips frequencies finer than the pixel resolution, preventing moiré patterns [S2].
    *   *Alternative Analytical Filtering*: For simple patterns, apply a smoothstep filter based on the derivative of the pattern coordinate $u$:
        $$ f_{filtered} = \text{smoothstep}(-f_w, f_w, f(u)) $$
        where $f_w = fwidth(uv)$ is the rate of change of $u$ across the pixel [S2].

2.  **Subpixel Jittering**:
    *   Define a grid of jitter offsets. A standard configuration uses 2 samples in $x$ and 4 samples in $y$, totaling 8 samples per pixel [S1, unofficial].
    *   The offsets $xx$ and $yy$ are distributed within the pixel space: $xx \in \{-0.25, 0.25\}$ and $yy \in \{-0.75, -0.25, 0.25, 0.75\}$ [S1, unofficial].
    *   For each sample $(i, j)$:
        a.  Adjust the ray direction $\mathbf{d}'$ by adding the offset $(xx_i, yy_j)$ to the ray's origin or direction vector, effectively shifting the ray's entry point into the pixel footprint [S1, unofficial].
        b.  Perform standard sphere tracing using the adjusted ray [S1, unofficial].
        c.  Store the resulting color $C_{ij}$ [S1, unofficial].

3.  **Aggregation**:
    *   Average the colors from all samples:
        $$ C_{final} = \frac{1}{N_{samples}} \sum_{i,j} C_{ij} $$
    *   For 8 samples, multiply the sum by $0.125$ ($1/8$) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_pixel_aa(ray_o, ray_d, pixel_size, max_octaves, noise_func, use_aa=True):
    """
    Renders a single pixel with antialiasing.
    :param ray_o: Camera origin (vec3)
    :param ray_d: Normalized ray direction (vec3)
    :param pixel_size: World units per pixel (float)
    :param max_octaves: Maximum noise octaves (int)
    :param noise_func: Function evaluating procedural noise (vec3 -> float)
    :param use_aa: Boolean flag for subpixel sampling
    :return: Final color (vec3)
    """
    
    # 1. Procedural LOD Calculation
    # Skip octaves finer than pixel size to prevent moire
    if pixel_size > 0:
        lod_octaves = min(max_octaves, int(math.log2(pixel_size)))
    else:
        lod_octaves = max_octaves

    if not use_aa:
        # Single sample, no jitter
        return sphere_trace(ray_o, ray_d, lod_octaves, noise_func)

    # 2. Subpixel Sampling Configuration
    # 2 samples in x, 4 samples in y -> 8 total samples
    x_offsets = [-0.25, 0.25]
    y_offsets = [-0.75, -0.25, 0.25, 0.75]
    
    total_samples = len(x_offsets) * len(y_offsets)
    accumulated_color = vec3(0.0)

    # 3. Loop over samples
    for xx in x_offsets:
        for yy in y_offsets:
            # Adjust ray direction/origin based on offset
            # Note: Exact adjustment depends on camera projection model.
            # Here we assume a simple offset addition to direction for jittering
            jittered_d = normalize(ray_d + vec3(xx, yy, 0.0))
            
            # Perform sphere trace with LOD-aware noise
            sample_color = sphere_trace(ray_o, jittered_d, lod_octaves, noise_func)
            accumulated_color += sample_color

    # 4. Average results
    return accumulated_color * 0.125

def sphere_trace(ray_o, ray_d, lod_octaves, noise_func):
    """
    Standard sphere tracing loop.
    """
    t = 0.0
    max_dist = 100.0
    epsilon = 0.001
    
    while t < max_dist:
        p = ray_o + t * ray_d
        dist = noise_func(p, lod_octaves) # Noise function respects lod_octaves
        
        if dist < epsilon:
            return vec3(1.0) # Surface hit shading [G]
        if dist < 0: # Inside geometry
            return vec3(0.0) # Inside geometry shading [G]
            
        t += dist
        
    return vec3(0.0) # Background color [G]
```

### Parameters
*   **AA Samples**: 8 samples per pixel (2x4 grid) is a typical configuration [S1, unofficial].
*   **LOD Octaves**: Determined dynamically by `int(log2(pixel_size))` [S2].
*   **Jitter Range**: $xx \in \{-0.25, 0.25\}$ and $yy \in \{-0.75, -0.25, 0.25, 0.75\}$ with step $0.5$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Zero Pixel Size**: If `pixel_size` is 0 or negative, `log2` is undefined. Guard against this by clamping `lod_octaves` to `max_octaves` [S2].
*   **Derivative Calculation**: Analytical filtering requires accurate `fwidth` (derivative) computation. In ray marching, this requires computing ray differentials $\frac{\partial \mathbf{r}}{\partial u}$ and $\frac{\partial \mathbf{r}}{\partial v}$ manually, as standard texture derivatives may not apply to procedural SDFs [S2].
*   **Performance**: Subpixel sampling increases cost linearly with sample count. 8 samples is a balance; higher counts (e.g., 16+) may be needed for high-frequency scenes but significantly reduce frame rate [S1, unofficial].

## Subtopics

### Super-Sampling Anti-Aliasing (SSAA)
SSAA renders the entire scene at a higher resolution and downsamples to the target resolution [S3, unofficial]. It reduces aliasing by averaging multiple subpixel samples via interpolation [S3, unofficial].
*   **When to choose**: When rendering complex scenes with multiple buffers (albedo, depth, normal) and hardware acceleration for interpolation is available [S3, unofficial]. It operates on the final image buffer rather than per-pixel jittering [S3, unofficial].
*   **Implementation**: Compute high-res dimensions $h = \text{make\_divisible}(h_0 \cdot ssaa, 8)$ and $w = \text{make\_divisible}(w_0 \cdot ssaa, 8)$ [S3, unofficial]. Render all buffers at $(h, w)$, then downsample using `torch.nn.functional.interpolate` with `mode='min'` (bilinear/bicubic minification) [S3, unofficial].
*   **Parameters**: `ssaa` integer multiplier, default 1. Must be $>1$ to trigger [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: Scales linearly with the number of AA samples [S1, unofficial]. LOD capping reduces evaluation cost for distant objects by skipping noise octaves [S2].
*   **Memory Complexity**: Scales with the render resolution for SSAA [S3, unofficial].
*   **Strengths**: Effectively eliminates moiré from procedural noise via LOD capping [S2] and jagged edges from geometry via subpixel sampling [S1, unofficial].
*   **Limitations**: Subpixel sampling increases computational cost linearly with sample count [S1, unofficial]. Procedural LOD requires careful derivative management for analytical filtering [S2].
*   **When to use**: Use LOD capping for procedural noise. Use subpixel jittering for geometric AA when performance allows. Use SSAA when post-processing downsampling is feasible and high-quality anti-aliasing is required for all buffers [S3, unofficial].

## Background
Antialiasing in ray marching evolved alongside procedural rendering. Early implementations used simple supersampling [S1, unofficial]. The integration of LOD-based octave capping for procedural noise was developed to address specific moiré artifacts in ridged noise and Voronoi patterns, which are particularly visible on axis-aligned surfaces [S2]. SSAA in the context of neural and implicit rendering adapts traditional rasterization downsampling techniques to volume rendering pipelines [S3, unofficial].

## Sources
[S1] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S3] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
