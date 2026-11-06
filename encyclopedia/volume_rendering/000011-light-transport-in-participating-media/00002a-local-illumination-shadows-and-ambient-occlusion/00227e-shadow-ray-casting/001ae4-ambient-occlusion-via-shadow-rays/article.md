# Ambient Occlusion via Shadow Rays

Also called: Ray-traced AO, Shadow Ray Ambient Occlusion

## Overview
Ambient Occlusion via Shadow Rays approximates the lack of secondary (reflected) light rays by integrating a visibility function over the hemisphere oriented by the surface normal [S1, unofficial]. It differs from its siblings [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md) by ignoring direct light directionality and source intensity, focusing solely on geometric obscurance, and from [Variance Shadow Mapping for NeRFs](3561-variance-shadow-mapping-for-nerfs.md) by using stochastic ray casting rather than probabilistic depth buffers. It serves as a view-independent shading term that can be combined with arbitrary direct illumination [S1, unofficial].

## How it works
**Inputs:** Surface point $\mathbf{p}$, surface normal $\mathbf{n}$, volume texture or scene geometry for ray intersection, number of rays $N_R$, samples per ray $N_S$, density thresholds $[T_{min}, T_{max}]$, and ambient color parameters.
**Outputs:** Ambient occlusion factor $AO \in [0, 1]$ or accumulated radiance $L_r$.

**Data Structures:**
*   **Volume Texture:** 3D texture storing density values (e.g., `volumeTexture`).
*   **Bounding Box:** Defined by `minSlices` and `maxSlices` for boundary checks.

**Formulas:**
The theoretical ambient occlusion $A(p)$ is defined as [S1, unofficial]:
$$ A(p) = \frac{1}{\pi} \int_{\Omega} V(p, \omega) \cos \theta \, d\omega $$
where $V(p, \omega)$ is the visibility function (1 if visible, 0 if occluded), $\Omega$ is the hemisphere, and $\theta$ is the angle between $\mathbf{n}$ and direction $\omega$.

A practical implementation using density-weighted attenuation [S4, unofficial] computes:
$$ AO = \text{clamp}\left(2 \cdot \exp\left(\frac{-7.0}{N_R N_S} \sum_{i=0}^{N_R-1} \sum_{j=0}^{N_S-1} I_{ij} \cdot \mathbb{I}(I_{ij} \in [T_{min}, T_{max}])\right), 0, 1\right) $$
where $I_{ij}$ is the density sampled at the $j$-th point along the $i$-th ray.

**Steps:**
1.  **Initialize:** Set `occlusion = 0.0` [S4, unofficial].
2.  **Ray Generation:** Loop $i$ from $0$ to $N_R - 1$:
    a.  Sample a direction $\mathbf{d}_i$ uniformly from the sphere or hemisphere [S4, unofficial][S6, unofficial].
    b.  Compute an initial offset $\mathbf{o} = \mathbf{d}_i \cdot \epsilon$ (e.g., $\epsilon = 0.01$) to avoid self-intersection [S4, unofficial].
    c.  Set sample position $\mathbf{p}_{sample} = \mathbf{p} + \mathbf{o}$.
3.  **Sampling:** Loop $j$ from $0$ to $N_S - 1$:
    a.  Check bounds: If $\mathbf{p}_{sample}$ is outside `minSlices` or `maxSlices`, break loop [S4, unofficial].
    b.  Sample density $I_{ij} = \text{texture}(\text{volumeTexture}, \mathbf{p}_{sample})$ [S4, unofficial].
    c.  Check threshold: If $T_{min} < I_{ij} < T_{max}$, add to occlusion sum:
        $$ \text{occlusion} \leftarrow \text{occlusion} + \frac{-7.0 \cdot I_{ij}}{N_R \cdot N_S} $$
        [S4, unofficial].
    d.  Advance position: $\mathbf{p}_{sample} \leftarrow \mathbf{p}_{sample} + \mathbf{o}$ [S4, unofficial].
4.  **Finalize:** Return $\text{clamp}(2 \cdot \exp(\text{occlusion}), 0.0, 1.0)$ [S4, unofficial].

**Alternative Implementation (Screen-Space/Ray-Traced) [S6, unofficial]:**
1.  **Tangent Basis:** Compute tangent $\mathbf{t}_0$ and bitangent $\mathbf{t}_1$ from normal $\mathbf{n}$.
    *   If $|n_z| > 0.5$, $\mathbf{t}_0 = \text{normalize}(\text{cross}(\mathbf{n}, \mathbf{vec3}(0,0,1)))$.
    *   Else, $\mathbf{t}_0 = \text{normalize}(\text{cross}(\mathbf{n}, \mathbf{vec3}(1,0,0)))$.
    *   $\mathbf{t}_1 = \text{cross}(\mathbf{n}, \mathbf{t}_0)$.
2.  **Loop:** For $k$ iterations (e.g., 2):
    a.  Sample direction $\mathbf{dir}_{local}$ from hemisphere using cosine-weighted sampling with blue noise [S6, unofficial].
    b.  Transform $\mathbf{dir}_{local}$ to world space: $\mathbf{dir} = \mathbf{t}_0 \cdot \mathbf{dir}_{local}.x + \mathbf{t}_1 \cdot \mathbf{dir}_{local}.y + \mathbf{n} \cdot \mathbf{dir}_{local}.z$.
    c.  Compute incoming factor: $\text{incoming} = \mathbf{dir}.y \cdot 0.2 + 0.8$ [S6, unofficial].
    d.  **Raycast:** Call `raycastAmbient(pos, normal, dir, maxDist)` to get `hitDist` [S6, unofficial].
        *   Helper `raycastAmbient` checks screen-space occlusion first. If not blocked, jitter position and call `raycastShadowVolumeSuperSparse` [S6, unofficial].
    e.  Compute falloff: $t = (\text{hitDist} / \text{maxDist})^{\text{ambientExponent}}$ (default exponent 1.3) [S6, unofficial].
    f.  Accumulate color:
        $$ \text{outputColor}.rgb += \text{ambientColor} \cdot (\text{incoming} \cdot t) $$
        $$ \text{outputColor}.rgb += \text{constantColor} \cdot (\text{incoming} \cdot \max(1.0, \text{hitDist} \cdot 0.25)) $$
        [S6, unofficial].
3.  **Return:** $\text{outputColor} / 2.0$ [S6, unofficial].

**Parameters:**
*   $N_R$: Number of rays. Typical values: 10 [S4, unofficial], 64 [S5].
*   $N_S$: Samples per ray. Typical value: 5 [S4, unofficial].
*   $\epsilon$: Offset distance. Typical value: 0.01 [S4, unofficial].
*   `ambientExponent`: Falloff exponent. Typical value: 1.3 [S6, unofficial].
*   `ambientColor`: Ambient light color. Typical value: `vec3(1.0)` [S6, unofficial].
*   `constantColor`: Constant ambient to prevent darkness. Typical value: `vec3(0.003)` [S6, unofficial].
*   Density Thresholds: $[T_{min}, T_{max}]$ define the range of density contributing to occlusion [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Low Sample Counts:** Using 1-2 samples per pixel leads to very noisy images [S3, unofficial]. Denoising (spatial/temporal convolutional or deep learning) is required [S3, unofficial].
*   **Boundary Checks:** Rays hitting volume boundaries must be handled to avoid sampling invalid memory [S4, unofficial].
*   **Self-Intersection:** Rays must be offset from the surface to avoid detecting the surface itself as an occluder [S4, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ao_density_weighted(pos, volume_texture, min_slices, max_slices, num_rays=10, num_samples=5):
    """
    Computes AO using density-weighted attenuation as per [S4, unofficial].
    """
    occlusion = 0.0
    inv_total = -7.0 / (num_rays * num_samples)
    
    for i in range(num_rays):
        # Sample direction uniformly from sphere
        direction = sample_uniform_sphere(i)
        offset = direction * 0.01
        
        sample_pos = pos + offset
        
        for j in range(num_samples):
            # Check bounds
            if any_less_than(sample_pos, min_slices) or any_greater_than(sample_pos, max_slices):
                break
            
            # Sample density
            neighbor_intensity = volume_texture.sample(sample_pos).r
            
            # Check threshold
            if min_density_threshold < neighbor_intensity < max_density_threshold:
                occlusion += inv_total * neighbor_intensity
            
            # Advance
            sample_pos += offset
            
    return clamp(2.0 * exp(occlusion), 0.0, 1.0)

def sample_uniform_sphere(index):
    """
    Placeholder for uniform sphere sampling. 
    In practice, use a precomputed sequence or spherical coordinate sampling.
    """
    # Implementation detail: [G] Using a simple spherical coordinate mapping for demonstration
    phi = 2.0 * 3.14159265 * (index / 10.0) # Simple distribution for 10 rays
    cos_theta = 1.0 - 2.0 * ((index % 5) / 5.0)
    sin_theta = sqrt(1.0 - cos_theta * cos_theta)
    return vec3(sin_theta * cos(phi), sin_theta * sin(phi), cos_theta)

def any_less_than(v, limit):
    return v.x < limit.x or v.y < limit.y or v.z < limit.z

def any_greater_than(v, limit):
    return v.x > limit.x or v.y > limit.y or v.z > limit.z
```

## Subtopics

### [Evans Ambient Occlusion](1083-evans-ambient-occlusion.md)
Evans Ambient Occlusion is a technique for approximating ambient occlusion by sampling a Signed Distance Field (SDF) along the surface normal to estimate sky visibility. It differs from its siblings [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md) and [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md) by relying on a global distance field texture rather than local geometric primitives or hemisphere sampling, making it suitable for dynamic or deforming scenes where explicit geometry is unavailable or expensive to query.

### [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md)
Incremental Local Ambient Occlusion (LAO) is a shading model for direct volume rendering that enhances ambient lighting by computing incident light from a local spherical neighborhood around each voxel, rather than using a constant ambient factor or global shadows. It solves the problem of missing local shadowing and context in volume rendering while avoiding global shadows that obscure relevant internal structures. Unlike traditional "all-or-nothing" ambient occlusion methods that treat voxels as opaque blockers, LAO incorporates semitransparency by integrating opacity along rays, and it uses an incremental update scheme to allow interactive frame rates during computation.

### [Solid Angle Early Termination for Ambient Occlusion](1146-solid-angle-early-termination-for-ambient-occlusion.md)
Solid Angle Early Termination for Ambient Occlusion accelerates shadow ray traversal in sparse voxel structures by stopping traversal when a node subtends a solid angle below a specified threshold, effectively using a lower level-of-detail representation for distant points. It differs from [Evans Ambient Occlusion](1083-evans-ambient-occlusion.md) and [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md) by relying on geometric projection thresholds rather than hemisphere sampling or precomputed angular distributions, and from [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md) by using a static threshold rather than an adaptive accumulation scheme.

### [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md)
Zhukov Ambient Occlusion is an empirical ambient light illumination model that simulates indirect illumination by computing the geometric obscurance of a surface point, representing how open the point is to the environment. It approximates darkening effects in obscured areas by attenuating the constant ambient term of the Phong lighting model, rather than calculating the intensity of secondary reflections as in radiosity. This approach is computationally cheaper and aesthetically driven, aiming to achieve illumination effects similar to radiosity in a shorter time period.

### [Spatially Cached Raytraced Ambient Occlusion](4106-spatially-cached-raytraced-ambient-occlusion.md)
Spatially Cached Raytraced Ambient Occlusion (SC-RTAO) is an optimization for raytraced ambient occlusion that stores accumulated occlusion values in spatial cells within a hashmap. Since ambient occlusion is view-independent, these values can be accumulated into a cell without reprojection when the camera moves, reducing redundant ray tracing for static geometry across frames. It distinguishes itself from naive RTAO by caching results to avoid redundant ray tracing, and from sibling methods like Evans AO or Zhukov AO by relying on spatial hashing and atomic accumulation rather than different sampling or approximation strategies.

### Ambient Occlusion Field
Ambient Occlusion Field is a storage and reconstruction method where the reciprocal of a quadratic polynomial encodes how ambient occlusion changes with distance [S3, unofficial]. The coefficients are stored in a cube map [S3, unofficial]. This allows modeling of directional variation of occlusion and the ambient occlusion effect an object has on its surroundings [S3, unofficial]. A key pitfall is that it requires reconstructing the ambient occlusion factor at runtime, which incurs a performance penalty [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays $N_R$ and samples per ray $N_S$. For sparse voxel structures, early termination based on solid angle can reduce effective complexity [S5].
*   **Memory Complexity:** Depends on the storage method. Precomputed methods (e.g., AO Field) require cube maps or 3D textures [S3, unofficial]. Ray-traced methods require minimal extra memory beyond the scene representation.
*   **Performance:** High sample counts (e.g., 64 rays) provide high quality but are noisy with low samples (1-2) [S3, unofficial]. Denoising is often necessary for real-time performance [S3, unofficial].
*   **Strengths:** View-independent (can be precomputed or cached) [S1, unofficial]. Physically plausible darkening in crevices and corners [S1, unofficial].
*   **Limitations:** Noisy with low samples [S3, unofficial]. Computationally expensive for high-quality results without denoising or caching [S3, unofficial].
*   **When to Use:** When view-independent shading is desired, or when combined with direct illumination for realism [S1, unofficial].
*   **When Not to Use:** When real-time performance is critical and denoising/caching is not feasible, or when dynamic lighting requires per-frame updates without caching.

## Background
Ambient Occlusion was introduced by Zhukov et al. [Zhukov 1998] and Iones [2003] as an efficient technique for perceptually similar results to global illumination with lower cost [S1, unofficial]. It was detailed by Landis [2002] for adding realism to models and extended to ray tracing by Méndez et al. [2003b] [S1, unofficial]. The term "obscurance" was used to describe the integral of a distance-based function over the hemisphere [S1, unofficial]. Modern implementations leverage hardware-accelerated ray tracing (DXR, Vulkan RT) to compute the original formulation without approximations [S3, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S5] (PDF) High resolution sparse voxel DAGs - ResearchGate, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S6] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
