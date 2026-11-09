# Absorption-only model

Also called: Pure absorption, Absorption Model, Absorption-only Volume Rendering

## Overview
The absorption-only model simulates a participating medium composed of cold, perfectly black particles that absorb all intercepted light without scattering or emitting [S1, unofficial][S3][S4, book][S6, S8]. It solves for the attenuation of light intensity or radiance along a ray as it traverses the medium, producing an X-ray-like projection or a "black smoke" obscuring effect [S4, book][S7, unofficial]. Unlike the [Emission-only Model](1016-emission-only-model.md) which adds light, or the [Single Scattering Model](1053-single-scattering-model.md) which redirects it, this model strictly reduces incoming radiance based on the medium's density [S1, unofficial][S4, book]. It is distinguished from [X-ray simulation via ray integration](1445-x-ray-simulation-via-ray-integration.md) by its general applicability to any light source (not just X-rays) and from the [Prism Model](3153-prism-model.md) by assuming a continuous volumetric density rather than a surface-based geometric approximation [S1, unofficial][S4, book].

## How it works
**Inputs:**
*   Ray origin $\mathbf{p}$ and direction $\boldsymbol{\omega}$ (unit vector) [S2, unofficial][S3].
*   Scene geometry (surfaces) to determine intersection points [S3].
*   Scalar field $f(\mathbf{x})$ or density distribution defining the medium [S1, S5, unofficial].
*   Transfer function mapping scalar values to extinction coefficient $\tau$ [S1, unofficial][S6].
*   Initial radiance/intensity $L_i(\mathbf{p}, \boldsymbol{\omega})$ or $I_0$ at the ray entry point [S1, S2, S7, unofficial].

**Outputs:**
*   Final radiance $L_o$ or intensity $I$ at the ray origin after attenuation [S1, unofficial][S3][S7, unofficial].

**Data Structures:**
*   **Extinction Coefficient Field:** A 3D grid or continuous function storing $\tau(\mathbf{x})$ (units: $m^{-1}$) [S1, S5, unofficial]. $\tau$ is derived from particle number density $\rho$ and projected area $A$ such that $\tau = \rho A$ [S1, S5, unofficial].
*   **Transfer Function:** A mapping $f \mapsto \tau$. Simplest case: $\tau = \infty$ if $f > K$ (threshold), else $\tau = 0$ [S6]. Or $\tau$ proportional to $f$ [S7, unofficial].

**Steps:**
1.  **Ray Casting:** Generate a primary camera ray for each screen position [S3].
2.  **Intersection:** Intersect the ray with the scene geometry to find the hit point $t_{\text{hit}}$ and surface radiance $L_e$ [S3]. If no intersection, return 0 [S3].
3.  **Integration:** Compute the optical depth $\tau_{\text{total}}$ by integrating the extinction coefficient along the ray segment from the camera to the hit point (or to infinity if no hit) [S1, S5, unofficial].
    $$ \tau_{\text{total}} = \int_0^{t_{\text{hit}}} \tau(\mathbf{p} + t\boldsymbol{\omega}) \, dt $$
    If the medium is homogeneous, $\tau(\mathbf{x}) = \sigma_a$ (constant), so $\tau_{\text{total}} = \sigma_a t_{\text{hit}}$ [S3].
4.  **Transmittance Calculation:** Calculate the fraction of light surviving transmission using the Beer-Lambert law [S1, unofficial][S3]:
    $$ T = \exp(-\tau_{\text{total}}) $$
5.  **Compositing:** Multiply the initial radiance (or surface emission radiance $L_e$) by the transmittance [S1, unofficial][S3]:
    $$ L_o = T \cdot L_i $$
    If the hit surface is a light source, $L_i = L_e$ [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_absorption_only(screen_pos, rng, scene, medium):
    """
    Renders a single pixel using the absorption-only model.
    
    Args:
        screen_pos: 2D coordinates of the pixel.
        rng: Random number generator (for sampling, though deterministic here).
        scene: Object with intersect() method.
        medium: Object with get_extinction() method.
        
    Returns:
        Radiance value.
    """
    # 1. Sample primary camera ray
    camera_ray = sample_primary(camera, screen_pos, rng)
    
    # 2. Intersect ray with scene
    isect = intersect(scene, camera_ray)
    
    if isect:
        # Distance to the hit point
        t_hit = isect.distance
        
        # 3. Integrate extinction coefficient along the ray from 0 to t_hit
        # For homogeneous medium, this simplifies to sigma_a * t_hit
        # For inhomogeneous, numerical integration (e.g., ray marching) is required
        optical_depth = 0.0
        step_size = medium.step_size
        t = 0.0
        while t < t_hit:
            # Get extinction coefficient at current position
            tau = medium.get_extinction(camera_ray.origin + camera_ray.direction * t)
            optical_depth += tau * step_size
            t += step_size
            
        # 4. Calculate transmittance
        transmittance = exp(-optical_depth)
        
        # 5. Get surface emission radiance
        Le = 0.0
        if is_light(isect):
            Le = isect.Le
            
        # 6. Return attenuated radiance
        return transmittance * Le
    else:
        # No intersection: ray escapes to infinity (or background)
        # Assuming no background emission for this simplified model
        return 0.0
```

**Parameters:**
*   $\sigma_a$ (Absorption coefficient): Units $m^{-1}$. Product of particle density $\rho$ and average projected area $A_p$ [S2, unofficial].
*   $\tau$ (Extinction coefficient): Defined as $\rho A$ [S1, S5, unofficial]. Often confused with opacity $\alpha$ [S5, unofficial][S6].
*   $K$ (Threshold): Scalar value in transfer function for binary opacity [S6].

**Edge Cases and Pitfalls:**
*   **Opacity vs. Extinction:** Opacity $\alpha$ depends on path length $l$. For a voxel of side $l$, $\alpha = 1 - \exp(-\tau l)$ [S1, unofficial][S6]. If the voxel is scaled or viewed diagonally, the path length changes, making $\alpha = 1 - \exp(-\tau l)$ invalid if $\tau$ is treated as constant opacity [S5, unofficial][S6].
*   **Small Voxel Approximation:** For small voxels, $\alpha$ can be approximated by $\min(1, \tau l)$ to ensure $\alpha \le 1$ [S1, unofficial][S6].
*   **Ray Differentials:** Ray differentials for volumetric scattering are often disabled or set to zero due to implementation complexity [S3].
*   **Environment Maps:** Often assumed absent in simplified implementations [S3].

## Subtopics

### [X-ray simulation via ray integration](1445-x-ray-simulation-via-ray-integration.md)
Generates a Digitally Reconstructed Radiograph (DR) by integrating the linear attenuation coefficient along viewing rays from a source to a film plane, producing an X-ray-like projection image. Unlike standard direct volume rendering which uses transfer functions to map scalar values to opacity for compositing, this method assumes a single energy level with no scatter, no beam hardening, and no detector response, relying strictly on the Beer-Lambert law to simulate physical attenuation. It is distinguished from its siblings by using actual Hounsfield Unit (HU) values directly in the attenuation calculation rather than arbitrary opacity mappings.

### [Prism Model](3153-prism-model.md)
The Prism Model is a surface-based optical model that simulates a thin participating volume by assigning an approximated depth to each triangle based on its orientation relative to the camera. It solves the problem of rendering volumetric absorption effects on dynamic, non-volumetric triangle meshes without requiring explicit voxelization or ray-marching through a 3D grid. It differs from its parent, the Absorption-only model, by replacing the physical particle density assumption with a geometric approximation where density is inversely proportional to triangle area and penetration depth is projected via the viewing angle.

### [Absorption Attenuation](7104-absorption-attenuation.md)
Absorption Attenuation is the optical model describing the exponential decay of radiance as light travels through a participating medium where particles absorb energy from the ray without scattering or emitting. It solves the problem of calculating the remaining light intensity after traversing a distance $d$ in a volume with a given absorption coefficient. It is distinguished from its sibling [X-ray simulation via ray integration](1445-x-ray-simulation-via-ray-integration.md) by its explicit use of the radiance differential equation and transmittance formulation, and from the [Prism Model](3153-prism-model.md) by its assumption of isotropic absorption rather than directional refraction or specific geometric prism constraints.

### [Absorbing Particle Rendering](9665-absorbing-particle-rendering.md)
Absorbing Particle Rendering simulates a participating medium composed of perfectly black particles that absorb all intercepted light without scattering or emitting. It solves for light intensity attenuation through a medium, resembling a cloud of black smoke or traditional X-ray imagery. Unlike the Density-Emitter Model, it ignores emission, and unlike the Prism Model, it treats particles as discrete absorbers rather than continuous prisms.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of integration steps along the ray [S1, S5, unofficial]. For homogeneous media, it is $O(1)$ if the distance is known [S3].
*   **Memory Complexity:** $O(V)$ for storing the volume data, where $V$ is the number of voxels [S1, unofficial].
*   **Performance:** The exponential function can be performed once per output pixel, potentially optimized via gamma correction table lookup [S1, unofficial][S6]. Fourier methods (Malzbender, Totsuka/Levoy) can compute integrals rapidly using the projection slice theorem and FFTs, avoiding per-pixel integration loops [S1, unofficial][S6].
*   **Strengths:** Simple to implement; computationally efficient for transparent or semi-transparent structures; effective for visualizing density distributions (e.g., bone in medical imaging) [S1, unofficial][S4, book][S7, unofficial].
*   **Limitations:** Does not account for scattering or emission, leading to unrealistic appearance for many media (e.g., fog, fire); assumes no inter-reflection [S1, unofficial][S4, book].
*   **When to Use:** X-ray simulation, visualization of dense structures (bone, vessels), simple smoke/fog effects where scattering is negligible, or as a base for more complex models [S1, unofficial][S4, book][S7, unofficial].
*   **When Not to Use:** Realistic rendering of participating media with significant scattering (e.g., clouds, milk) or emission (e.g., flames) [S1, unofficial][S4, book].

## Background
The absorption-only model is the simplest participating medium model, derived from the geometric optics of light interacting with small particles [S1, unofficial]. It assumes cold, perfectly black particles that absorb all intercepted light and do not scatter or emit [S1, S5, unofficial][S6]. The differential equation for intensity decay was derived by considering a cylindrical slab of medium and calculating the fraction of light occluded by particles [S1, unofficial][S4, book][S5, unofficial]. The solution is based on the Beer-Lambert law [S3][S7, unofficial]. Early medical visualizations used this model to show bone or dense tissues by thresholding scalar values [S1, unofficial][S6]. It is also used in X-ray imagery to simulate attenuation [S4, book][S7, unofficial]. The model is foundational for understanding more complex optical models like emission-absorption and scattering [S1, unofficial][S3].

## Sources
[S1] Optical Models for Direct Volume Rendering, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering, https://leeyngdo.github.io/blog/computer-graphics/2024-03-28-volume-rendering/, fetched 2026-10-27 (unofficial)
[S3] Volumetric Path Tracing, https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework2.pdf, fetched 2026-10-30
[S4] Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S5] Optical Models for Direct Volume Rendering, https://docslib.org/doc/3914352/optical-models-for-direct-volume-rendering, fetched 2026-10-27 (unofficial)
[S6] Optical models for direct volume rendering, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S7] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S8] Direct_Volume_Rendering.md, https://github.com/pyvista/pyvista/files/9562449/Direct_Volume_Rendering.md, fetched 2026-10-29
