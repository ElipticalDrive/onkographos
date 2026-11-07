# Ambient Occlusion via Shadow Rays

Also called: Ray-traced AO, Shadow Ray Ambient Occlusion

## Overview
Ambient Occlusion via Shadow Rays approximates the geometric obscurance of a surface point by casting shadow rays into the hemisphere above the surface normal to determine visibility [S2, unofficial][S3, unofficial]. It solves the problem of missing secondary (reflected) light in local illumination models by providing a view-independent occlusion factor that can be combined with arbitrary direct illumination [S1, unofficial]. It differs from its siblings [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md) and [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md) by relying on explicit ray casting against geometry or volumes rather than precomputed angular distributions or local spherical neighborhood integration [S2, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Surface position $\mathbf{p}$, surface normal $\mathbf{n}$, scene geometry or volume data structure, number of shadow rays $N_R$, number of samples per ray $N_S$, and optional falloff parameters.
*   **Outputs:** Ambient occlusion factor $A \in [0, 1]$, where 1 indicates no occlusion (completely open) and 0 indicates full occlusion [S1, unofficial].

**Data Structures**
*   **Scene Representation:** The method relies on a scene representation capable of ray intersection tests. This can be a triangle mesh, a Signed Distance Field (SDF), or a sparse voxel structure (e.g., SVO, DAG) [S5].
*   **Sampling Buffers:** No persistent buffers are required for the core algorithm, though implementations may use spatial caches for optimization [S1, unofficial].

**Formulas**
The fundamental definition of ambient occlusion $A(p)$ substitutes the distance-based function $\rho$ of obscurance with a binary visibility function $V(p, \omega)$ [S1, unofficial]:

$$ A(p) = \frac{1}{\pi} \int_{\Omega} V(p, \omega) \cos \theta \, d\omega $$

where:
*   $\Omega$ is the hemisphere oriented according to the surface normal at $p$.
*   $V(p, \omega)$ is 1 if the direction $\omega$ is visible (no geometry intersection) and 0 otherwise.
*   $\theta$ is the angle between the normal $\mathbf{n}$ and direction $\omega$.

Alternatively, the more general obscurance formulation uses a distance falloff function $\rho(d(p, \omega))$ [S1, unofficial]:

$$ O(p) = \frac{1}{\pi} \int_{\Omega} \rho(d(p, \omega)) \cos \theta \, d\omega $$

where $d(p, \omega)$ is the distance to the first intersection of a ray from $p$ in direction $\omega$ [S1, unofficial].

**Parameters**
*   **Number of shadow rays ($N_R$):** Typically 64 rays per pixel are used for high-quality results [S5]. In practice, 1 to 2 samples per pixel/surface are often used due to performance constraints, requiring denoising [S3, unofficial].
*   **Samples per ray ($N_S$):** Often 5 samples per ray are used to estimate density or occlusion along the ray path [S4, unofficial].
*   **Falloff Exponent:** An exponent applied to the normalized hit distance to control the falloff of occlusion (e.g., 1.3) [S6, unofficial].
*   **Ambient Color:** The color of the ambient light (e.g., `vec3(1.0)`) [S6, unofficial].
*   **Constant Color:** A minimum ambient color to prevent total darkness (e.g., `vec3(0.003)`) [S6, unofficial].

**Steps**
1.  **Initialize Accumulator:** Set `occlusion` to 0.0.
2.  **Sample Directions:** For each ray $i$ from 0 to $N_R - 1$:
    a.  Sample a direction $\mathbf{d}_i$ from the hemisphere above the normal $\mathbf{n}$. Common strategies include uniform solid angle sampling or cosine-weighted sampling [S2, unofficial].
    b.  For cosine-weighted sampling, the contribution is weighted by $\cos \theta_i$ [S2, unofficial].
3.  **Cast Shadow Ray:** Trace a ray from $\mathbf{p}$ in direction $\mathbf{d}_i$.
    a.  Check for intersection with scene geometry or volume density [S2, unofficial].
    b.  If an intersection is found at distance $t$, the direction is occluded.
4.  **Accumulate Occlusion:**
    *   **Binary Visibility:** If using binary visibility, add $(1 - \text{inShadow}) \cdot \cos \theta_i \cdot \frac{2}{\pi}$ to the accumulator for uniform sampling, or $\frac{1 - \text{inShadow}}{\pi}$ for cosine-weighted sampling [S2, unofficial].
    *   **Density-Weighted (Volume):** For volume rendering, sample points along the ray. If the density at a sample point is within a threshold range $[T_{min}, T_{max}]$, add a weighted contribution. A common implementation adds $-7.0 \cdot \text{neighborIntensity} / (N_R \cdot N_S)$ to the occlusion accumulator [S4, unofficial].
5.  **Finalize:**
    *   For binary visibility, average the result over $N_R$ samples: $L_r = L_r / N_R$ [S2, unofficial].
    *   For density-weighted, apply the exponential falloff: $A = \text{clamp}(2 \cdot \exp(\text{occlusion}), 0.0, 1.0)$ [S4, unofficial].
    *   Alternatively, compute a falloff term $t = (\text{hitDist} / \text{maxDist})^{\text{ambientExponent}}$ and combine with ambient color [S6, unofficial].
6.  **Apply:** Multiply the object's base color by the accumulated occlusion factor to get the final pixel color [S2, unofficial].

**Pseudocode**
The following pseudocode implements the density-weighted approach for volume rendering as described in [S4, unofficial], and the general ray-traced approach as described in [S6, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ambient_occlusion(pos, normal, volume_texture, num_rays=10, samples_per_ray=5):
    """
    Computes ambient occlusion for a point in a volume.
    Based on [S4, unofficial].
    """
    occlusion = 0.0
    min_density_threshold = 0.0
    max_density_threshold = 1.0
    
    for i in range(num_rays):
        # Sample a direction uniformly on the sphere/hemisphere
        # Note: For hemisphere, restrict to upper half relative to normal
        direction = sample_uniform_sphere(i)
        
        # Offset to avoid self-intersection
        offset = direction * 0.01
        
        sample_pos = pos + offset
        
        for j in range(samples_per_ray):
            # Move along the ray
            sample_pos += offset
            
            # Check bounds
            if is_out_of_bounds(sample_pos):
                break
                
            # Sample density from volume
            neighbor_intensity = texture(volume_texture, sample_pos).r
            
            # Check if density is in threshold range
            if min_density_threshold < neighbor_intensity < max_density_threshold:
                # Accumulate occlusion
                # Formula from [S4, unofficial]: occlusion += -7.0 * neighborIntensity / (NUM_RAYS * NUM_SAMPLES_PER_RAY)
                occlusion += -7.0 * neighbor_intensity / (num_rays * samples_per_ray)
                
    # Finalize occlusion value
    # Formula from [S4, unofficial]: clamp(2 * exp(occlusion), 0.0, 1.0)
    ao = clamp(2.0 * exp(occlusion), 0.0, 1.0)
    
    return ao

def compute_ray_traced_ao(pos, normal, scene, num_samples=2, ambient_exponent=1.3):
    """
    Computes ray-traced ambient occlusion for a surface point.
    Based on [S6, unofficial].
    """
    output_color = vec3(0.0)
    ambient_color = vec3(1.0)
    constant_color = vec3(0.003)
    
    # Compute tangent and bitangent for hemisphere sampling
    # Choose t0 based on normal.z to avoid degeneracy
    if abs(normal.z) > 0.5:
        t0 = normalize(vec3(0.0, 1.0, 0.0) - normal * dot(vec3(0.0, 1.0, 0.0), normal))
    else:
        t0 = normalize(vec3(1.0, 0.0, 0.0) - normal * dot(vec3(1.0, 0.0, 0.0), normal))
        
    t1 = cross(normal, t0)
    
    for _ in range(num_samples):
        # Sample direction from hemisphere using cosine-weighted sampling
        # blue_noise() provides jittered random numbers
        local_dir = cosine_sample_hemisphere(blue_noise_2d())
        
        # Transform to world space
        dir_world = t0 * local_dir.x + t1 * local_dir.y + normal * local_dir.z
        
        # Compute incoming light factor (simplified)
        incoming = dir_world.y * 0.2 + 0.8
        
        # Cast shadow ray
        hit_dist = raycast_ambient(pos, normal, dir_world, max_distance=24.0)
        
        # Compute falloff
        t = hit_dist / 24.0
        t = pow(t, ambient_exponent)
        
        # Accumulate color
        output_color += ambient_color * (incoming * t)
        output_color += constant_color * (incoming * max(1.0, hit_dist * 0.25))
        
    # Average and return
    return output_color / num_samples
```

**Edge Cases and Pitfalls**
*   **Low Sample Counts:** Using 1-2 samples per pixel leads to very noisy images [S3, unofficial]. Denoising algorithms (spatial/temporal convolutional or deep learning) are commonly required [S3, unofficial].
*   **Precision:** When computing normals from volume gradients, small gradients can lead to artifacts; a magnitude check is required [S4, unofficial].
*   **Boundaries:** Ray samples must be checked against volume bounds to avoid out-of-bounds memory access [S4, unofficial].

## Subtopics

### [Evans Ambient Occlusion](1083-evans-ambient-occlusion.md)
Evans Ambient Occlusion is a technique for approximating ambient occlusion by sampling a Signed Distance Field (SDF) along the surface normal to estimate sky visibility. It differs from its siblings [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md) and [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md) by relying on a global distance field texture rather than local geometric primitives or hemisphere sampling, making it suitable for dynamic or deforming scenes where explicit geometry is unavailable or expensive to query.

### [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md)
Incremental Local Ambient Occlusion (LAO) is a shading model for direct volume rendering that enhances ambient lighting by computing incident light from a local spherical neighborhood around each voxel, rather than using a constant ambient factor or global shadows. It solves the problem of missing local shadowing and context in volume rendering while avoiding global shadows that obscure relevant internal structures. Unlike traditional "all-or-nothing" ambient occlusion methods that treat voxels as opaque blockers, LAO incorporates semitransparency by integrating opacity along rays, and it uses an incremental update scheme to allow interactive frame rates during computation.

### [Solid Angle Early Termination for Ambient Occlusion](1146-solid-angle-early-termination-for-ambient-occlusion.md)
Solid Angle Early Termination for Ambient Occlusion accelerates shadow ray traversal in sparse voxel structures by stopping traversal when a node subtends a solid angle below a specified threshold, effectively using a lower level-of-detail representation for distant points. It differs from [Evans Ambient Occlusion](1083-evans-ambient-occlusion.md) and [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md) by relying on geometric projection thresholds rather than hemisphere sampling or precomputed angular distributions, and from [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md) by using a static threshold rather than an adaptive accumulation scheme.

### [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md)
Zhukov Ambient Occlusion is an empirical illumination model that simulates indirect lighting by computing the geometric obscurance of a surface point, representing how open it is to the environment. Unlike radiosity, which measures secondary reflection intensity, it attenuates the constant ambient term of the Phong model based on local geometric occlusion, prioritizing computational speed and aesthetic appeal over physical accuracy. It builds on [Ambient Occlusion via Shadow Rays](878-ambient-occlusion-via-shadow-rays.md) by incorporating a cosine-weighted hemisphere integration directly into the sampling scheme.

### [Spatially Cached Raytraced Ambient Occlusion](4106-spatially-cached-raytraced-ambient-occlusion.md)
Spatially Cached Raytraced Ambient Occlusion (SC-RTAO) is an optimization for raytraced ambient occlusion that stores accumulated occlusion values in spatial cells within a hashmap. Since ambient occlusion is view-independent, these values can be accumulated into a cell without reprojection when the camera moves, reducing redundant ray tracing for static geometry across frames. It distinguishes itself from naive RTAO by caching results to avoid redundant ray tracing, and from sibling methods like Evans AO or Zhukov AO by relying on spatial hashing and atomic accumulation rather than different sampling or approximation strategies.

### Ambient Occlusion Field
Ambient Occlusion Field is a storage and reconstruction method where the reciprocal of a quadratic polynomial encodes how ambient occlusion changes with distance [S3, unofficial]. The coefficients are stored in a cube map [S3, unofficial]. This allows modeling of directional variation of occlusion and the ambient occlusion effect an object has on its surroundings [S3, unofficial]. A key pitfall is that it requires reconstructing the ambient occlusion factor at runtime, which incurs a performance penalty [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of shadow rays $N_R$ and the number of samples per ray $N_S$ [S5]. For volume rendering, it also depends on the cost of volume traversal and density sampling [S4, unofficial].
*   **Memory Complexity:** Low, as it does not require storing large precomputed tables for the basic method, though caching variants require additional memory [S1, unofficial].
*   **Measured Performance:** Shooting 64 shadow rays per pixel provides high quality but is expensive [S5]. In practice, 1-2 samples are used, requiring denoising [S3, unofficial].
*   **Strengths:** View-independent, works with arbitrary direct illumination, physically plausible for indirect lighting [S1, unofficial]. Hardware-accelerated ray tracing allows for exact visibility checks [S3, unofficial].
*   **Limitations:** Noisy with low sample counts [S3, unofficial]. Computational complexity scales with scene complexity for dynamic scenes [S3, unofficial].
*   **When to Use:** When high-quality indirect lighting is needed and performance allows for denoising, or when precomputing occlusion for static scenes.
*   **When Not to Use:** When real-time performance is critical and denoising is not an option, or when the scene is highly dynamic and caching is not feasible.

## Background
Ambient Occlusion was introduced by Zhukov et al. [Zhukov 1998] and Iones [2003] as an efficient technique for perceptually similar results to global illumination with lower cost [S1, unofficial]. The computation was originally done as a preprocess [S1, unofficial]. Landis [2002] detailed how ambient occlusion could be used to add realism to models, and Méndez et al. [2003b] extended the use of obscurances to ray tracing [S1, unofficial]. The method has been further developed with hardware-accelerated ray tracing APIs like DirectX Ray Tracing and Vulkan Ray Tracing, enabling real-time implementations [S3, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S5] (PDF) High resolution sparse voxel DAGs - ResearchGate, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S6] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
