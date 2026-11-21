# Sampling and Jittering
Also called: Jittered Ray Casting, Jittered Sampling Volume Rendering, RayCaster, Volume Ray Casting with Jittered Sampling and Front-to-Back Blending

## Overview
Sampling and Jittering is a direct volume ray casting technique that casts rays from the camera through the volume, samples voxel data with jittered step offsets to reduce banding, and composites the result using front-to-back alpha blending [S1, unofficial]. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by perturbing sample locations with random offsets, transforming structured aliasing into less visually noticeable noise [S1, unofficial]. It is particularly effective when using low sample counts or nearest-neighbor interpolation, where regular grid patterns are most visible [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position ($\mathbf{o}$), ray direction ($\mathbf{d}$), volume texture ($V$), jitter texture ($J$), volume bounding box ($[\mathbf{VolExtentMin}, \mathbf{VolExtentMax}]$), lighting parameters ($\mathbf{k}_{Ambient}, \mathbf{k}_{Diffuse}, \mathbf{k}_{Specular}, k_{Shininess}$), and control flags ($ComputeLighting$).
*   **Outputs:** Final fragment color ($\mathbf{dst}$).

**Data Structures:**
*   **Voxel Sampler:** A 3D texture storing scalar or RGBA volume data.
*   **Jitter Texture:** A 2D texture storing pre-computed random offsets. The width and height are defined by $JitterTexSize$. The red channel is used for the jitter value.
*   **Ray State:** Current position $\mathbf{p}$, direction $\mathbf{d}$, step size $s$.

**Parameters:**
*   $NumSamples$: Maximum iterations per ray.
*   $RayStepSize$: Distance between consecutive samples ($s$).
*   $JitterTexSize$: Dimension of the jitter texture.
*   $ComputeLighting$: Boolean flag for shading.
*   $\mathbf{k}_{Ambient} = (0.05, 0.05, 0.05)$
*   $\mathbf{k}_{Diffuse} = (1.0, 1.0, 1.0)$
*   $\mathbf{k}_{Specular} = (1.0, 1.0, 1.0)$
*   $k_{Shininess} = 100.0$

**Algorithm Steps:**
1.  **Initialization:** Set destination color $\mathbf{dst} = (0.0, 0.0, 0.0, 0.0)$. Calculate ray direction $\mathbf{d} = \text{normalize}(\mathbf{p}_{frag} - \mathbf{o})$. Calculate step vector $\mathbf{s} = \mathbf{d} \times RayStepSize$.
2.  **Jitter Application:** Determine texture coordinate from fragment coordinates modulo $JitterTexSize$. Fetch jitter value $j$ from the red channel of $J$. Update ray position: $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{s} \times j$.
3.  **Boundary Check:** Check if $\mathbf{p}$ is inside the volume bounds. Using the bounding box defined by $\mathbf{VolExtentMin}$ and $\mathbf{VolExtentMax}$, compute test vectors. If $\text{dot}(\mathbf{test1}, \mathbf{test2}) \ge 3.0$, the point is inside [S1, unofficial].
4.  **Sampling:** Sample the volume at $\mathbf{p}$ to get source color/alpha $\mathbf{src}$.
5.  **Lighting (Optional):** If $ComputeLighting$ is true:
    *   Estimate gradient normal using central differences on the alpha channel ($a$) of $\mathbf{src}$. Use offsets `ivec4(1, 0, 0, -1)` and permutations (yxz, yzx, wyz, ywz, yzw) to sample neighbors [S1, unofficial].
    *   Calculate $\mathbf{n} = \text{normalize}(\mathbf{sample2} - \mathbf{sample1})$. If $\mathbf{n}$ is NaN, skip shading [S1, unofficial].
    *   Apply Phong shading to $\mathbf{src}.rgb$ using ambient, diffuse, and specular components [S1, unofficial].
6.  **Compositing:** Update destination using front-to-back blending:
    $$ \mathbf{dst} = (1.0 - \mathbf{dst}.a) \times \mathbf{src} + \mathbf{dst} $$
7.  **Termination:** If $\mathbf{dst}.a > 0.95$, break the loop (early ray termination) [S1, unofficial].
8.  **Advance:** Update $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{s}$. Re-evaluate boundary check. Repeat until $NumSamples$ is reached or ray exits.
9.  **Output:** Return $\mathbf{dst}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_jittered_volume(ray_position, camera_position, ray_step_size, num_samples, 
                           jitter_tex, jitter_tex_size, voxel_tex, vol_extent_min, vol_extent_max,
                           compute_lighting, light_position):
    # 1. Initialize
    dst = vec4(0.0, 0.0, 0.0, 0.0)
    ray_dir = normalize(ray_position - camera_position)
    ray_step = ray_dir * ray_step_size
    
    # 2. Jitter offset application
    # Get 2D texture coordinate from fragment coordinates modulo JitterTexSize
    frag_coords = get_fragment_coords() # Implementation detail: derived from screen position
    jitter_uv = vec2(frag_coords.x % jitter_tex_size, frag_coords.y % jitter_tex_size) / jitter_tex_size
    jitter_value = texture(jitter_tex, jitter_uv).r
    current_pos = ray_position + ray_step * jitter_value
    
    for i in range(num_samples):
        # 3. Boundary Check
        # Check if current_pos is inside volume bounds
        # Using sign functions and dot product logic from source
        test1 = vec3(sign(current_pos - vol_extent_min))
        test2 = vec3(sign(vol_extent_max - current_pos))
        if dot(test1, test2) < 3.0:
            # Point is outside, advance and continue or break depending on implementation
            # Source implies re-evaluating inside condition for the loop
            current_pos += ray_step
            continue
            
        # 4. Sample Voxel Data
        src = texture(voxel_tex, current_pos)
        
        # 5. Lighting
        if compute_lighting:
            # Estimate gradient normal using central differences on alpha channel
            # Offsets: ivec4(1, 0, 0, -1) and permutations (yxz, yzx, wyz, ywz, yzw)
            # Note: Source implies specific neighbor sampling logic for gradient
            sample1 = texture(voxel_tex, current_pos + vec3(0.0, 0.0, 0.0)) # Placeholder for specific neighbor logic
            sample2 = texture(voxel_tex, current_pos + vec3(0.0, 0.0, 0.0)) # Placeholder for specific neighbor logic
            # Actual implementation requires fetching specific neighbors based on permutations
            # For the sake of self-contained pseudocode, we assume a gradient function exists
            normal = estimate_gradient_normal(voxel_tex, current_pos)
            
            if not isnan(normal):
                to_camera = normalize(camera_position - current_pos)
                to_light = normalize(light_position - current_pos)
                src.rgb = shading(src.rgb, normal, to_camera, to_light)
        
        # 6. Front-to-Back Blending
        dst = (1.0 - dst.a) * src + dst
        
        # 7. Early Ray Termination
        if dst.a > 0.95:
            break
            
        # 8. Advance Ray
        current_pos += ray_step
        
    return dst

def shading(rgb, normal, to_camera, to_light):
    halfway = normalize(to_light + to_camera)
    diffuse_light = max(dot(to_light, normal), 0.0)
    diffuse = vec3(0.0, 0.0, 0.0)
    specular = vec3(0.0, 0.0, 0.0)
    
    if diffuse_light > 0.0:
        diffuse = vec3(1.0, 1.0, 1.0) * rgb * diffuse_light # k_Diffuse
        specular_light = pow(max(dot(halfway, normal), 0.0), 100.0) # k_Shininess
        specular = vec3(1.0, 1.0, 1.0) * rgb * specular_light # k_Specular
        
    ambient = vec3(0.05, 0.05, 0.05) * rgb # k_Ambient
    return ambient + diffuse + specular
```

**Edge Cases and Pitfalls:**
*   **NaN Normals:** Gradient calculation may result in NaN if `sample1` equals `sample2` or if values are identical. This is handled by checking `isnan(normal)` and skipping shading [S1, unofficial].
*   **Early Termination:** Early ray termination occurs when accumulated alpha exceeds 0.95, potentially missing faint structures behind dense ones [S1, unofficial].

## Subtopics

### [Stochastic Jittering](924-stochastic-jittering.md)
Stochastic jittering is a sampling technique used in direct volume ray casting to mitigate banding and quantization artifacts caused by regular, uniformly spaced sample positions. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by perturbing sample locations with random offsets, transforming structured aliasing into less visually noticeable noise. It is particularly effective when using low sample counts or nearest-neighbor interpolation, where regular grid patterns are most visible.

### [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md)
Hierarchical sampling (Inverse CDF Sampling) is a method for generating importance samples along a ray in volume rendering by inverting the cumulative distribution function (CDF) of the rendering contribution, which is mathematically equivalent to the transmittance function. It solves the problem of efficiently placing sample points in regions of high density without redundant radiance queries, enabling differentiable end-to-end optimization of neural radiance fields. Unlike standard hierarchical sampling which samples from a piecewise constant probability density defined by coarse weights, this method uses inverse transform sampling on a piecewise approximated density field (constant or linear), allowing gradients to back-propagate through the sampling process to the density parameters. It builds on [Sampling and Jittering](2516-sampling-and-jittering.md) by providing a deterministic or stratified mechanism to concentrate fine samples where the coarse pass indicated high opacity.

### [Uniform Ray Sampling](1261-uniform-ray-sampling.md)
Uniform Ray Sampling generates sample points along camera rays at equal distances between a near and far clipping plane to discretize the volume for integration. It assumes every point along the ray contributes equally to the final rendering, implying a constant probability density function (PDF) and linearly decaying transmittance. This method sets itself apart from its siblings by using fixed, deterministic spacing without adaptive logic or stratification, making it the simplest to implement but often the least efficient for scenes with large empty regions. It serves as the baseline for NeRF-style rendering.

### [Slab-based Ray Sampling](2853-slab-based-ray-sampling.md)
Slab-based ray sampling is an image-order caching mechanism for direct volume ray casting that groups rays into rectangular screen regions to form stacked slabs, storing voxel samples in shared memory to optimize neighborhood access for lighting and compositing. Unlike object-order bricking, which struggles with GPU hardware mapping due to per-brick lists, this method leverages the regular structure of rays within a slab to reduce memory bandwidth and latency when calculating gradients or performing ambient occlusion. It is distinct from sibling methods like [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by explicitly managing spatial locality in shared memory across multiple rays, and from [Stochastic Jittering](924-stochastic-jittering.md) by prioritizing deterministic cache coherence over variance reduction.

### [Supersampling](3493-supersampling.md)
Supersampling is a technique to minimize aliasing artifacts by casting multiple rays per pixel and averaging the results. It handles high-frequency components in transfer functions by performing dense quadrature of the volume render integral, assuming linearity of the scalar field between sample points but not linearity of the integral itself. Unlike pre-integration, which requires equidistant sample points and fails for nonlinear integrals, supersampling accounts for the nonlinearity of the volume render integral while avoiding oversampling of the scalar field. It differs from [Stochastic Jittering](924-stochastic-jittering.md) by using deterministic, fixed offsets for low sample counts (1x–4x) rather than random or stratified patterns, and from [Monte Carlo Volume Rendering](2060-monte-carlo-volume-rendering.md) by relying on simple arithmetic averaging rather than importance-weighted integration.

### [Importance Sampling for SPH Rendering](3624-importance-sampling-for-sph-rendering.md)
Importance Sampling for SPH Rendering is a technique to increase rendering performance by reducing the number of particles used during rendering, using a non-uniform sampling strategy based on data importance and view-dependent criteria. It solves the problem of high computational cost when rendering large, unstructured Smoothed Particle Hydrodynamics (SPH) datasets by ensuring that particles in complex regions (e.g., phase interfaces, vortices) are sampled more densely than those in homogeneous areas. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by avoiding equal-time high-error trade-offs.

### [Epipolar Sampling](3899-epipolar-sampling.md)
Epipolar sampling is a technique for aggregating features from multiple views by sampling along epipolar lines rather than along the camera ray in 3D space. It solves the problem of scale ambiguity in 3D reconstruction from sparse views, where camera poses are scaled by an arbitrary factor, preventing the inference of metric depth from a single view. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths.

### [Warp-based Sampling](4298-warp-based-sampling.md)
Warp-based Sampling is a GPU optimization strategy for direct volume ray casting that improves texture cache performance by reorganizing thread warps to align sampling directions with the volume's memory strides. Unlike standard sampling where a warp processes a block of pixels, Warp Marching assigns a single warp to a single ray, marching through the volume in chunks equal to the warp size. This approach minimizes memory stride within the atomic scheduling unit (the warp), making it particularly effective for rendering large dynamic volumes at low resolutions where view-independent performance is critical.

### [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md)
Hierarchical sampling is a two-stage strategy for volume rendering that improves image quality by concentrating samples in regions of high geometric or color complexity, thereby addressing the sensitivity of neural radiance fields to sparse initial sampling. It sets itself apart from uniform sampling by using a coarse network to estimate density and guide importance sampling for a fine network, and from [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md) by explicitly defining the coarse-to-fine pipeline as the primary mechanism for efficiency, often implemented via the inverse-CDF of coarse weights.

### Resampling Techniques
Resampling techniques align geometry and color between scenes along corresponding rays [S2]. Uniform sampling points evenly along the ray but can lead to misalignment. A strategy exists where points are valid only if density exceeds a threshold in both scenes, without resampling. Density-based resampling identifies density peaks along each ray and adjusts sampling to align key points in both scenes, ensuring improved color-geometry consistency [S2].

### Consistent Ray Sampling
Consistent Ray Sampling ensures that ray sample intervals are consistent across all subvolumes to allow correct reconstruction of the original volume and prevent feature loss or enhancement [S3]. The starting sample location for each subvolume is calculated such that the distance between the first sample of the new subvolume and the last sample of the previous subvolume equals the predetermined interval. Incorrect calculation of sample locations causes small features in the data to be lost or enhanced erroneously [S3].

### Ray Direction Jitter
Ray Direction Jitter improves generalizability by applying sub-pixel jitter to ray directions [S4]. Instead of shooting in the pixel centers, a jitter $\psi$ is applied such that:
$$ \mathbf{x}_i = \mathbf{o} + t_i (\mathbf{d} + \psi) $$
where $\psi$ is sampled uniformly such that the resulting ray still confines to the boundaries of its designated pixel [S4].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray ($NumSamples$) and the number of pixels.
*   **Memory Complexity:** Depends on the volume texture size and the jitter texture size ($JitterTexSize^2$).
*   **Strengths:** Reduces banding artifacts significantly compared to uniform sampling, especially at low sample counts. Simple to implement in shaders.
*   **Limitations:** Early ray termination may miss faint structures. Gradient estimation can be noisy or produce NaNs.
*   **When to use:** When visual quality (reduction of banding) is prioritized over strict physical accuracy or when using low sample counts.
*   **When not to use:** When high precision is required for faint structures behind dense ones, or when deterministic sampling is strictly required for reproducibility without random seeds.

## Background
The method originates from direct volume ray casting techniques that utilize jittered sampling to mitigate aliasing. The specific implementation details, including the jitter texture lookup and Phong shading integration, are documented in the VoxViz thesis [S1, unofficial]. Consistent sampling across subvolumes for parallel rendering was discussed in early parallel ray tracing algorithms [S3]. Ray direction jitter for generalizability in neural rendering contexts was introduced in NeRF for Outdoor Scene Relighting [S4].

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S2] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
[S3] A data distributed, parallel algorithm for ray-traced volume ..., https://www.cs.ucdavis.edu/~ma/papers/PRS93.pdf, fetched 2026-10-27
[S4] NeRF for Outdoor Scene Relighting, https://doi.org/10.1007/978-3-031-19787-1_35, fetched 2026-10-26
