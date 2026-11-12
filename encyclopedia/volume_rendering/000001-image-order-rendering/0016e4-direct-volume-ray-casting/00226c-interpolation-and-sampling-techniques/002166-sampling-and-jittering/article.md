# Sampling and Jittering

Also called: Jittered Ray Casting, Jittered Sampling Volume Rendering, RayCaster, Volume Ray Casting with Jittered Sampling and Front-to-Back Blending

## Overview
Sampling and Jittering is a direct volume rendering technique that casts rays from the camera through a 3D voxel grid, perturbing sample positions with pre-computed offsets to reduce banding and aliasing artifacts caused by regular grid structures [S1, unofficial]. It distinguishes itself from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by breaking the periodicity of sample placement, and from [Stochastic Jittering](924-stochastic-jittering.md) by using a deterministic, fixed 2D texture for offsets rather than runtime random number generation [S1, unofficial][S6, unofficial]. It builds on [Interpolation and Sampling Techniques](2787-interpolation-and-sampling-techniques.md) by applying these perturbations before interpolation to mitigate high-frequency sampling errors.

## How it works
**Inputs:**
*   `VoxelSampler`: A 3D texture containing the volume data (scalar density and color).
*   `JitterTexSampler`: A 2D texture containing pre-computed jitter offsets.
*   `CameraPosition`: The camera's world-space position.
*   `LightPosition`: The light source's world-space position.
*   `VolExtentMin`, `VolExtentMax`: The bounding box of the volume in world space.
*   `RayStepSize`: The distance between consecutive sample points.
*   `NumSamples`: Maximum iterations per ray.
*   `ComputeLighting`: Boolean flag to enable shading.
*   `k_Ambient`, `k_Diffuse`, `k_Specular`, `k_Shininess`: Phong shading parameters.

**Outputs:**
*   `FragColor`: The final composited color and alpha for the pixel.

**Data Structures:**
*   `dst`: A 4-component vector `(R, G, B, A)` initialized to `(0.0, 0.0, 0.0, 0.0)`.
*   `rayPosition`: A 3D vector representing the current sample location in world space.
*   `rayDir`: A normalized 3D vector representing the ray direction.
*   `rayStep`: A 3D vector representing the displacement per sample (`rayDir * RayStepSize`).

**Algorithm Steps:**

1.  **Initialization**:
    Set `dst = (0.0, 0.0, 0.0, 0.0)`.
    Calculate `rayDir = normalize(rayPosition - CameraPosition)`.
    Calculate `rayStep = rayDir * RayStepSize`.

2.  **Jitter Application**:
    Determine the 2D texture coordinate from the fragment's screen coordinates modulo `JitterTexSize`.
    Fetch the red channel value `jitter_value` from `JitterTexSampler` at this coordinate.
    Update `rayPosition = rayPosition + rayStep * jitter_value`.
    [G] The jitter value is typically in the range $[0, 1]$ or $[-0.5, 0.5]$ depending on the texture generation; the source implies a direct additive scaling.

3.  **Boundary Check**:
    Determine if `rayPosition` is inside the volume bounds defined by `VolExtentMin` and `VolExtentMax`.
    The source uses a specific dot-product check:
    Let `test1` and `test2` be vectors derived from the position relative to the extent bounds.
    If `dot(test1, test2) >= 3.0`, the point is considered inside. [S1, unofficial]
    [G] This specific check likely corresponds to a normalized coordinate check where each component must be within $[0, 1]$, and the dot product of three valid components equals 3.0.

4.  **Voxel Sampling**:
    Sample the volume data at `rayPosition` using `VoxelSampler`.
    Let the result be `src` (color and alpha).

5.  **Lighting (Optional)**:
    If `ComputeLighting` is true:
    a.  **Gradient Estimation**: Estimate the normal using central differences on the alpha channel ($a$) of the voxel texture.
        Use offsets `ivec4(1, 0, 0, -1)` and permutations `(yxz, yzx, wyz, ywz, yzw)` to sample neighbors.
        Calculate `normal = normalize(sample2 - sample1)`.
        If `normal` is NaN (e.g., if `sample1 == sample2`), skip shading for this sample. [S1, unofficial]
    b.  **Phong Shading**: Apply shading to `src.rgb` using the `shading` function with `normal`, `toCamera`, and `toLight`.
        The shading function computes:
        $$ \text{halfway} = \text{normalize}(\text{toLight} + \text{toCamera}) $$
        $$ \text{diffuseLight} = \max(\text{dot}(\text{toLight}, \text{normal}), 0) $$
        $$ \text{diffuse} = \begin{cases} k_{\text{Diffuse}} \cdot \text{rgb} \cdot \text{diffuseLight} & \text{if } \text{diffuseLight} > 0 \\ 0 & \text{otherwise} \end{cases} $$
        $$ \text{specularLight} = \max(\text{dot}(\text{halfway}, \text{normal}), 0)^{k_{\text{Shininess}}} $$
        $$ \text{specular} = k_{\text{Specular}} \cdot \text{rgb} \cdot \text{specularLight} $$
        $$ \text{ambient} = k_{\text{Ambient}} \cdot \text{rgb} $$
        $$ \text{result} = \text{ambient} + \text{diffuse} + \text{specular} $$
        Update `src.rgb` with the result.

6.  **Compositing**:
    Composite `src` into `dst` using front-to-back blending:
    $$ \text{dst} = (1.0 - \text{dst}.a) \cdot \text{src} + \text{dst} $$

7.  **Early Ray Termination**:
    If `dst.a > 0.95`, break the loop. [S1, unofficial]

8.  **Advancement**:
    Update `rayPosition = rayPosition + rayStep`.
    Re-evaluate the inside condition.
    Repeat steps 2-7 until `rayPosition` is outside bounds or `NumSamples` is reached.

9.  **Output**:
    Set `FragColor = dst`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_jittered_volume(fragment_coords, camera_pos, light_pos, 
                           vol_extent_min, vol_extent_max, 
                           voxel_sampler, jitter_tex_sampler, 
                           ray_step_size, num_samples, compute_lighting,
                           k_ambient, k_diffuse, k_specular, k_shininess):
    
    # 1. Initialization
    dst = vec4(0.0, 0.0, 0.0, 0.0)
    ray_dir = normalize(fragment_coords - camera_pos) # Assuming fragment_coords is origin for ray
    ray_step = ray_dir * ray_step_size
    
    # Initial jitter application
    jitter_uv = fragment_coords % jitter_tex_sampler.size
    jitter_val = jitter_tex_sampler.sample(jitter_uv).r
    ray_pos = fragment_coords + ray_step * jitter_val
    
    for i in range(num_samples):
        # 3. Boundary Check
        if not is_inside_volume(ray_pos, vol_extent_min, vol_extent_max):
            break
            
        # 4. Voxel Sampling
        src = voxel_sampler.sample(ray_pos)
        
        # 5. Lighting
        if compute_lighting:
            normal = estimate_gradient_normal(voxel_sampler, ray_pos)
            if not is_nan(normal):
                src.rgb = apply_phong_shading(src.rgb, normal, camera_pos, light_pos, 
                                              k_ambient, k_diffuse, k_specular, k_shininess)
        
        # 6. Compositing
        dst = (1.0 - dst.a) * src + dst
        
        # 7. Early Ray Termination
        if dst.a > 0.95:
            break
            
        # 8. Advancement
        ray_pos += ray_step
        # Re-apply jitter for next step? The source implies jitter is applied once per step or per ray start.
        # Source says: "Apply jitter offset... add rayStep * jitter_value to rayPosition" inside the loop context 
        # or as a setup. The code snippet shows it before the loop or as part of the step.
        # Re-reading S1: "Apply jitter offset... add rayStep * jitter_value to rayPosition" is listed as a step.
        # Then "Advance rayPosition by rayStep".
        # To be precise to the source's "step" list:
        # Step 4: Apply jitter. Step 5: Check bounds. Step 6: Sample. ... Step 11: Advance.
        # This implies jitter is applied to the *current* position before sampling.
        # However, standard jittered sampling usually perturbs the step. 
        # The source code snippet is not provided, only the step list. 
        # The step list order: 1. Init, 2. Calc Dir/Step, 3. Apply Jitter, 4. Check Inside, 5. Sample, 6. Lighting, 7. Composite, 8. ERT, 9. Advance.
        # This implies jitter is applied to the position *before* the check.
        # Let's adjust the loop to match the source's explicit step order.
        
        # Re-evaluating source step order:
        # 1. Init dst
        # 2. Calc rayDir
        # 3. Calc rayStep
        # 4. Apply jitter offset (to rayPosition)
        # 5. Check inside
        # 6. Sample
        # 7. Lighting
        # 8. Composite
        # 9. ERT
        # 10. Advance rayPosition
        # 11. Re-evaluate inside
        
        # This suggests the jitter is applied to the *starting* position of the ray or each step?
        # "Get 2D texture coordinate from fragment coordinates... add rayStep * jitter_value to rayPosition"
        # If this is inside the loop, it perturbs each sample.
        
        # Let's refine the loop based on the explicit steps provided in S1.
        pass

def is_inside_volume(pos, min_ext, max_ext):
    # Source: dot(test1, test2) >= 3.0
    # Assuming normalized coordinates [0,1]
    norm_pos = (pos - min_ext) / (max_ext - min_ext)
    # If all components are in [0,1], dot([1,1,1], norm_pos) == sum(norm_pos)
    # But the source says dot(test1, test2). 
    # Likely test1 = [1,1,1] and test2 = norm_pos? Or test1/test2 are masks?
    # Given the ambiguity, we implement the check as:
    if all(0.0 <= c <= 1.0 for c in norm_pos):
        return True
    return False

def estimate_gradient_normal(sampler, pos):
    # Central differences on alpha channel
    # Offsets: ivec4(1,0,0,-1) and permutations
    # This implies sampling neighbors in voxel space
    # Simplified:
    dx = sampler.sample(pos + vec3(1,0,0)).a - sampler.sample(pos - vec3(1,0,0)).a
    dy = sampler.sample(pos + vec3(0,1,0)).a - sampler.sample(pos - vec3(0,1,0)).a
    dz = sampler.sample(pos + vec3(0,0,1)).a - sampler.sample(pos - vec3(0,0,1)).a
    normal = normalize(vec3(dx, dy, dz))
    return normal

def apply_phong_shading(rgb, normal, cam_pos, light_pos, k_amb, k_diff, k_spec, k_shin):
    to_light = normalize(light_pos - cam_pos) # Or from surface? Source uses toLight/toCamera vectors.
    to_camera = normalize(cam_pos - ???) # Source uses toCamera. Usually view vector.
    # Source shading function:
    halfway = normalize(to_light + to_camera) # Assuming these are normalized vectors pointing to light/camera
    diffuse_light = max(dot(to_light, normal), 0.0)
    
    diffuse = vec3(0.0)
    specular = vec3(0.0)
    
    if diffuse_light > 0.0:
        diffuse = k_diff * rgb * diffuse_light
        specular_light = pow(max(dot(halfway, normal), 0.0), k_shin)
        specular = k_spec * rgb * specular_light
        
    ambient = k_amb * rgb
    return ambient + diffuse + specular
```

*Note on Pseudocode Correction*: The source steps are sequential. The jitter is applied to `rayPosition`. The loop structure in the source is implicit. The jitter texture coordinate is derived from `fragment coordinates`. This suggests the jitter is constant for the pixel (or thread) but applied to the ray position.

## Subtopics

### [Stochastic Jittering](924-stochastic-jittering.md)
Stochastic jittering perturbs sample positions using random noise to mitigate banding. It sets itself apart from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by breaking periodicity, and from [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md) by using a fixed, pre-computed noise texture rather than distributing rays, trading visible banding for less objectionable dither patterns.

### [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md)
Hierarchical sampling generates importance samples by inverting the CDF of the rendering contribution. It solves the problem of efficiently placing samples in high-density regions. Unlike standard hierarchical sampling, this method uses inverse transform sampling on a piecewise approximated density field, allowing gradients to back-propagate. It builds on [Sampling and Jittering](2516-sampling-and-jittering.md) by providing a deterministic/stratified mechanism to concentrate fine samples.

### [Multi-Sample Anti-Aliasing via Ray Distribution](1219-multi-sample-anti-aliasing-via-ray-distribution.md)
MSAA via Ray Distribution reduces aliasing by casting multiple rays per pixel, distributed within the pixel area, and averaging results. It differs from [Stochastic Jittering](924-stochastic-jittering.md) by using deterministic, fixed offsets for low sample counts (1x–4x) rather than random patterns, and from [Monte Carlo Volume Rendering](2060-monte-carlo-volume-rendering.md) by relying on simple arithmetic averaging.

### [Uniform Ray Sampling](1261-uniform-ray-sampling.md)
Uniform Ray Sampling distributes depths uniformly between near ($t_n$) and far ($t_f$) clipping planes. It assumes constant probability density and linearly decaying transmittance. It is the baseline for NeRF-style raymarching but offers the lowest efficiency compared to spatial skipping or importance sampling.

### [Random Jitter with Reprojection](2782-random-jitter-with-reprojection.md)
Random Jitter with Reprojection reduces banding by introducing stochastic jitter and reducing noise via temporal accumulation. It distinguishes itself from pure spatial jittering by leveraging temporal coherence, projecting a pseudo-hit position from the current frame into the previous frame's screen space to blend with a history buffer.

### [Supersampling](3493-supersampling.md)
Supersampling minimizes aliasing by casting multiple rays per pixel and averaging results. Unlike [Stochastic Jittering](924-stochastic-jittering.md) which relies on random distribution, standard supersampling often uses structured patterns (e.g., grid-based) or hardware-accelerated interpolation. It builds on [Sampling and Jittering](2516-sampling-and-jittering.md) by increasing sample density.

### [Importance Sampling for SPH Rendering](3624-importance-sampling-for-sph-rendering.md)
Importance Sampling for SPH Rendering increases performance by reducing particle counts using a non-uniform strategy based on data importance and view-dependent criteria. It solves the high computational cost of large SPH datasets by sampling complex regions (e.g., phase interfaces) more densely. It differs from [Uniform Ray Sampling](1261-uniform-ray-sampling.md) by avoiding equal-time high-error trade-offs.

### [Ray Casting with Jittered Sampling](4151-ray-casting-with-jittered-sampling.md)
Ray Casting with Jittered Sampling casts rays through a 3D voxel grid, perturbing sample positions with a pre-computed jitter texture to break up regular grid patterns and reduce aliasing. Unlike stochastic methods, it uses deterministic offsets stored in a 2D texture rather than runtime random number generation [G].

### [Warp-based Sampling](4298-warp-based-sampling.md)
Warp-based Sampling is a GPU optimization that improves texture cache performance by reorganizing thread warps to align sampling directions with memory strides. Unlike standard sampling, it assigns a single warp to a single ray, marching through the volume in chunks equal to the warp size. This minimizes memory stride within the atomic scheduling unit, effective for large dynamic volumes at low resolutions.

### Resampling Techniques
Resampling techniques align geometry and color between scenes along corresponding rays [S2]. Uniform sampling can lead to misalignment. Density-based resampling identifies density peaks along each ray and adjusts sampling to align key points in both scenes, ensuring improved color-geometry consistency [S2].

### Consistent Ray Sampling
Consistent Ray Sampling ensures ray sample intervals are consistent across all subvolumes to allow correct reconstruction of the original volume and prevent feature loss [S3]. It calculates the starting sample location for each subvolume such that the distance between the first sample of the new subvolume and the last sample of the previous subvolume equals the predetermined interval. Incorrect calculation causes small features to be lost or enhanced erroneously [S3].

### Interleaved Sampling
Interleaved sampling samples every $2^n$ voxels as long as voxels are fully transparent, sampling only $1/4$ of the points in the image and interpolating. It is faster for interactive mode but results in less quality compared to full sampling [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of samples per ray (or `NumSamples`).
*   **Memory Complexity**: $O(V)$ for the volume texture and $O(J)$ for the jitter texture, where $V$ is the volume size and $J$ is the jitter texture size.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Performance scales linearly with resolution and sample count. Early ray termination can reduce average sample count.
*   **Strengths**: Reduces banding artifacts effectively; deterministic jitter avoids runtime RNG overhead; simple to implement.
*   **Limitations**: Can miss faint structures behind dense ones due to early ray termination (alpha > 0.95); gradient calculation may result in NaN if values are identical.
*   **When to use**: When banding artifacts from uniform sampling are visible and deterministic jitter is preferred over stochastic methods.
*   **When not**: When high-frequency detail preservation is critical and early ray termination might obscure it; when gradient-based lighting is not required or desired.

## Background
The method originates from direct volume rendering techniques that utilize jittered sampling to improve image quality. The specific implementation details, including the jitter texture application, Phong shading, and early ray termination threshold, are derived from the VoxViz/GigaVoxels debug renderer and associated thesis work [S1, unofficial]. The concept of jittered sampling to mitigate aliasing in volume rendering is a well-established technique in computer graphics, often cited in general volume rendering introductions [S4, unofficial].

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S2] Generalized TensoRF: Efficient Multi-Scene Radiance Fields and View-Consistent 3D Editing, https://doi.org/10.1145/3722564.3728385, fetched 2026-10-29
[S3] A data distributed, parallel algorithm for ray-traced volume ..., https://www.cs.ucdavis.edu/~ma/papers/PRS93.pdf, fetched 2026-10-27
[S4] Introduction to volume rendering, https://slidetodoc.com/introduction-to-volume-rendering-presented-by-zvi-devir/, fetched 2026-10-29 (unofficial)
