# Single Scattering Model

Also called: Heidelberg ray tracing model, Kajiya and Von Herzen model, Single Scattering Radiance Transport, Single-Scattering Participating Media Rendering, Two-Pass Single Scattering

## Overview
The Single Scattering Model approximates light transport in participating media by accounting only for light that undergoes zero or exactly one scattering event, ignoring multiple scattering and emitted radiance [S1, S2][S3, unofficial]. It solves the problem of rendering volumetric effects like smoke or fluids efficiently by decomposing the outgoing radiance into direct transmission and a single-scattering term [S2]. It is distinguished from the [Absorption-only model](839-absorption-only-model.md) by including scattering phase functions and from the [Emission-Absorption Volume Rendering](4040-emission-absorption-volume-rendering.md) by explicitly modeling the redirection of light via a phase function and scattering albedo [S2][S3, unofficial].

## How it works
**Inputs:**
*   Viewpoint position $x$.
*   Viewing direction $\omega$ (unit vector).
*   Volume density field $\rho(x)$ or directly extinction coefficients.
*   Optical properties: Absorption coefficient $\kappa_a(x)$, Scattering coefficient $\kappa_s(x)$, Extinction coefficient $\kappa_t(x) = \kappa_a(x) + \kappa_s(x)$, Scattering albedo $\Omega(x) = \kappa_s(x)/\kappa_t(x)$ [S2].
*   Phase function $p(\omega, \omega')$ depending on the angle between viewing direction $\omega$ and light direction $\omega'$ [S2][S3, unofficial].
*   Incident ambient radiance $L_{in}(\omega)$ from the environment or light sources [S2].
*   Volume boundary points $x_a$ (entry) and $x_b$ (exit) along the viewing ray, at distances $D_a$ and $D_b$ from the viewpoint $x$ [S2].

**Outputs:**
*   Outgoing radiance $L_{out}(x, \omega)$ at the viewpoint [S2].

**Data Structures:**
*   **Density Field:** A 3D grid or voxel array storing $\rho(x)$ or directly $\kappa_t(x)$ [S1, S2].
*   **Illumination Field:** A 3D grid storing the incoming illumination $I(X)$ at each voxel $X$, computed in Pass 1 [S3, unofficial].

**Formulas:**
1.  **Transmittance:** The transmittance $\tau(t_1, t_2)$ between two points at distances $t_1$ and $t_2$ along a ray is the exponential of the negative integral of the extinction coefficient along the path [S2]:
    $$ \tau(t_1, t_2) = e^{-\int_{t_1}^{t_2} \kappa_t(x_u) du} $$
    where $x_u$ is a point along the ray at distance $D_u$ from the viewpoint [S2].

2.  **Outgoing Radiance Decomposition:**
    $$ L_{out}(x, \omega) = L_{ri}(x, \omega) + L_{ss}(x, \omega) \quad (1) $$
    where $L_{ri}$ is the reduced incident radiance (direct transmission) and $L_{ss}$ is the single-scattering radiance [S1, S2].

3.  **Reduced Incident Radiance:**
    $$ L_{ri}(x, \omega) = L_{in}(\omega) \tau(D_a, 0) \quad (2) $$
    This represents ambient light arriving directly at the viewpoint after passing through the volume boundary $x_a$ [S2].

4.  **Single-Scattering Radiance:**
    $$ L_{ss}(x, \omega) = \int_{D_a}^{D_b} \tau(D_u, 0) \kappa_s(x_u) I(x_u) du \quad (3) $$
    This integral sums contributions from all points $x_u$ along the ray, weighted by the transmittance from the viewpoint to $x_u$, the scattering coefficient, and the illumination $I(x_u)$ reaching $x_u$ from light sources [S2][S3, unofficial].

**Algorithm (Two-Pass Method):**
The model is often implemented using a two-pass approach, particularly for complex lighting or when precomputing illumination [S3, unofficial].

**Pass 1: Compute Illumination $I(X)$**
Compute the illumination $I(X)$ reaching each point $X$ in the volume from the light source(s). This involves propagating flux from the light source through the volume, accounting for the transparency (attenuation) of each layer [S3, unofficial].
*   For simple point or parallel lights, $I(X)$ can be calculated directly [S1].
*   For environment map lighting, $I(X)$ is estimated using importance sampling over the unit sphere $S^2$ [S1].
*   Mathematically, for a point $X$, the illumination is attenuated by the transmittance from the light source to $X$ [S3, inferred from context, unofficial].

**Pass 2: Gather Scattered Radiance**
Reflect or scatter the precomputed illumination $I(X)$ to the viewpoint using a shading rule $g(X)$ [S3, unofficial].
*   The shading rule for scattering by particles is:
    $$ g(X) = r(X, \omega, \omega') I(X) $$
    where $r(X, \omega, \omega') = a \tau(X) p(\omega, \omega')$ [S3, unofficial]. Here $a$ is the albedo (fraction of extinction representing scattering), $\tau(X)$ is the transmittance from $X$ to the viewpoint, and $p$ is the phase function [S3, unofficial].
*   Gather the reflected intensity $g(X)$ along viewing rays:
    $$ L_{ss}(x, \omega) = \int_{D_a}^{D_b} g(x_u) du $$
    This corresponds to integrating the scattered contribution along the viewing ray [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def single_scattering_render(viewpoint, direction, kappa_t_grid, kappa_s_grid, light_radiance, light_direction, phase_function):
    """
    Renders a volume using the single scattering model.
    direction and light_direction must be normalized 3D vectors.
    kappa_t_grid and kappa_s_grid are 3D arrays with a sample(pos) method.
    """
    # 1. Find volume boundaries along the viewing ray
    t_near = None
    t_far = None
    t = 0.0
    search_step = 0.1
    max_ray_length = 100.0
    while t < max_ray_length:
        pos = [viewpoint[i] + t * direction[i] for i in range(3)]
        if kappa_t_grid.sample(pos) > 0.0:
            if t_near is None:
                t_near = t
            t_far = t
        elif t_near is not None:
            break
        t += search_step
    
    if t_near is None:
        return 0.0

    # 2. Compute direct transmission (Reduced Incident Radiance)
    # L_ri = L_in(omega) * tau(t_near, 0)
    L_in_ambient = light_radiance # Simplified: assume uniform ambient or pass as arg
    optical_depth_to_entry = 0.0
    t = 0.0
    while t < t_near:
        pos = [viewpoint[i] + t * direction[i] for i in range(3)]
        optical_depth_to_entry += kappa_t_grid.sample(pos) * 0.05
        t += 0.05
    tau_entry = math.exp(-optical_depth_to_entry)
    L_ri = L_in_ambient * tau_entry

    # 3. Compute single-scattering radiance via ray marching
    L_ss = 0.0
    optical_depth = 0.0
    t = t_near
    march_step = 0.05
    while t < t_far:
        pos = [viewpoint[i] + t * direction[i] for i in range(3)]
        kappa_s = kappa_s_grid.sample(pos)
        
        # Transmittance from current point to viewpoint
        tau_to_view = math.exp(-optical_depth)
        
        # Illumination at pos from light source
        # Transmittance from light to pos is pre-integrated or approximated for directional lights [G]
        light_transmittance = math.exp(-integrate_extinction_along_ray([viewpoint[i] + 1000.0 * light_direction[i] for i in range(3)], pos, kappa_t_grid))
        I_xu = light_radiance * light_transmittance
        
        # Phase function for scattering from light_direction to viewing direction
        p_val = phase_function(direction, light_direction)
        
        # Contribution: tau * kappa_s * I_xu * p_val
        L_ss += tau_to_view * kappa_s * I_xu * p_val * march_step
        
        optical_depth += kappa_t_grid.sample(pos) * march_step
        t += march_step

    return L_ri + L_ss

def integrate_extinction_along_ray(start, end, kappa_t_grid):
    """Computes optical depth from start to end."""
    direction = [end[i] - start[i] for i in range(3)]
    dist = math.sqrt(sum(d**2 for d in direction))
    if dist < 1e-6:
        return 0.0
    direction = [d / dist for d in direction]
    optical_depth = 0.0
    t = 0.0
    step = 0.05
    while t < dist:
        pos = [start[i] + t * direction[i] for i in range(3)]
        optical_depth += kappa_t_grid.sample(pos) * step
        t += step
    return optical_depth
```

## Subtopics

### [Single Scattering Propagator](1299-single-scattering-propagator.md)
Models light transport with zero or exactly one scattering event, serving as the "direct" component. It rigorously includes volume attenuation and phase function integration, unlike the Utah Approximation which ignores occlusion and attenuation between the light source and the voxel [1299-single-scattering-propagator.md].

### [Single Scattering (Blinn Model)](4581-single-scattering-blinn-model.md)
A model for participating media with low albedo, considering only one secondary ray traversing the material towards the light source [4581-single-scattering-blinn-model.md].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the viewing ray and the volume size [G]. The two-pass method adds a precomputation step proportional to the volume size and light complexity [S3, unofficial].
*   **Memory Complexity:** Requires storage for the volume density/extinction field and potentially an illumination field if precomputed [S1][S3, unofficial].
*   **Performance:** Significantly faster than full multiple scattering models because it avoids iterative solutions or deep path tracing for multiple bounces [S1, S2].
*   **Strengths:** Physically plausible for low-density media (smoke, fog, fluids) where single scattering dominates [S2]. Enables efficient differentiable rendering for reconstruction tasks [S1, S2].
*   **Limitations:** Invalid for high-density or high-albedo media where multiple scattering is significant [S3, unofficial]. Does not account for emitted radiance (e.g., fire, glowing plasma) unless added separately [S1].
*   **When to use:** Smoke reconstruction, fluid visualization, medical imaging (e.g., CT/MRI volume rendering with scattering approximation) [S2][S3, unofficial].
*   **When not to use:** Dense fog, clouds with high optical depth, scenes with strong multiple scattering effects, or media with significant self-emission [S3, unofficial].

## Background
The Single Scattering Model originates from the work of Kajiya and Von Herzen, who proposed a two-pass ray tracing method for volume rendering [S3, unofficial]. This approach was later referred to as the "Heidelberg ray tracing model" when applied systematically to medical images by Meinzer et al. [S3, unofficial]. The model is based on the Radiative Transfer Equation (RTE) but simplifies it by assuming single scattering dominance [S1, S2]. It has been widely used in differentiable rendering for fluid and smoke reconstruction due to its computational efficiency and differentiability [S1, S2].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
[S2] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
[S3] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
