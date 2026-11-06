# Antialiasing (AA)

Also called: Aliasing Prevention, Procedural Texture Anti-Aliasing, Subpixel Sampling

## Overview
Antialiasing (AA) in distance-field ray marching prevents moiré artifacts and aliasing caused by high-frequency noise octaves or fine geometric details on distant surfaces [S2]. It differs from standard rasterization AA by operating in the procedural or ray-marched domain, often using subpixel sampling or frequency-aware LOD culling rather than simple post-process blurring [S2].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, pixel coordinates $(u, v)$, scene SDF $f(\mathbf{p})$, maximum octaves $N_{max}$, and pixel size $s_{pix}$ (or derivative information).
**Outputs:** Final pixel color $C$.

**Data Structures:**
*   **Ray Offsets:** A set of 2D offset vectors $\{\mathbf{offset}_i\}_{i=0}^{N_s-1}$ representing subpixel positions within the pixel footprint.
*   **Sample Accumulator:** A color buffer $C_{acc}$ initialized to zero.

**Steps:**
1.  **Calculate Subpixel Offsets:** Determine the ray direction perturbations for each sample. For a grid-based approach, offsets are derived from a grid spanning the pixel area [S1, unofficial].
2.  **Determine Active Octaves:** Calculate the number of noise octaves to evaluate based on the pixel size to avoid sampling frequencies higher than the Nyquist limit [S2].
    $$ N_{active} = \min(N_{max}, \lfloor \log_2(s_{pix}) \rfloor) $$
    If $N_{active} < 0$, set $N_{active} = 0$ [S2].
3.  **Loop Over Samples:** For each sample index $i$ from $0$ to $N_s-1$:
    a. **Perturb Ray:** Adjust the ray direction $\mathbf{d}'$ by adding the subpixel offset $\mathbf{offset}_i$ to the ray origin or direction vector [S1, unofficial].
    b. **Ray March:** Perform sphere tracing using $f(\mathbf{p})$ with $N_{active}$ octaves for procedural textures or standard SDF evaluation [S2].
    c. **Shade:** Compute the color $C_i$ for the hit point or volume contribution.
    d. **Accumulate:** Add $C_i$ to $C_{acc}$.
4.  **Average:** Compute the final color by dividing the accumulator by the number of samples $N_s$ [S1, unofficial].
    $$ C = \frac{1}{N_s} \sum_{i=0}^{N_s-1} C_i $$

**Parameters:**
*   **AA Samples:** A 2x4 grid yields 8 samples per pixel [S1, unofficial].
*   **Offset Range:** Offsets typically range from -0.25 to 0.75 or similar subpixel coordinates [S1, unofficial].

**Edge Cases:**
*   **Distant Surfaces:** When $s_{pix}$ is large, $\log_2(s_{pix})$ may exceed $N_{max}$, culling all but the coarsest octaves [S2].
*   **Simple Patterns:** For non-noise patterns, analytical filtering using `smoothstep` with `fwidth` (derivative-based width) is preferred over sampling [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel_aa(ray_o, ray_d, pixel_size, max_octaves, sdf, noise_func):
    # 1. Define subpixel offsets for 2x4 grid (8 samples)
    # Offsets are relative to the pixel center or corner, scaled appropriately
    offsets = []
    for yy in [-0.75, -0.25, 0.25, 0.75]:
        for xx in [-0.25, 0.25]:
            offsets.append((xx, yy))
            
    num_samples = len(offsets)
    acc_color = vec3(0.0)
    
    # 2. Determine active octaves based on pixel size
    # Skip octaves finer than the pixel size to prevent aliasing
    if pixel_size > 0:
        active_octaves = min(max_octaves, int(log2(pixel_size)))
    else:
        active_octaves = max_octaves
        
    # 3. Loop over samples
    for xx, yy in offsets:
        # Adjust ray direction by offsets [S1, unofficial]
        # Note: Exact perturbation method depends on camera model.
        # Here we assume a simple direction perturbation or origin shift.
        perturbed_d = ray_d + vec3(xx, yy, 0.0) # Placeholder for specific camera math
        perturbed_d = normalize(perturbed_d)
        
        # Ray march with limited octaves for procedural parts [S2]
        color = ray_march_with_lod(ray_o, perturbed_d, sdf, noise_func, active_octaves)
        acc_color += color
        
    # 4. Average results
    final_color = acc_color * (1.0 / num_samples)
    return final_color
```

## Subtopics

### Super-Sampling Anti-Aliasing (SSAA)
SSAA renders the scene at a higher resolution and downsamples to the target resolution [S3, unofficial]. It is chosen when full scene consistency is required across all buffers (albedo, depth, normals) rather than just procedural noise [S3, unofficial]. It involves computing high-res dimensions, rendering all buffers, and minifying using interpolation (e.g., bilinear/bicubic) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_s \cdot N_{steps})$, where $N_s$ is the number of AA samples (e.g., 8) and $N_{steps}$ is the average ray march steps.
*   **Memory Complexity:** $O(1)$ additional memory per pixel for accumulation.
*   **Performance:** 8x slower than single-sample ray marching [S1, unofficial].
*   **Strengths:** Effectively removes moiré from procedural noise and high-frequency details [S2].
*   **Limitations:** Computationally expensive; SSAA requires storing and processing multiple buffers [S3, unofficial].

## Background
Antialiasing in ray marching is critical for procedural textures where high-frequency noise (e.g., ridged noise, Voronoi) creates aliasing artifacts on distant surfaces [S2]. Techniques like LOD-based octave culling and subpixel sampling were developed to address these specific artifacts in real-time and offline rendering contexts [S2]. SSAA is a broader technique applied to ray-traced or ray-marched scenes to ensure spatial consistency [S3, unofficial].

## Sources
[S1] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S3] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
