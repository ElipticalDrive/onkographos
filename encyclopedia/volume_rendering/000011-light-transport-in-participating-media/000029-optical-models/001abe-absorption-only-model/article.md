# Absorption-only model

Also called: Pure absorption, Absorption Model, Absorption-only Volume Rendering

## Overview
The absorption-only model renders a participating medium composed of cold, perfectly black particles that absorb all intercepted light without scattering or emitting [S1, unofficial][S4, book][S6]. It solves for light intensity attenuation along a ray, producing an X-ray-like projection or a "black smoke" obscuring a background [S4, book][S7, unofficial]. Unlike the [Emission-only Model](1016-emission-only-model.md) which adds radiance, or the [Single Scattering Model](1053-single-scattering-model.md) which redirects light, this model considers only the exponential decay of incident radiance [S3, S8]. It is distinguished from [X-ray simulation via ray integration](1445-x-ray-simulation-via-ray-integration.md) by its general applicability to any scalar field mapped to extinction, rather than strictly physical X-ray attenuation, and from [Absorbing Particle Rendering](9665-absorbing-particle-rendering.md) by its continuous medium formulation [S1, unofficial][S3].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar field $f(\mathbf{x})$ representing density or material property; a transfer function mapping $f$ to the extinction coefficient $\tau(\mathbf{x})$; a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$; and an incident radiance $L_i$ at the ray entry point (or background intensity $I_0$) [S1, unofficial][S3][S7, unofficial].
*   **Output:** The transmitted radiance $L_o$ at the ray origin (or pixel) [S3][S7, unofficial].

**Data Structures**
*   **Scalar Field:** A 3D grid or continuous function $f(\mathbf{x})$ [S1, S5, unofficial].
*   **Extinction Coefficient:** A scalar field $\tau(\mathbf{x})$ derived from $f(\mathbf{x})$ via a transfer function. Units are $m^{-1}$ [S2, S5, unofficial].
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (unit vector) [S3].

**Formulas**
The differential change in radiance $L$ over a path length $dt$ is governed by:
$$ \frac{dL}{dt} = -\tau(\mathbf{r}(t)) L(\mathbf{r}(t)) $$
where $\tau(\mathbf{r}(t))$ is the extinction coefficient at position $\mathbf{r}(t)$ [S1, unofficial][S3][S7, unofficial].

The closed-form solution for radiance at the ray origin ($t=0$), given incident radiance $L_i$ at entry point $t_{entry}$ and exit point $t_{exit}$, is:
$$ L_o = L_i \exp\left( -\int_{t_{entry}}^{t_{exit}} \tau(\mathbf{r}(t)) \, dt \right) $$
If the ray intersects a surface at distance $t_{hit}$, the incident radiance $L_i$ is the surface emission $L_e$ at that point [S3]. The term $T = \exp\left( -\int \tau \, dt \right)$ is the transmittance [S1, unofficial][S3].

The opacity $\alpha$ of a voxel of side $l$ viewed parallel to an edge, assuming constant $\tau$, is:
$$ \alpha = 1 - \exp(-\tau l) $$
For small voxels, this is often approximated as $\alpha \approx \min(1, \tau l)$ [S1, unofficial][S6].

**Parameters**
*   $\tau(\mathbf{x})$: Extinction coefficient. Can be constant (homogeneous) or varying (inhomogeneous) [S3][S7, unofficial].
*   $I_0$ or $L_i$: Initial intensity/radiance. Can be a constant background or pixel-dependent [S7, unofficial][S3].

**Steps**
1.  **Sample Ray:** Generate a primary camera ray for the screen position [S3].
2.  **Intersect Scene:** Determine the entry and exit points of the ray within the volume or the intersection with a surface [S3].
    *   If the ray hits a surface at $t_{hit}$, calculate the transmittance from the ray origin to $t_{hit}$ [S3].
    *   If the ray misses the scene, return 0 (or background) [S3].
3.  **Integrate Extinction:** Compute the optical depth $\tau_{opt} = \int_{t_{entry}}^{t_{exit}} \tau(\mathbf{r}(t)) \, dt$.
    *   For homogeneous media, $\tau_{opt} = \tau \cdot (t_{exit} - t_{entry})$ [S3].
    *   For inhomogeneous media, sample $\tau$ along the ray and sum (e.g., trilinear interpolation of scalar field $f$ mapped to $\tau$) [S1, S5, unofficial].
4.  **Compute Transmittance:** Calculate $T = \exp(-\tau_{opt})$ [S3].
5.  **Compute Output:**
    *   If a surface was hit: $L_o = T \cdot L_e(\mathbf{p}_{hit})$ [S3].
    *   If no surface (background): $L_o = T \cdot I_0$ [S7, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_absorption_only(screen_pos, camera, scene, volume):
    # 1. Sample ray
    ray = sample_primary(camera, screen_pos)
    
    # 2. Intersect with scene to find hit point and surface emission
    isect = intersect(scene, ray)
    
    if isect:
        # Distance from ray origin to hit point
        t_hit = isect.distance
        
        # 3. Integrate extinction along the ray segment [0, t_hit]
        # For homogeneous medium with constant sigma_a:
        optical_depth = volume.sigma_a * t_hit
        
        # For inhomogeneous medium, integrate tau(t) along the ray
        # optical_depth = integrate(volume.get_extinction, ray, 0, t_hit)
        
        # 4. Compute transmittance
        transmittance = exp(-optical_depth)
        
        # 5. Get surface emission radiance
        Le = isect.material.emission_radiance
        
        # Return attenuated surface radiance
        return transmittance * Le
    else:
        # No intersection: return attenuated background
        # Assume background intensity I0 is available or 0
        I0 = get_background_intensity(screen_pos)
        
        # Integrate through the entire volume or to a far plane
        # Assuming volume extends to t_far
        t_far = volume.far_plane_distance
        optical_depth = volume.sigma_a * t_far
        
        transmittance = exp(-optical_depth)
        
        return transmittance * I0
```

**Edge Cases and Pitfalls**
*   **Opacity vs. Extinction:** Do not confuse the extinction coefficient $\tau$ (units $m^{-1}$) with voxel opacity $\alpha$ (dimensionless). $\alpha$ depends on the path length through the voxel. If a voxel is scaled or viewed diagonally, the path length differs from the edge length $l$, making $\alpha = 1 - \exp(-\tau l)$ invalid if $\tau$ is treated as constant opacity [S1, unofficial][S6].
*   **Thresholding:** For binary volumes, $\tau$ can be set to $\infty$ for opaque regions and $0$ for transparent ones. This allows rendering via z-buffer or painter's algorithm without integration [S1, unofficial][S6].
*   **Precision:** The exponential function can be computed once per pixel. In early implementations, gamma correction tables were used for this lookup [S1, unofficial][S6].

## Subtopics

### [X-ray simulation via ray integration](1445-x-ray-simulation-via-ray-integration.md)
Simulates X-ray attenuation using Hounsfield Units directly, assuming single energy level and no scatter. Chosen for medical DR generation.

### [Prism Model](3153-prism-model.md)
Approximates absorption on triangle meshes by mapping depth to density. Chosen for dynamic surfaces without voxelization.

### [Absorption Attenuation](7104-absorption-attenuation.md)
The fundamental process of radiance reduction. Chosen as the base compositing step in more complex models.

### [Absorbing Particle Rendering](9665-absorbing-particle-rendering.md)
Simulates discrete black particles. Chosen for specific particle-based effects like smoke clouds.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray. For homogeneous media, $O(1)$ if analytical integration is used [S3].
*   **Memory Complexity:** $O(V)$ for the scalar field storage, where $V$ is the volume size [S1, unofficial].
*   **Performance:** Fastest of the optical models. The exponential term can be precomputed or looked up [S1, unofficial][S6].
*   **Strengths:** Physically accurate for non-emitting, non-scattering media (e.g., X-rays, black smoke). Simple to implement [S4, book][S7, unofficial].
*   **Limitations:** Cannot render volumetric lighting, shadows, or color bleeding caused by scattering. Unsuitable for realistic clouds or fire [S3][S4, book].
*   **When to use:** X-ray visualization, simple occlusion effects, or as a component in hybrid models [S4, book][S7, unofficial].

## Background
The absorption-only model is the simplest participating medium model, derived from the geometric optics of light interacting with particles [S1, unofficial]. It was formalized in the context of direct volume rendering by Max [S1, unofficial][S6] and Blinn [S1, S7, unofficial]. The differential equation for intensity decay is analogous to neutron transport absorption cross-sections [S7, unofficial]. Early implementations used Fourier methods (Malzbender, Totsuka/Levoy) to compute integrals rapidly via the projection slice theorem [S1, unofficial][S6]. It is foundational for understanding more complex models like emission-absorption and scattering [S1, unofficial][S3].

## Sources
[S1] Optical Models for Direct Volume Rendering, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S4] Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S5] Optical Models for Direct Volume Rendering, https://docslib.org/doc/3914352/optical-models-for-direct-volume-rendering, fetched 2026-10-27 (unofficial)
[S6] Optical models for direct volume rendering, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S7] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S8] Direct_Volume_Rendering.md, https://github.com/pyvista/pyvista/files/9562449/Direct_Volume_Rendering.md, fetched 2026-10-29
