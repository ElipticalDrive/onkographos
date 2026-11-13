# Photon-based Estimation

Also called: Photon Gathering, Photon Density Estimation

## Overview
Photon-based estimation methods solve the light transport problem in participating media by decoupling the simulation of light paths from the final image synthesis. Instead of integrating the Radiative Transfer Equation (RTE) directly via ray marching or path tracing, these methods store photon interactions (positions, directions, and energy) in a spatial data structure and estimate radiance at render time via density estimation [S1]. This approach is particularly effective for capturing complex multiple-scattering effects, caustics, and indirect illumination that are difficult to resolve with local illumination models or standard Monte Carlo path tracing due to high variance [S2, S5].

## Shared principles
Photon-based estimation relies on the following shared mechanisms and mathematical foundations:

1.  **Photon Emission and Tracing:** Photons are emitted from light sources and traced through the scene. When a photon enters a participating medium, it undergoes a random walk. The step size $t$ between interactions is sampled from the extinction coefficient $\sigma_t$:
    $$ t = -\frac{\ln(r)}{\sigma_t} $$
    where $r$ is a uniform random number in $[0, 1]$ [S3, unofficial].

2.  **Interaction Handling:** At each step, the photon's fate is determined by the ratio of absorption $\sigma_a$ and scattering $\sigma_s$ to the total extinction $\sigma_t = \sigma_a + \sigma_s$.
    *   **Scattering:** The photon's direction is updated using a phase function $p(\omega_i, \omega_o)$ (e.g., Henyey-Greenstein), and its energy is attenuated by the scattering albedo $\omega = \sigma_s / \sigma_t$.
    *   **Absorption:** The photon is terminated or its energy is reduced.
    *   **Null Collisions:** In heterogeneous media, a majorant $\sigma_{max} \ge \sigma_t(\mathbf{x})$ is used. Steps are sampled using $\sigma_{max}$, and a null collision occurs with probability $P_{null} = (\sigma_{max} - \sigma_t)/\sigma_{max}$, where the photon continues without interaction but with transmittance adjustment [S3, unofficial].

3.  **Storage:** Interactions are stored in a spatial data structure. Common structures include:
    *   **KD-Trees:** For efficient nearest-neighbor queries [S1, S4, unofficial].
    *   **Volume Textures:** Grid-based storage for GPU acceleration, trading precision for speed and memory locality [S1, S2, unofficial].
    *   **BVHs:** For beam-based representations to handle overlapping paths [S1].

4.  **Density Estimation (Gathering):** During rendering, for a point $\mathbf{x}$ on a camera ray, the in-scattered radiance $L_s(\mathbf{x})$ is estimated by gathering nearby stored photons. The general form is:
    $$ L_s(\mathbf{x}) \approx \frac{1}{N} \sum_{i=1}^{N} \frac{P_i}{V_i} K\left(\frac{\|\mathbf{x} - \mathbf{p}_i\|}{r}\right) p(\omega_i, \omega_o) $$
    where:
    *   $N$ is the number of photons in the search radius $r$.
    *   $P_i$ is the power of the $i$-th photon.
    *   $V_i$ is the volume of influence or a normalization factor.
    *   $K$ is a kernel function (e.g., uniform or Gaussian) [S4, S6, unofficial].
    *   $p(\omega_i, \omega_o)$ is the phase function evaluating the likelihood of scattering from the photon's incoming direction $\omega_i$ to the camera direction $\omega_o$ [S1].

5.  **Integration:** The estimated in-scattered radiance is integrated along the camera ray using the transmittance $T$:
    $$ L(\mathbf{x}_0, \omega_0) = L_0 T(\mathbf{x}_0, \mathbf{x}_n) + \sum_{j=1}^{n} T(\mathbf{x}_0, \mathbf{x}_{j-1}) \sigma_s(\mathbf{x}_j) L_s(\mathbf{x}_j) \Delta s_j $$
    [S1].

## Subtopics

### [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md)
A GPU-accelerated Monte Carlo method that processes photons in parallel using compute shaders. It differs from traditional photon mapping by directly updating radiance feedback buffers per-frame rather than storing photons in a spatial structure for later density estimation. This makes it suitable for dynamic scenes and real-time feedback. It uses a fixed number of null-collision steps per photon and incremental radiance accumulation.

### [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md)
A global illumination technique that decouples indirect illumination simulation from direct illumination. It solves the problem of computing realistic in-scattering and absorption in complex volumes (e.g., medical ultrasound) by storing photon interactions in a spatial data structure (often a volume texture for GPU optimization) and estimating radiance via density estimation. It sets itself apart from beam-based methods by using discrete point-to-point density estimation.

### [Photon Beam Storage in BVH](2275-photon-beam-storage-in-bvh.md)
Organizes photon beams into a Bounding Volume Hierarchy (BVH) to accelerate spatial queries during volumetric radiance estimation. It solves the problem of efficiently retrieving all beams intersecting a specific eye ray. Unlike brute-force scans, it modifies BVH construction to handle highly overlapping beams (common from point sources) by splitting them into sub-beams with unit aspect ratios.

### [Point-Based Light Transport in Participating Media](3763-point-based-light-transport-in-participating-media.md)
A global illumination method for homogeneous translucent materials with refractive boundaries. It decorrelates scene complexity from illumination computation by replacing the scene with a shaded point cloud (volume samples) for indirect illumination. It computes single, double, and multiple scattering contributions separately using precomputed light surface and volume samples, handling refraction at boundaries which breaks screen-space coherence.

### Generalized Photon Surface Estimation
A method by Deng et al. that generalizes photon planes to photon surfaces, introducing estimators like photon cone, cylinder, and sphere, combined with Multiple Importance Sampling (MIS) for robustness [S1]. It couples the light and camera subpaths using a delta kernel to avoid bias [S1]. It reduces variance significantly compared to prior work and handles any scattering event, including single scattering [S1]. However, it is unable to handle medium interactions immediately following scattering from a surface, such as volumetric caustics [S1].

## Background
Photon-based methods for participating media originated from the adaptation of photon mapping techniques, originally developed for surface rendering by Jensen [G]. Early applications focused on homogeneous media, but the complexity of heterogeneous media and multiple scattering led to the development of specialized estimators. The survey by [S1] highlights the evolution from simple photon planes to more complex photon surfaces and beam-based representations to handle singularities and improve variance. The method is widely used in offline rendering for realistic effects like caustics in fog or smoke, and in real-time applications via GPU-accelerated variants like Compute Shader Photon Integration.

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
