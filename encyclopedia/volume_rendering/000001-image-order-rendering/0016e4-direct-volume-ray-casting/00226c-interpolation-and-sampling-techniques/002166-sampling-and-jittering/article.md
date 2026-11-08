# Sampling and Jittering

Also called: Jittered Ray Casting, Jittered Sampling Volume Rendering, RayCaster, Volume Ray Casting with Jittered Sampling and Front-to-Back Blending

## Overview
Sampling and Jittering is a direct volume rendering technique that casts rays from the camera through a 3D voxel grid, samples voxel data with jittered step offsets to reduce banding artifacts, and composites the result using front-to-back alpha blending [S1, unofficial]. It perturbs sample positions along each ray using a pre-computed jitter texture to break up regular grid patterns [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D voxel texture (`voxel_sampler`), a 2D jitter texture (`jitter_tex_sampler`), camera parameters (`camera_position`, `ray_position_start`), volume bounds (`vol_extent_min`, `vol_extent_max`), and rendering parameters (`num_samples`, `ray_step_size`, `jitter_tex_size`, `compute_lighting`, lighting coefficients) [S1, unofficial].
*   **Outputs:** A final color value `dst` (vec4) for each fragment [S1, unofficial].

**Data Structures:**
*   `voxel_sampler`: A 3D texture containing scalar or RGBA voxel data [S1, unofficial].
*   `jitter_tex_sampler`: A 2D texture storing pre-computed jitter offsets; the red channel is sampled for the offset magnitude [S1, unofficial].
*   `dst`: Accumulator variable initialized to `(0.0, 0.0, 0.0, 0.0)` [S1, unofficial].

**Parameters:**
*   `num_samples`: Maximum number of samples to take per ray before the loop terminates [S1, unofficial].
*   `ray_step_size`: The distance between consecutive sample points along the ray [S1, unofficial].
*   `jitter_tex_size`: The width/height of the jitter texture used for sampling offsets [S1, unofficial].
*   `compute_lighting`: Boolean flag to enable or disable shading calculations (gradient estimation and Phong shading) [S1, unofficial].
*   `vol_extent_min`, `vol_extent_max`: The minimum and maximum bounding box coordinates of the volume in world space [S1, unofficial].
*   `k_Ambient`: Ambient lighting factor, value `(0.05, 0.05, 0.05)` [S1, unofficial].
*   `k_Diffuse`: Diffuse lighting factor, value `(1.0, 1.0, 1.0)` [S1, unofficial].
*   `k_Specular`: Specular lighting factor, value `(1.0, 1.0, 1.0)` [S1, unofficial].
*   `k_Shininess`: Specular shininess exponent, value `100.0` [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    Initialize the destination color accumulator:
    $$ \text{dst} = (0.0, 0.0, 0.0, 0.0) $$
    Calculate the ray direction:
    $$ \text{rayDir} = \text{normalize}(\text{rayPositionStart} - \text{cameraPosition}) $$
    Calculate the base step vector:
    $$ \text{rayStep} = \text{rayDir} \times \text{rayStepSize} $$

2.  **Ray Marching Loop:**
    Iterate $i$ from $0$ to $\text{numSamples} - 1$:

    a.  **Jitter Application:**
        Determine the 2D texture coordinate from fragment coordinates modulo `jitterTexSize`:
        $$ u = \text{fragmentCoord.x} \pmod{\text{jitterTexSize}} $$
        $$ v = \text{fragmentCoord.y} \pmod{\text{jitterTexSize}} $$
        Fetch the jitter value from the red channel of the jitter texture:
        $$ \text{jitterValue} = \text{jitterTexSampler}.sample(u, v).r $$
        Update the ray position:
        $$ \text{rayPosition} \leftarrow \text{rayPosition} + \text{rayStep} \times \text{jitterValue} $$

    b.  **Boundary Check:**
        Check if the current `rayPosition` is inside the volume bounds using sign functions and dot product [S1, unofficial]. Define:
        $$ \text{test1} = \text{sign}(\text{rayPosition} - \text{volExtentMin}) $$
        $$ \text{test2} = \text{sign}(\text{volExtentMax} - \text{rayPosition}) $$
        The point is inside if:
        $$ \text{dot}(\text{test1}, \text{test2}) \ge 3.0 $$
        If the point is outside, skip sampling for this iteration [S1, unofficial].

    c.  **Voxel Sampling:**
        Sample the voxel data at the current `rayPosition` from `voxel_sampler` into `src` [S1, unofficial]. Let $\text{src}.rgb$ be the color and $\text{src}.a$ be the opacity.

    d.  **Lighting (Optional):**
        If `computeLighting` is true, estimate the gradient normal using central differences on the alpha channel ($a$) of the voxel texture [S1, unofficial]. Use offset `ivec4(1, 0, 0, -1)` and permutations `(yxz, yzx, wyz, ywz, yzw)` to sample neighbors [S1, unofficial]. Calculate normal:
        $$ \text{normal} = \text{normalize}(\text{sample2} - \text{sample1}) $$
        If `normal` is NaN (e.g., if `sample1` equals `sample2`), skip shading [S1, unofficial]. Apply Phong shading to `src.rgb` using the shading function with ambient, diffuse, and specular components [S1, unofficial].

    e.  **Front-to-Back Blending:**
        Composite the sample into the destination:
        $$ \text{dst} = (1.0 - \text{dst}.a) \times \text{src} + \text{dst} $$

    f.  **Early Ray Termination:**
        If $\text{dst}.a > 0.95$, break the loop [S1, unofficial].

    g.  **Advance Ray:**
        Advance `rayPosition` by `rayStep` and re-evaluate the inside condition for the loop [S1, unofficial].

3.  **Output:**
    Output the final `dst` as `FragColor` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fragment(fragment_coord, camera_position, ray_position_start, ray_step_size, 
                    num_samples, jitter_tex_size, compute_lighting, 
                    vol_extent_min, vol_extent_max, light_position, 
                    voxel_sampler, jitter_tex_sampler):
    # Initialize destination accumulator
    dst = vec4(0.0, 0.0, 0.0, 0.0)
    
    # Calculate ray parameters
    ray_dir = normalize(ray_position_start - camera_position)
    ray_step = ray_dir * ray_step_size
    ray_position = ray_position_start
    
    # Pre-calculate jitter coordinate
    u = fragment_coord.x % jitter_tex_size
    v = fragment_coord.y % jitter_tex_size
    
    for i in range(num_samples):
        # 1. Apply Jitter Offset
        jitter_value = jitter_tex_sampler.sample(u, v).r
        ray_position = ray_position + ray_step * jitter_value
        
        # 2. Check Bounds
        if is_inside_volume(ray_position, vol_extent_min, vol_extent_max):
            # 3. Sample Voxel Data
            src = voxel_sampler.sample(ray_position)
            
            # 4. Lighting (Optional)
            if compute_lighting:
                # Estimate gradient normal using central differences on alpha channel
                # Source specifies offset ivec4(1, 0, 0, -1) and permutations (yxz, yzx, wyz, ywz, yzw)
                sample1 = voxel_sampler.sample(ray_position + vec3(1.0, 0.0, 0.0)).a
                sample2 = voxel_sampler.sample(ray_position - vec3(1.0, 0.0, 0.0)).a
                normal = normalize(sample2 - sample1)
                
                if not isnan(normal):
                    # Phong Shading
                    to_camera = normalize(camera_position - ray_position)
                    to_light = normalize(light_position - ray_position)
                    halfway = normalize(to_light + to_camera)
                    
                    diffuse_light = max(dot(to_light, normal), 0.0)
                    diffuse = vec3(1.0, 1.0, 1.0) * src.rgb * diffuse_light
                    
                    specular_light = pow(max(dot(halfway, normal), 0.0), 100.0)
                    specular = vec3(1.0, 1.0, 1.0) * src.rgb * specular_light
                    
                    ambient = vec3(0.05, 0.05, 0.05) * src.rgb
                    src.rgb = ambient + diffuse + specular
            
            # 5. Front-to-Back Blending
            dst = (1.0 - dst.a) * src + dst
            
            # 6. Early Ray Termination
            if dst.a > 0.95:
                break
                
        # 7. Advance Ray
        ray_position = ray_position + ray_step
        
    return dst

def is_inside_volume(pos, min_ext, max_ext):
    test1 = sign(pos - min_ext)
    test2 = sign(max_ext - pos)
    return dot(test1, test2) >= 3.0
```

## Subtopics

### [Stochastic Jittering](924-stochastic-jittering.md)
Stochastic jittering perturbs sample positions along rays using random noise to mitigate banding and aliasing artifacts caused by regular, undersampled grid structures. It sets itself apart from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by breaking the periodicity of sample placement, and from [Monte Carlo Volume Rendering](2060-monte-carlo-volume-rendering.md) by typically using a fixed, pre-computed noise texture rather than generating high-variance random numbers per sample, trading visible banding for less objectionable dither patterns.

### [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md)
Hierarchical Sampling (Inverse CDF Sampling) is a method for generating fine-grained depth samples along a ray in volume rendering by drawing from a probability distribution derived from coarse rendering weights. It solves the problem of inefficient uniform sampling in empty space or low-density regions by concentrating computational effort on high-contribution areas identified during a coarse pass. Unlike [Uniform Ray Sampling](1261-uniform-ray-sampling.md) which distributes samples evenly regardless of scene content, or [Stochastic Jittering](924-stochastic-jittering.md) which only adds noise to fixed positions, this method adaptively resamples based on the estimated density or opacity of the scene. It builds on [Sampling and Jittering](2516-sampling-and-jittering.md) by providing a mechanism to refine sample locations based on learned or estimated importance.

### [Uniform Ray Sampling](1261-uniform-ray-sampling.md)
Uniform Ray Sampling generates sample points along camera rays by distributing depths uniformly between a near clipping plane ($t_n$) and a far clipping plane ($t_f$). It assumes that every point along the ray contributes equally to the final rendering, implying a constant probability density function (PDF) and a linearly decaying transmittance. This method serves as the baseline for NeRF-style raymarching and is the simplest sampling strategy to implement, though it typically offers the lowest efficiency compared to spatial skipping or importance sampling methods because it does not adapt to scene geometry or density variations.

### [Monte Carlo Volume Rendering](2060-monte-carlo-volume-rendering.md)
Monte Carlo Volume Rendering is a stochastic approach to volumetric visualization that avoids uniform step sampling by using random distance sampling and null-collision techniques to estimate light transport in participating media. It differs from its siblings, such as [Uniform Ray Sampling](1261-uniform-ray-sampling.md) and [Ray Casting with Jittered Sampling](4151-ray-casting-with-jittered-sampling.md), by explicitly modeling scattering events via random walks (photon tracing) or stochastic ray casting with exponential distance sampling, allowing for accurate handling of anisotropic scattering and complex light transport at the cost of convergence noise.

### [Random Jitter with Reprojection](2782-random-jitter-with-reprojection.md)
Random Jitter with Reprojection is a technique to reduce banding artifacts caused by low sample counts in volume rendering by introducing stochastic jitter during ray marching and subsequently reducing the resulting noise via temporal accumulation. It distinguishes itself from pure spatial jittering methods by leveraging temporal coherence, projecting a pseudo-hit position from the current frame into the previous frame's screen space to blend the current result with a history buffer, thereby allowing for lower per-frame sample counts while maintaining visual stability.

### [Importance Sampling for SPH Rendering](3624-importance-sampling-for-sph-rendering.md)
Importance Sampling for SPH Rendering is a technique to increase rendering performance by reducing the number of particles used during rendering, using a non-uniform sampling strategy based on data importance and view-dependent criteria. It solves the problem of high computational cost when rendering large, unstructured Smoothed Particle Hydrodynamics (SPH) datasets by ensuring that particles in complex regions (e.g., phase interfaces, vortices) are sampled more densely than those in homogeneous areas. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by avoiding equal-time high-error trade-offs.

### [Ray Casting with Jittered Sampling](4151-ray-casting-with-jittered-sampling.md)
Ray Casting with Jittered Sampling is a direct volume rendering technique that casts rays through a 3D voxel grid to accumulate color and opacity, using jittered sampling offsets to reduce aliasing artifacts. It distinguishes itself from uniform ray sampling by perturbing sample positions to break up regular grid patterns, and from stochastic methods by using a deterministic, pre-computed jitter texture rather than runtime random number generation.

### [Warp-based Sampling](4298-warp-based-sampling.md)
Warp-based Sampling is a GPU optimization strategy for direct volume ray casting that improves texture cache performance by reorganizing thread warps to align sampling directions with the volume's memory strides. Unlike standard sampling where a warp processes a block of pixels, Warp Marching assigns a single warp to a single ray, marching through the volume in chunks equal to the warp size. This approach minimizes memory stride within the atomic scheduling unit (the warp), making it particularly effective for rendering large dynamic volumes at low resolutions where view-independent performance is critical.

### Resampling Techniques
Resampling techniques align geometry and color between scenes along corresponding rays [S2]. Uniform sampling points are distributed evenly along the ray but can lead to misalignment [S2]. An alternative strategy validates points only if density exceeds a threshold in both scenes without resampling [S2]. Density-based resampling identifies density peaks along each ray and adjusts sampling to align key points in both scenes, improving consistency by aligning initial object intersections [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of fragments (pixels) and $S$ is the number of samples per ray (`numSamples`) [S1, unofficial]. Early ray termination can reduce the effective $S$ [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the voxel texture and $O(J^2)$ for the jitter texture, where $V$ is the volume size and $J$ is the jitter texture size [S1, unofficial].
*   **Measured Performance:** Performance depends heavily on `numSamples` and early ray termination efficiency [S1, unofficial].
*   **Strengths:** Simple to implement; reduces banding artifacts compared to uniform sampling; deterministic if using a fixed jitter texture; supports shading [S1, unofficial].
*   **Limitations:** Early ray termination may miss faint structures behind dense ones [S1, unofficial]; gradient calculation may result in NaN if sample values are identical [S1, unofficial]; jitter texture size limits the variety of jitter patterns [S1, unofficial].
*   **When to Use:** Real-time volume rendering where banding artifacts are visible with uniform sampling and deterministic results are preferred [S1, unofficial].
*   **When Not to Use:** When accurate light transport simulation is required (use Monte Carlo); when adaptive sampling based on density is needed (use Hierarchical Sampling) [S1, unofficial].

## Background
Sampling and Jittering is a direct volume rendering technique described in the context of the VoxViz project [S1, unofficial]. It builds on the principles of ray casting and front-to-back blending, incorporating jittering to mitigate aliasing [S1, unofficial]. The method is part of the broader family of interpolation and sampling techniques in volume rendering [S1, unofficial].

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S2] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
