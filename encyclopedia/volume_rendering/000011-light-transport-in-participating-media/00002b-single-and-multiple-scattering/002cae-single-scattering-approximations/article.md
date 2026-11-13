# Single Scattering Approximations

Also called: Single-scattering model, One-bounce approximation

## Overview
Single Scattering Approximations simplify the volume rendering integral by assuming that light scatters only once within the participating medium before reaching the observer [S1]. This method solves the Radiative Transfer Equation (RTE) by neglecting the multiple scattering term, effectively treating the medium as a collection of points that scatter direct light from sources once [S1]. It sets itself apart from [Multiple Scattering Models](10687-multiple-scattering-models.md) by ignoring indirect light bounces, making it computationally efficient but inaccurate for dense media with high albedo [S6, unofficial]. It differs from [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md) by not being restricted to spherical planetary geometries or specific atmospheric phase functions, though it shares the core assumption of single-bounce transport [S1, S6, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Medium properties (extinction coefficient $\sigma_t(\mathbf{x})$, emission coefficient $\sigma_e(\mathbf{x})$, phase function $p(\boldsymbol{\omega}_o, \boldsymbol{\omega}_i)$), light source definitions (position, direction, intensity), and scene geometry (for shadowing).
*   **Outputs:** Radiance $L(\mathbf{x})$ along the view ray.

**Data Structures**
The method typically relies on a discretization of the medium. Common structures include:
*   **Voxel Grid:** A 3D array storing $\sigma_t$, $\sigma_e$, and potentially precomputed incident radiance $L_{ri}$ per voxel [S1].
*   **Shadow/Illumination Volumes:** Geometric volumes (e.g., frustums from lights) intersected by view rays to determine lit segments [S1].
*   **Integration Segments:** Line segments along the view ray that lie within illuminated regions of the medium [S1].

**Formulas**
The source radiance $J(\mathbf{x})$ is approximated by the single-scattering source radiance $J_{ss}(\mathbf{x})$, ignoring the contribution of scattered medium radiance $J_m(\mathbf{x})$ [S1]:

$$ J(\mathbf{x}) \approx J_{ss}(\mathbf{x}) = \sigma_e(\mathbf{x}) + \int_{4\pi} L_{ri}(\mathbf{x}, \boldsymbol{\omega}_i) p(\boldsymbol{\omega}_o, \boldsymbol{\omega}_i) d\boldsymbol{\omega}_i $$

where:
*   $\sigma_e(\mathbf{x})$ is the emission coefficient.
*   $L_{ri}(\mathbf{x}, \boldsymbol{\omega}_i)$ is the incident radiance arriving at $\mathbf{x}$ from direction $\boldsymbol{\omega}_i$ (typically from light sources, attenuated by transmittance).
*   $p(\boldsymbol{\omega}_o, \boldsymbol{\omega}_i)$ is the phase function.

The final radiance $L(\mathbf{x})$ at a point $\mathbf{x}$ along a ray starting at $\mathbf{x}_0$ is computed by integrating the single-scattering source term, attenuated by the transmittance from the scattering point to the eye [S1]:

$$ L(\mathbf{x}) = \tau(\mathbf{x}_0, \mathbf{x}) L(\mathbf{x}_0) + \int_{\mathbf{x}_0}^{\mathbf{x}} \tau(\mathbf{u}, \mathbf{x}) \sigma_t(\mathbf{u}) J_{ss}(\mathbf{u}) d\mathbf{u} $$

where:
*   $\tau(\mathbf{a}, \mathbf{b}) = e^{-\int_{\mathbf{a}}^{\mathbf{b}} \sigma_t(\mathbf{v}) d\mathbf{v}}$ is the transmittance between points $\mathbf{a}$ and $\mathbf{b}$ [G].
*   $L(\mathbf{x}_0)$ is the background radiance (e.g., sky or scene geometry behind the volume).

**Numbered Steps**
1.  **Discretize/Represent the Medium:** Represent the participating medium using a voxel grid, implicit function, or geometric volumes [S1].
2.  **Compute Incident Radiance ($L_{ri}$):** For each point (or voxel) in the medium, compute the radiance arriving from light sources. This involves ray tracing from the point to the light sources, accounting for occlusion by scene geometry and attenuation through the medium [S1].
    *   *Note:* In analytic methods (e.g., Blinn), this integral is solved analytically for specific geometries (lights/viewers at infinity) [S1]. In deterministic methods (e.g., Kajiya & Von Herzen), this is computed per voxel [S1].
3.  **Compute Single-Scattering Source ($J_{ss}$):** Combine emission and the integral of incident radiance weighted by the phase function [S1].
    *   For isotropic scattering, the integral simplifies to $\pi L_{ri}$ if $L_{ri}$ is uniform over the hemisphere, or requires numerical integration over the sphere otherwise [S1, G].
4.  **Integrate along View Ray:** For each camera ray, traverse the medium.
    *   Identify **lit segments**: portions of the ray where the medium is illuminated by light sources [S1].
    *   For each lit segment, accumulate the contribution: $\sigma_t(\mathbf{u}) J_{ss}(\mathbf{u}) \tau(\mathbf{u}, \mathbf{x}) d\mathbf{u}$ [S1].
    *   Apply transmittance from the camera to the current point $\mathbf{u}$ to the background radiance [S1].
5.  **Handle Shadows/Glow:** Explicitly account for shadow volumes cast by geometry into the medium to determine which segments are lit (Max's method) or use ray casting to determine visibility (Nishita's method) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering_render(ray_origin, ray_direction, medium, lights, geometry):
    """
    Renders a single ray using single-scattering approximation.
    """
    # 1. Initialize accumulators
    transmittance_to_eye = 1.0
    accumulated_radiance = 0.0
    
    # 2. Traverse the medium (e.g., via ray marching or segment intersection)
    # Assume 'segments' are precomputed lit intervals along the ray
    for segment in medium.get_lit_segments(ray_origin, ray_direction, lights, geometry):
        start, end = segment.start, segment.end
        
        # Calculate transmittance from eye to segment start
        tau_start = exp(-integrate_extinction(ray_origin, start, medium))
        
        # Numerical integration over the segment
        # In practice, this might be a sum over voxels or analytical evaluation
        segment_contribution = 0.0
        for u in segment.sample_points():
            # Transmittance from scattering point u to eye
            tau_u_to_eye = exp(-integrate_extinction(u, ray_origin, medium))
            
            # Get incident radiance at u from lights (precomputed or computed on fly)
            L_ri = compute_incident_radiance(u, lights, geometry, medium)
            
            # Compute single scattering source J_ss
            # Simplified for isotropic scattering: J_ss = sigma_e + sigma_s * L_ri
            sigma_t = medium.get_extinction(u)
            sigma_a = medium.get_absorption(u)
            sigma_s = sigma_t - sigma_a
            sigma_e = medium.get_emission(u)
            
            # Phase function p(wo, wi) integrated over incoming directions
            # For isotropic: integral is 4*pi * (1/4pi) * p = 1 * p? 
            # Standard phase function normalization: integral over 4pi is 4pi? 
            # Usually p is normalized such that integral is 4pi for isotropic p=1.
            # Let's assume p is the phase function value for the specific geometry.
            # For simplicity, assume L_ri is the average incident radiance weighted by phase.
            J_ss = sigma_e + sigma_s * L_ri # Simplified scalar approximation
            
            # Add contribution: J_ss * sigma_t * tau(u, eye) * du
            # Note: sigma_t * J_ss is the source term in the RTE
            du = segment.step_size
            segment_contribution += J_ss * sigma_t * tau_u_to_eye * du
            
        # Add segment contribution to total, attenuated by transmittance to segment start
        accumulated_radiance += tau_start * segment_contribution
        
        # Update transmittance to eye for the next segment (or background)
        transmittance_to_eye *= exp(-integrate_extinction(start, end, medium))
        
    # 3. Add background radiance
    background_radiance = geometry.get_radiance_at(ray_origin, ray_direction)
    final_radiance = transmittance_to_eye * background_radiance + accumulated_radiance
    
    return final_radiance

def compute_incident_radiance(point, lights, geometry, medium):
    """
    Computes L_ri at a point by summing contributions from lights.
    """
    L_ri = 0.0
    for light in lights:
        # Vector to light
        to_light = light.position - point
        dist = length(to_light)
        direction = normalize(to_light)
        
        # Check occlusion by geometry
        if not geometry.is_occluded(point, light.position):
            # Transmittance from point to light
            tau_point_to_light = exp(-integrate_extinction(point, light.position, medium))
            
            # Light intensity (simplified)
            intensity = light.intensity / (dist * dist) # Inverse square law
            
            # Phase function p(wo, wi) where wo is towards eye, wi is towards light
            # Note: In the integral, we integrate over all incoming directions.
            # Here we approximate with discrete lights.
            phase = medium.phase_function(direction, -ray_direction) # wo is -ray_direction? 
            # Actually, L_ri is incident from wi. The scattering redirects wi to wo.
            # So p(wo, wi) is needed.
            
            L_ri += intensity * tau_point_to_light * phase
            
    return L_ri
```

**Parameters**
*   **Albedo ($\Omega = \sigma_s / \sigma_t$):** The method is valid when $\Omega$ is low. Blinn states multiple scattering cannot be neglected when $\Omega > 0.3$ [S1].
*   **Optical Depth:** Valid for optically thin media where transmittance is near one [S1].

**Edge Cases and Pitfalls**
*   **High Albedo Media:** Single scattering fails for clouds (albedo 0.7–0.9), rendering them dark and lifeless [S1, S2].
*   **Glow:** Standard single scattering ignores the "glow" effect (light scattering into the view ray from multiple bounces) [S1]. Max's method attempts to approximate this glow using shadow volumes [S1].
*   **Boundaries:** Analytic methods often assume infinite or simple layered media; handling complex boundaries requires numerical integration or voxelization [S1].

## Subtopics

### [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md)
Computes the color of the sky and atmospheric haze by numerically integrating the in-scattered light from a single distant light source (the sun) along a view ray passing through a spherical atmosphere shell. It solves the RTE under the assumption that light scatters only once, accounting for Rayleigh and Mie scattering, as well as ozone absorption. It sets itself apart from [Multiple Scattering Models](10687-multiple-scattering-models.md) by ignoring indirect light bounces, making it computationally efficient but inaccurate for deep atmospheres or high optical depths. It differs from [Single Scattering Approximation](5264-single-scattering-approximation.md) by specifically modeling the spherical geometry of a planetary atmosphere with exponential density profiles.

### [Microflake Model](4739-microflake-model.md)
The Microflake Model is a physically-based participating medium model that represents a medium as a homogeneous cloud of microscopic, perfectly specular flakes. It solves the problem of accurately rendering anisotropic media (such as fur, hair, or certain fabrics) by deriving direction-dependent attenuation and phase functions directly from a microflake normal distribution function (NDF). Unlike generic anisotropic phase functions which often violate Helmholtz reciprocity, the Microflake Model satisfies reciprocity by construction. It is distinct from the [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md) which typically assumes isotropic or simple Henyey-Greenstein scattering, and from the [Single Scattering Approximation](5264-single-scattering-approximation.md) which may use arbitrary phase functions without physical grounding in surface geometry.

### [Single Scattering Approximation](5264-single-scattering-approximation.md)
The Single Scattering Approximation simplifies the volume rendering integral by assuming low albedo and/or density, such that the probability of light scattering more than once is small. Only single scattering is considered, and glow is ignored.

### [Single Scattering](5566-single-scattering.md)
Single scattering is an approximation of light transport in participating media where in-scattered radiance is assumed to originate directly from light sources, similar to direct lighting in vacuum. It solves the problem of computing volumetric illumination by integrating over the spherical domain of incident light directions, effectively treating the medium as a collection of points that scatter light from lights once before reaching the camera. This method is distinct from multiple scattering models which account for light bouncing within the medium, and from atmosphere-specific approximations that rely on precomputed textures or simplified phase functions [Siblings].

### [Real-Time Volumetric Shadows using 1D Min-Max Mipmaps](9426-real-time-volumetric-shadows-using-1d-min-max-mipmaps.md)
This method computes real-time volumetric shadows in single-scattering participating media on the GPU by evaluating the scattering integral over the intersections of camera rays with a shadow map treated as a 2D height field. It distinguishes itself from sibling methods by using epipolar rectification to reduce the problem to 1D height fields and employing 1D min-max mipmaps to find lit segments in parallel, allowing all camera rays to be processed simultaneously using only standard pixel shader features without requiring GPGPU APIs or camera rectification.

### [Multiple Scattering Models](10687-multiple-scattering-models.md)
Multiple scattering models compute the radiance contribution from light that undergoes two or more scattering events within a participating medium before reaching the eye. While single-scattering approximations suffice for thin media like haze, they fail for dense media such as clouds or smoke, where photons bounce dozens or hundreds of times, causing the medium to reflect significant light (e.g., up to 90% for thick cumulus clouds). These models solve the problem of simulating this internal light transport, which is computationally expensive because every additional bounce multiplies the work required to trace rays. They are distinguished from single-scattering siblings by their attempt to capture indirect lighting and energy redistribution within the volume, rather than only direct light attenuation and one-time deflection.

### [Optically Thin Approximation Correction](12649-optically-thin-approximation-correction.md)
The Optically Thin Approximation Correction improves the accuracy of the single-scattering approximation by modeling photon loss due to absorption and secondary scattering along the light path. It addresses the error introduced when assuming no attenuation between the light source, the scattering volume, and the observer. Unlike the base [Single Scattering Approximation](5264-single-scattering-approximation.md), it applies exponential reduction factors for optical depth along the Sun-to-scatterer path and the scatterer-to-observer path.

### Analytical Single Scattering without Boundaries
Proposes an analytical solution for single scattering by participating media without boundaries under a point light [S3]. Can achieve a real-time frame rate [S3].

### Differentiable Radiance Derivative Computation
A method to compute radiance derivatives with respect to density analytically, avoiding gradient tracking and redundant operations inherent in automatic differentiation (AD) [S4, unofficial]. It derives an analytical form of the radiance transfer equation. Derives discrete forms of transmittance $\tau$ and single-scattering radiance $L_{ss}$ from their continuous integral forms to facilitate computer program implementation [S4, unofficial]. Unlike automatic differentiation (AD) which propagates gradients through long computational graphs carrying inherent overhead, this method directly computes derivatives analytically, avoiding gradient tracking and redundant operations [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the implementation. Analytic methods are $O(1)$ per pixel for simple geometries but limited in applicability [S1]. Deterministic voxel-based methods are $O(N)$ where $N$ is the number of voxels traversed, plus the cost of computing incident radiance for each voxel (which may involve ray tracing to lights) [S1]. Stochastic methods (Monte Carlo) converge as $O(1/\sqrt{N})$ but require many samples for noise-free results in complex lighting [S1].
*   **Memory Complexity:** Voxel-based methods require $O(V)$ memory for the voxel grid, where $V$ is the number of voxels [S1]. Analytic methods require minimal memory [S1].
*   **Measured Performance:** Analytic methods can achieve real-time frame rates for simple scenes (e.g., Sun in atmosphere) [S3]. Deterministic methods are faster than Monte Carlo but limited to specific media types [S1].
*   **Strengths:** Computationally efficient compared to multiple scattering; suitable for optically thin media (haze, fog) [S1, S2].
*   **Limitations:** Inaccurate for dense media (clouds, smoke) with high albedo; ignores glow and indirect illumination [S1, S2].
*   **When to Use:** Thin media, real-time applications where multiple scattering is too expensive, or as a base layer for hybrid methods.
*   **When Not to Use:** Dense clouds, smoke, or any medium where $\Omega > 0.3$ or optical depth is high [S1].

## Background
Single scattering approximations were among the first methods developed for realistic rendering of participating media. Blinn [11] was the first researcher concerned with the visualization of participating media, specifically for cloud layers and atmospheres, solving the integral transport equation analytically for single scattering with lights and viewers at infinity [S1]. Willis [106] focused on flight simulators, using a simple model with isotropic scattering and constant power emission [S1]. Kajiya and Von Herzen [41] extended Blinn's model for ray tracing, eliminating viewing and lighting restrictions by using a voxel model [S1]. Max [54] introduced scan-line based methods dealing with shadow volumes to account for glow in haze [S1]. Nishita et al. [62] further developed these ideas using ray casting and shadow/illumination volumes [S1]. These methods are generally deterministic and impose strong limitations on the media, making them difficult to adapt to general situations [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
[S3] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S4] Physics-Based Differentiable Rendering for Efficient and ... - MDPI, https://www.mdpi.com/1099-4300/25/9/1348, fetched 2026-10-27 (unofficial)
