# Ambient Occlusion via Shadow Rays

Also called: Ray-traced AO, Shadow Ray Ambient Occlusion

## Overview
Ambient Occlusion via Shadow Rays approximates the lack of secondary (reflected) light rays by evaluating the geometric obscurance of a point $\mathbf{p}$ within or on the boundary of a volume [S1, unofficial]. It solves the problem of flat, unshaded volume appearances by darkening crevices and areas shielded from the environment, providing a view-independent shading term that can be combined with arbitrary direct illumination [S1, unofficial]. Unlike [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md) which uses probabilistic depth buffers, or [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md) which relies on precomputed spatial indices, this method directly samples the hemisphere around $\mathbf{p}$ using shadow rays to determine visibility, offering a balance between the physical accuracy of [Recursive Ray Tracing](5600-recursive-ray-tracing.md) and the speed of screen-space approximations [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Surface point position $\mathbf{p}$, surface normal $\mathbf{n}$, volume density field $V(\mathbf{x})$ (or extinction coefficient $\sigma_t$), number of shadow rays $N_R$, number of samples per ray $N_S$, and optional parameters for falloff or density thresholds.
*   **Outputs:** Ambient occlusion factor $A \in [0, 1]$ or radiance contribution $L_{AO}$.

**Data Structures:**
*   **Volume Texture/Field:** A 3D texture or sparse voxel structure storing density values. Access is performed via trilinear interpolation or ray marching [S4, unofficial].
*   **Sample Directions:** Precomputed or generated on-the-fly directions $\omega_i$ distributed over the hemisphere oriented by $\mathbf{n}$ [S2, unofficial].

**Formulas:**
The fundamental definition of ambient occlusion $A(p)$ substitutes the distance-based obscurance function with a binary visibility function $V(p, \omega)$ [S1, unofficial]:

$$ A(p) = \frac{1}{\pi} \int_{\Omega} V(p, \omega) \cos \theta \, d\omega $$

where $\Omega$ is the hemisphere oriented according to the surface normal, $\theta$ is the angle between the normal $\mathbf{n}$ and direction $\omega$, and $V(p, \omega)$ is 1 if the path is clear and 0 if occluded [S1, unofficial].

In implementations using density-weighted attenuation (obscurance), the formula becomes [S4, unofficial]:

$$ AO = \text{clamp}\left(2 \cdot \exp\left(\frac{-7.0}{N_R N_S} \sum_{i=0}^{N_R-1} \sum_{j=0}^{N_S-1} I_{ij} \cdot \mathbb{I}(I_{ij} \in [T_{min}, T_{max}])\right), 0, 1\right) $$

where $I_{ij}$ is the density sampled at the $j$-th point along the $i$-th ray, and $\mathbb{I}$ is the indicator function for density thresholds [S4, unofficial].

**Steps:**
1.  **Normal Estimation:** If not provided, compute the normal $\mathbf{n}$ from the volume gradient $\mathbf{g} = \nabla V$ at $\mathbf{p}$ using central differences [S4, unofficial].
2.  **Tangent Basis Construction:** Construct a local tangent space basis $(\mathbf{t}_0, \mathbf{t}_1)$ from $\mathbf{n}$ to transform hemisphere samples to world space [S6, unofficial].
    *   If $|n_z| > 0.5$, set $\mathbf{t}_0 = (-n_y, n_x, 0)$, else $\mathbf{t}_0 = (1, 0, 0)$.
    *   Normalize $\mathbf{t}_0$.
    *   Compute $\mathbf{t}_1 = \mathbf{n} \times \mathbf{t}_0$.
3.  **Ray Casting Loop:** For each of $N_R$ rays:
    *   Sample a direction $\omega$ from the hemisphere using cosine-weighted or uniform solid angle sampling [S2, unofficial].
    *   Transform $\omega$ to world space: $\mathbf{d} = \omega_x \mathbf{t}_0 + \omega_y \mathbf{t}_1 + \omega_z \mathbf{n}$.
    *   Cast a shadow ray from $\mathbf{p} + \epsilon \mathbf{d}$ in direction $\mathbf{d}$ [S2, unofficial].
    *   Evaluate visibility $V$ by checking for intersections with volume boundaries or density thresholds [S6, unofficial].
4.  **Accumulation:**
    *   **Binary Visibility:** Accumulate $(1 - V) \cdot \cos \theta$ [S2, unofficial].
    *   **Density Weighted:** Accumulate density-weighted attenuation terms as per the obscurance formula [S4, unofficial].
5.  **Normalization:** Average the accumulated value over $N_R$ samples [S2, unofficial].
6.  **Final Color:** Combine the AO factor with ambient lighting: $L_{ambient} = L_{sky} \cdot A \cdot \text{color}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ambient_occlusion(pos, normal, volume_texture, num_rays=10, num_samples_per_ray=5):
    """
    Computes ambient occlusion using shadow rays and density sampling.
    Based on [S4, unofficial] and [S6, unofficial].
    """
    # 1. Tangent Basis Construction [S6, unofficial]
    # Choose t0 based on normal to avoid degeneracy
    if abs(normal.z) > 0.5:
        t0 = vec3(-normal.y, normal.x, 0.0)
    else:
        t0 = vec3(1.0, 0.0, 0.0)
    
    t0 = normalize(t0)
    t1 = cross(normal, t0)
    
    occlusion_sum = 0.0
    
    # 2. Ray Casting Loop [S4, unofficial][S6, unofficial]
    for i in range(num_rays):
        # Sample direction from hemisphere [S6, unofficial]
        # Using a simple uniform sphere sample for demonstration, 
        # cosine-weighted is preferred for physical accuracy [S2, unofficial]
        dir_local = sample_uniform_sphere(i) 
        
        # Transform to world space [S6, unofficial]
        dir_world = t0 * dir_local.x + t1 * dir_local.y + normal * dir_local.z
        
        # Offset to avoid self-intersection [S2, unofficial]
        offset = dir_world * 0.01
        sample_pos = pos + offset
        
        # 3. Sample along ray [S4, unofficial]
        for j in range(num_samples_per_ray):
            sample_pos += offset
            
            # Boundary check [S4, unofficial]
            if any(less_than(sample_pos, min_slices)) or any(greater_than(sample_pos, max_slices)):
                break
            
            # Sample density [S4, unofficial]
            neighbor_intensity = texture(volume_texture, sample_pos).r
            
            # Check density threshold [S4, unofficial]
            if min_density_threshold < neighbor_intensity < max_density_threshold:
                # Accumulate attenuation [S4, unofficial]
                occlusion_sum += -7.0 * neighbor_intensity / (num_rays * num_samples_per_ray)
                
    # 4. Final Calculation [S4, unofficial]
    ao_factor = clamp(2.0 * exp(occlusion_sum), 0.0, 1.0)
    return ao_factor

def compute_ao_radiance(pos, normal, ambient_color, constant_color, ambient_exponent=1.3):
    """
    Computes final AO radiance contribution.
    Based on [S6, unofficial].
    """
    output_color = vec3(0.0)
    
    # Tangent Basis [S6, unofficial]
    if abs(normal.z) > 0.5:
        t0 = vec3(-normal.y, normal.x, 0.0)
    else:
        t0 = vec3(1.0, 0.0, 0.0)
    t0 = normalize(t0)
    t1 = cross(normal, t0)
    
    # Sample Hemisphere [S6, unofficial]
    # Loop 2 times for low sample count [S3, unofficial]
    for _ in range(2):
        dir_local = cosine_sample_hemisphere(blue_noise_2d())
        dir_world = t0 * dir_local.x + t1 * dir_local.y + normal * dir_local.z
        
        # Incoming light approximation [S6, unofficial]
        incoming = dir_world.y * 0.2 + 0.8
        
        # Raycast for distance [S6, unofficial]
        hit_dist = raycast_ambient(pos, normal, dir_world, max_distance=24.0)
        
        # Falloff [S6, unofficial]
        t = hit_dist / 24.0
        t = pow(t, ambient_exponent)
        
        # Accumulate [S6, unofficial]
        output_color.rgb += ambient_color * (incoming * t)
        output_color.rgb += constant_color * (incoming * max(1.0, hit_dist * 0.25))
        
    return output_color / 2.0
```

**Parameters:**
*   $N_R$ (Number of rays): Typically 10–64 per pixel/surface [S4, unofficial][S5].
*   $N_S$ (Samples per ray): Typically 5 [S4, unofficial].
*   $T_{min}, T_{max}$ (Density thresholds): Define the range of density considered for occlusion [S4, unofficial].
*   `ambientExponent`: Exponent for the falloff term, typically 1.3 [S6, unofficial].
*   `ambientColor`: Color of the ambient light, typically `vec3(1.0)` [S6, unofficial].
*   `constantColor`: Constant ambient color to prevent total darkness, typically `vec3(0.003)` [S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Noise:** Low sample counts (1–2 samples per pixel) lead to very noisy images [S3, unofficial]. Denoising algorithms (spatial/temporal convolutional) are often required [S3, unofficial].
*   **Precision:** Normalizing very small volume gradients can lead to artifacts; a magnitude threshold (e.g., 0.0001) should be applied [S4, unofficial].
*   **Boundaries:** Ray samples falling outside the volume bounds must be handled to avoid out-of-bounds memory access [S4, unofficial].

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
A storage and reconstruction method where the reciprocal of a quadratic polynomial encodes how ambient occlusion changes with distance. Coefficients are stored in a cube map [S3, unofficial].
*   **Layout:** Data structure is a cube map storing coefficients of the reciprocal of a quadratic polynomial [S3, unofficial].
*   **Pitfall:** Requires reconstructing the ambient occlusion factor at runtime, which incurs a performance penalty [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_R \cdot N_S \cdot T_{ray})$, where $N_R$ is the number of rays, $N_S$ is samples per ray, and $T_{ray}$ is the cost of ray traversal/sampling in the volume. With $N_R=64$ and $N_S=5$, this is computationally expensive for real-time without acceleration [S4, unofficial][S5].
*   **Memory Complexity:** $O(1)$ auxiliary memory per pixel, plus the storage for the volume texture [S3, unofficial].
*   **Measured Performance:** On high-resolution sparse voxel DAGs, shooting 64 shadow rays per pixel provides high quality but requires denoising for low sample counts [S5]. Hardware-accelerated ray tracing (DXR, Vulkan RT) allows implementation of the original formulation, but 1–2 samples per pixel are often used in practice due to cost, leading to noise [S3, unofficial].
*   **Strengths:** View-independent; combines well with arbitrary direct illumination; physically plausible when using cosine-weighted sampling [S1, unofficial].
*   **Limitations:** Noisy with low samples; scales with scene complexity (not constant like screen-space methods); requires efficient volume traversal [S3, unofficial].
*   **When to Use:** When view-independent shading is required and hardware ray tracing is available, or for offline/precomputed shading.
*   **When Not to Use:** When real-time performance is critical and denoising is not an option, or for dynamic scenes where cache invalidation is frequent.

## Background
Ambient occlusion was introduced by Zhukov et al. [Zhukov 1998] and Iones [2003] as an efficient technique for perceptually similar results to global illumination with lower cost [S1, unofficial]. The term "obscurance" was used to describe the measure of the hemisphere obscured by neighboring surfaces [S1, unofficial]. Landis [2002] detailed its use for adding realism to models, and Méndez et al. [2003b] extended it to ray tracing [S1, unofficial]. Kontkanen and Laine [29] proposed storing occlusion data in an "ambient occlusion field" using a cube map of quadratic polynomial coefficients [S3, unofficial]. With the advent of hardware-accelerated ray tracing (DXR, Vulkan RT), the original formulation could be implemented directly as Ray-traced Ambient Occlusion (RTAO) [S3, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S5] (PDF) High resolution sparse voxel DAGs - ResearchGate, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S6] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
