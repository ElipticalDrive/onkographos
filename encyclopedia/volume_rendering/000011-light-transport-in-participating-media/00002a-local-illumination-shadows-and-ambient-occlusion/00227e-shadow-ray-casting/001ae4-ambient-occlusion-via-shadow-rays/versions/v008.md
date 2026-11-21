# Ambient Occlusion via Shadow Rays

Also called: Ray-traced AO, Shadow Ray Ambient Occlusion

## Overview
Ambient Occlusion via Shadow Rays approximates the lack of secondary (reflected) light rays by casting shadow rays into the hemisphere above a surface point to determine geometric obscurance [S1, unofficial][S2, unofficial]. It differs from direct illumination methods by integrating over the hemisphere rather than a single light direction, and from screen-space approximations (SSAO) by using explicit ray tracing against the scene geometry or volume [S3, unofficial]. This method provides view-independent occlusion values that can be combined with arbitrary direct illumination [S1, unofficial].

## How it works
**Inputs:** Surface point $\mathbf{p}$, surface normal $\mathbf{n}$, volume density field or scene geometry, number of rays $N_R$, number of samples per ray $N_S$ (or total samples $N$), and optional falloff parameters.
**Outputs:** Ambient occlusion factor $A \in [0, 1]$ or radiance contribution $L_r$.

**Data Structures:**
*   **Hemisphere Sampling:** A set of $N_R$ directions $\omega_i$ distributed over the hemisphere oriented by $\mathbf{n}$. Distributions include Uniform Solid Angle or Cosine-Weighted [S2, unofficial].
*   **Visibility Function:** $V(\mathbf{p}, \omega)$ is binary: 1 if the path from $\mathbf{p}$ in direction $\omega$ is clear, 0 if occluded [S1, unofficial].
*   **Distance Function (Obscurance):** $\rho(d)$ maps the distance to the first intersection to an attenuation value [S1, unofficial]. In simplified Ambient Occlusion, this is replaced by the binary visibility function [S1, unofficial].

**Steps:**
1.  **Coordinate Frame Construction:** Construct a local tangent space at $\mathbf{p}$ using the normal $\mathbf{n}$. Compute tangent $\mathbf{t}_0$ and bitangent $\mathbf{t}_1$ [S6, unofficial].
    *   If $|n_z| > 0.5$, set $\mathbf{t}_0 = (-n_y, n_x, 0)$, else $\mathbf{t}_0 = (1, 0, 0)$ [S6, unofficial].
    *   Normalize $\mathbf{t}_0$.
    *   Compute $\mathbf{t}_1 = \mathbf{n} \times \mathbf{t}_0$ [S6, unofficial].
2.  **Direction Sampling:** For each ray $i \in [0, N_R-1]$, sample a direction $\omega_i$ in the hemisphere using the chosen distribution (e.g., cosine-weighted or uniform solid angle) [S2, unofficial][S6, unofficial]. Transform $\omega_i$ from local tangent space to world space: $\mathbf{d}_i = \mathbf{t}_0 \omega_{i,x} + \mathbf{t}_1 \omega_{i,y} + \mathbf{n} \omega_{i,z}$ [S6, unofficial].
3.  **Ray Casting:** For each direction $\mathbf{d}_i$, cast a shadow ray from $\mathbf{p}$ (offset slightly along $\mathbf{n}$ to avoid self-intersection) [S2, unofficial].
    *   Check for intersection with scene geometry or volume density [S2, unofficial][S3, unofficial].
    *   Determine visibility $V_i$: 1 if no intersection within the maximum distance bound $d_{max}$, 0 otherwise [S1, unofficial].
    *   *Optimization:* In sparse voxel structures, traversal may terminate early if a node subtends a solid angle below a threshold [S5].
4.  **Integration:** Compute the ambient occlusion factor.
    *   **Binary Visibility (Standard AO):**
        $$ A(\mathbf{p}) = \frac{1}{\pi} \int_{\Omega} V(\mathbf{p}, \omega) \cos \theta \, d\omega \approx \frac{1}{N} \sum_{i=1}^{N} V_i \cos \theta_i $$
        where $\theta_i$ is the angle between $\mathbf{n}$ and $\mathbf{d}_i$ [S1, unofficial][S2, unofficial].
    *   **Distance-Based (Obscurance):**
        $$ O(\mathbf{p}) = \frac{1}{\pi} \int_{\Omega} \rho(d(\mathbf{p}, \omega)) \cos \theta \, d\omega $$
        where $\rho$ is a falloff function of the distance to the first hit [S1, unofficial].
    *   **Volume Density Implementation:** For volumetric data, sample density along the ray. Accumulate attenuation based on density thresholds [S4, unofficial].
        $$ \text{occlusion} = \sum_{i=0}^{N_R-1} \sum_{j=0}^{N_S-1} \frac{-7.0 \cdot I_{ij}}{N_R N_S} \cdot \mathbb{I}(T_{min} \le I_{ij} \le T_{max}) $$
        where $I_{ij}$ is the density at sample $j$ of ray $i$ [S4, unofficial].
5.  **Final Color:** Combine the occlusion factor with ambient lighting.
    $$ L_{ambient} = L_{sky} \cdot A(\mathbf{p}) $$
    Or in the specific implementation from [S6, unofficial]:
    $$ \text{outputColor} = \text{ambientColor} \cdot t^{\text{ambientExponent}} + \text{constantColor} $$
    where $t = \frac{\text{hitDist}}{24.0}$ [S6, unofficial].

**Parameters:**
*   $N_R$: Number of shadow rays per pixel. Typical values: 10 [S4, unofficial], 64 [S5], or 1-2 samples per surface for real-time hardware ray tracing [S3, unofficial].
*   $N_S$: Samples per ray. Typical value: 5 [S4, unofficial].
*   $d_{max}$: Maximum distance for occlusion calculation. [S1, unofficial].
*   $\text{ambientExponent}$: Exponent for falloff. Typical value: 1.3 [S6, unofficial].
*   $\text{ambientColor}$: Color of ambient light. Typical value: `vec3(1.0)` [S6, unofficial].
*   $\text{constantColor}$: Constant ambient to prevent total darkness. Typical value: `vec3(0.003)` [S6, unofficial].

**Pitfalls:**
*   **Noise:** Low sample counts (e.g., 1-2 samples) lead to very noisy images [S3, unofficial]. Denoising algorithms (spatial/temporal convolutional or deep learning) are required [S3, unofficial].
*   **Precision:** Small gradients in volume data can lead to incorrect normals if not clamped [S4, unofficial].
*   **Performance:** Computational complexity scales with scene complexity [S3, unofficial].

## Subtopics

### [Evans Ambient Occlusion](1083-evans-ambient-occlusion.md)
Evans Ambient Occlusion approximates ambient occlusion by sampling a Signed Distance Field (SDF) along the surface normal to estimate sky visibility. It measures occlusion by comparing the distance field value at sampled points against the actual distance traveled from the surface. It is chosen when SDFs are available and fast approximation is needed.

### [Incremental Local Ambient Occlusion](1089-incremental-local-ambient-occlusion.md)
Incremental Local Ambient Occlusion (LAO) computes ambient lighting by integrating attenuated transfer function density over a local spherical neighborhood around each voxel, rather than using a constant ambient factor. It solves the problem of missing local shadowing in direct volume rendering while avoiding global shadows. Unlike siblings, LAO-I updates the occlusion result incrementally, processing one ray direction per frame and storing the result in an ambient light texture using OpenGL blend operations, allowing for user interaction during computation.

### [Solid Angle Early Termination for Ambient Occlusion](1146-solid-angle-early-termination-for-ambient-occlusion.md)
Solid Angle Early Termination for Ambient Occlusion accelerates shadow ray traversal in sparse voxel structures by stopping traversal when a node subtends a solid angle below a specified threshold, effectively using a lower level-of-detail representation for distant points. It differs from Evans and Zhukov AO by relying on geometric projection thresholds rather than hemisphere sampling or precomputed angular distributions, and from LAO by using a static threshold rather than an adaptive accumulation scheme.

### [Zhukov Ambient Occlusion](2245-zhukov-ambient-occlusion.md)
Zhukov Ambient Occlusion is an empirical illumination model that simulates indirect lighting by computing the geometric obscurance of a surface point, representing how open it is to the environment. Unlike radiosity, it attenuates the constant ambient term of the Phong model based on local geometric occlusion, prioritizing computational speed and aesthetic appeal over physical accuracy. It builds on Ambient Occlusion via Shadow Rays by incorporating a cosine-weighted hemisphere integration directly into the sampling scheme.

### [Spatially Cached Raytraced Ambient Occlusion](4106-spatially-cached-raytraced-ambient-occlusion.md)
Spatially Cached Raytraced Ambient Occlusion (SC-RTAO) is an optimization for raytraced ambient occlusion that stores accumulated occlusion values in spatial cells within a hashmap. Since ambient occlusion is view-independent, these values can be accumulated into a cell without reprojection when the camera moves, reducing redundant ray tracing for static geometry across frames. It distinguishes itself from naive RTAO by caching results to avoid redundant ray tracing, and from sibling methods like Evans AO or Zhukov AO by relying on spatial hashing and atomic accumulation rather than different sampling or approximation strategies.

### Ambient Occlusion Field
Ambient Occlusion Field is a storage and reconstruction method where the reciprocal of a quadratic polynomial encodes how ambient occlusion changes with distance. Coefficients are stored in a cube map [S3, unofficial]. The data structure is a cube map storing coefficients of the reciprocal of a quadratic polynomial [S3, unofficial]. A pitfall is that it requires reconstructing the ambient occlusion factor at runtime, which incurs a performance penalty [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_R \cdot N_S \cdot T)$, where $T$ is the cost of ray traversal/intersection. Scales linearly with the number of rays and samples [S5].
*   **Memory Complexity:** $O(1)$ per pixel for the calculation, plus storage for the scene/volume. Caching methods (SC-RTAO) add $O(C)$ for cell storage [S1, unofficial].
*   **Measured Performance:** Shooting 64 shadow rays per pixel provides high quality but is expensive [S5]. Real-time implementations often use 1-2 samples with denoising [S3, unofficial].
*   **Strengths:** View-independent, physically plausible (when using obscurance), works with arbitrary direct illumination [S1, unofficial].
*   **Limitations:** Noisy with low samples, computationally expensive for high-quality results, requires efficient ray tracing infrastructure [S3, unofficial].
*   **When to use:** When view-independent ambient occlusion is required and hardware ray tracing is available, or for offline precomputation.
*   **When not to use:** When real-time performance is critical and denoising is not an option, or for dynamic scenes where caching is ineffective.

## Background
Ambient Occlusion was introduced by Zhukov et al. [Zhukov 1998] and Iones [2003] as an efficient technique for perceptually similar results to global illumination with lower cost [S1, unofficial]. The term "obscurances" was used to describe the measure of the part of the hemisphere obscured by neighboring surfaces [S1, unofficial]. Landis [2002] detailed its use for adding realism to models, and Méndez et al. [2003b] extended it to ray tracing [S1, unofficial]. The method is a property of geometry, independent of lighting conditions and viewpoint [S1, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S4] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S5] (PDF) High resolution sparse voxel DAGs - ResearchGate, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S6] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
