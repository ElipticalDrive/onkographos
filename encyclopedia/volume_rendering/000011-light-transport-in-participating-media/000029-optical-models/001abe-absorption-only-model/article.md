# Absorption-only model

Also called: Pure absorption, Absorption Model, Absorption-only Volume Rendering

## Overview
The absorption-only model is a simplified optical model for participating media where particles absorb all intercepted light and do not scatter or emit any [S1, unofficial]. It solves for light intensity attenuation along a ray, effectively modeling a medium of "cold, perfectly black particles" [S1, unofficial]. Unlike the [Standard volume rendering equation](1165-standard-volume-rendering-equation.md) or [Single Scattering Model](1053-single-scattering-model.md), it ignores in-scattering and emission, reducing the problem to a pure transmittance calculation [S3]. It is distinct from [Emission-only Model](1016-emission-only-model.md) which adds light, and [Homogeneous Medium Transmittance](1022-homogeneous-medium-transmittance.md) which is a specific case of this model for constant coefficients [S3].

## How it works
**Inputs:**
*   Ray origin $\mathbf{p}$ and direction $\boldsymbol{\omega}$ (unit vector).
*   Scalar field $f(\mathbf{x})$ representing volume density or material property.
*   Transfer function mapping $f$ to extinction coefficient $\tau$ (or absorption coefficient $\sigma_a$).
*   Background radiance $L_{bg}$ or surface emission $L_e$ at the ray exit point.

**Outputs:**
*   Final radiance $L$ at the camera.

**Data Structures:**
*   **Scalar Field:** A 3D grid of values $f$. Access is typically trilinear or nearest-neighbor interpolated [S1, unofficial].
*   **Extinction Coefficient:** $\tau(s)$ or $\sigma_a(s)$, derived from $f$. Units are $m^{-1}$ [S2, unofficial].

**Steps:**
1.  **Ray Casting:** Generate a primary camera ray for each pixel [S3].
2.  **Integration:** Integrate the extinction coefficient along the ray segment inside the volume.
    *   For a homogeneous medium with constant $\sigma_a$ and hit distance $t_{hit}$:
        $$ \tau_{total} = \sigma_a \cdot t_{hit} $$
    *   For a heterogeneous medium, compute the optical depth:
        $$ \tau_{total} = \int_0^{t_{hit}} \sigma_a(\mathbf{p} + t\boldsymbol{\omega}) \, dt $$
        [S2, unofficial]
3.  **Transmittance Calculation:** Compute the fraction of light surviving the journey:
    $$ T = \exp(-\tau_{total}) $$
    [S1, unofficial]
4.  **Compositing:** Multiply the background/surface radiance by the transmittance.
    *   If the ray hits a surface with emission $L_e$:
        $$ L = T \cdot L_e $$
        [S3]
    *   If the ray misses all surfaces (background $L_{bg}$):
        $$ L = T \cdot L_{bg} $$
        [S7, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_absorption_only(screen_pos, rng, scene, volume, transfer_function):
    # 1. Sample primary ray
    camera_ray = sample_primary(camera, screen_pos, rng)
    
    # 2. Intersect with scene
    isect = intersect(scene, camera_ray)
    
    # 3. Calculate transmittance through the volume
    # Assume volume bounds are handled by intersect or explicit check
    # For homogeneous case with constant sigma_a:
    if isect:
        t_hit = isect.distance
        sigma_a = transfer_function(volume.sample(isect.position))
        transmittance = exp(-sigma_a * t_hit)
        
        # 4. Compositing
        Le = 0
        if is_light(isect):
            Le = isect.Le
        return transmittance * Le
    else:
        # Ray misses scene, assume background or 0
        # If background exists:
        # return transmittance * background_radiance
        return 0
```
[G] In heterogeneous cases, the integral $\int \sigma_a dt$ is approximated by ray marching (summing $\sigma_a \cdot \Delta t$) or using Fourier methods [S1, unofficial].

**Parameters:**
*   $\sigma_a$ (Absorption coefficient): Units $m^{-1}$. Product of particle density $\rho$ and projected area $A$ [S2, unofficial].
*   $\tau$ (Extinction coefficient): Often used interchangeably with $\sigma_a$ in this context [S1, unofficial].

**Edge Cases:**
*   **Opacity vs. Extinction:** Opacity $\alpha$ is not $\tau$. For a voxel of side $l$, $\alpha = 1 - \exp(-\tau l)$ [S1, unofficial]. Confusing $\tau$ with $\alpha$ leads to errors when voxel size or viewing angle changes [S5, unofficial].
*   **Thresholding:** If $\tau = \infty$ (opaque), $T=0$. If $\tau = 0$, $T=1$ [S6].

## Subtopics

### [X-ray simulation via ray integration](1445-x-ray-simulation-via-ray-integration.md)
Generates DR images by integrating attenuation using HU values, producing projection-style images without scatter.

### [Prism Model](3153-prism-model.md)
Approximates absorption on triangle meshes by assigning depth based on orientation, avoiding voxelization.

### [Absorption Attenuation](7104-absorption-attenuation.md)
Strictly models probabilistic photon loss due to absorption, distinct from geometric approximations.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel for ray marching, where $N$ is the number of samples. Can be reduced to $O(1)$ per pixel using Fourier methods (Projection Slice Theorem) for homogeneous or specific heterogeneous cases [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the volume grid, where $V$ is the number of voxels.
*   **Strengths:** Computationally simple; produces X-ray-like images useful for visualizing density distributions (e.g., bone in medical imaging) [S4, book].
*   **Limitations:** No scattering or emission; cannot render volumetric effects like fog, smoke with internal lighting, or subsurface scattering [S1, unofficial].
*   **When to use:** When only density attenuation is needed, or as a base layer for more complex models.

## Background
Originated from the physics of light interaction with particulate media [S1, unofficial]. The model assumes cold, perfectly black particles [S1, unofficial]. It is a foundational concept in direct volume rendering, often cited alongside Blinn [9] and Williams and Max [5] [S1, unofficial]. Used in early medical visualizations for bone rendering and X-ray simulation [S1, unofficial].

## Sources
[S1] Optical Models for Direct Volume Rendering, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S4] Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S5] Optical Models for Direct Volume Rendering, https://docslib.org/doc/3914352/optical-models-for-direct-volume-rendering, fetched 2026-10-27 (unofficial)
[S6] Optical models for direct volume rendering, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S7] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S8] Direct_Volume_Rendering.md, https://github.com/pyvista/pyvista/files/9562449/Direct_Volume_Rendering.md, fetched 2026-10-29
