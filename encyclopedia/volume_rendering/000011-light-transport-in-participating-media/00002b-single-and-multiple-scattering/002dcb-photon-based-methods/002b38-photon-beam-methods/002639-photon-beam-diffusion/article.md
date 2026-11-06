# Photon Beam Diffusion

Also called: Photon Beam Diffusion for Rendering

## Overview
Photon Beam Diffusion (PBD) is a hybrid Monte Carlo method for rendering subsurface scattering in translucent materials that interprets incident light as a continuous beam of photons and solves the diffusion equation via numerical integration [S1, unofficial]. It eliminates the need for quantization and sum-of-Gaussians approximations used in Quantized Diffusion (QD) by treating the beam as an extended source, providing increased speed, flexibility, and numerical stability [S1, unofficial]. Unlike QD, which requires 20–60 Gaussians for equivalent quality, PBD achieves similar results with only 3–5 Monte Carlo samples [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: Material optical properties (scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, anisotropy $g$, or derived diffuse albedo $R_d$ and mean free path $1/t_0$); geometry; incident light direction and position; camera ray.
*   **Outputs**: Radiance contribution at the shading point, combining single-scattering and multiple-scattering components.

**Data Structures**
*   **Photon Beam**: Defined by origin $\mathbf{x}_b$, direction $\boldsymbol{\omega}_b$, power $\Phi_b$, and spatial extent. Unlike discrete photons, it stores entry parameters and direction, allowing it to be treated as an extended source [S1, unofficial].
*   **Profile Cache**: A table of reflectance profiles evaluated at specific distances, used to avoid redundant integration during rendering [S1, unofficial].

**Method Steps**

1.  **Beam Generation (Forward Pass)**
    Trace paths from light sources. At each scattering event, generate a continuous, directional photon beam entering the surface instead of discrete photon points [S1, unofficial]. Store the beam's origin, direction, and power. For oblique illumination, the beam's entry point and direction are recorded [S1, unofficial].

2.  **Single-Scattering Profile Calculation**
    Compute the single-scattering component $S^{(1)}$ using the beam integral.
    *   If the surface separates volumes with different indices of refraction, use a separate sample sequence starting from the critical angle, as contributions below the critical angle are zero [S1, unofficial].
    *   Use the same Monte Carlo estimator as for multiple scattering but with the single-scattering phase function and explicit Fresnel terms [S1, unofficial].
    *   This integral is exact under the assumed planar configuration and boundary conditions [S1, unofficial].

3.  **Multiple-Scattering Profile Calculation (Monte Carlo Integration)**
    Formulate the solution as a Monte Carlo integration problem of the extended source integral using the improved diffusion model [S1, unofficial].
    *   **Sampling**: Use equiangular/exponential sampling for photon beam diffusion to achieve faster convergence, especially at oblique light angles [S1, unofficial].
    *   **Step Size**: Use an exponentially-varying set of distances $x = 0, x_1, x_1 2, x_1 3, \dots$ with a step size of 1.2 and $x_1 \le 0.01/t$ [S1, unofficial].
    *   **Estimator**: View each Monte Carlo sample of the beam integral as a weighted photon (monopole) at position $\mathbf{x}_r(t_i)$ inside the medium [S1, unofficial].
    *   **Boundary Conditions**: Mirror the monopole exactly as in Photon Diffusion (PD) to compute the three boundary conditions (dipole, multipole, quadrupole) as proposed by Donner and Jensen [DJ07]. Blend between these conditions for oblique illumination on arbitrary geometries [S1, unofficial].
    *   **Integration**: Numerically integrate the diffusion contribution from these weighted monopoles along the beam trajectory [S1, unofficial].

4.  **Rendering (Backward Pass)**
    *   For each shading point, query the cached profile or compute it on the fly.
    *   Combine the single-scattering and multiple-scattering profiles to compute the final BSSRDF value [S1, unofficial].
    *   Reshape the exitant radiance by a Fresnel term, assuming the exitant boundary is diffuse but not Lambertian [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pbd_profile(material, beam, num_samples=5):
    """
    Computes the subsurface scattering profile for a given photon beam.
    Returns the combined single and multiple scattering contribution.
    """
    # Extract material parameters
    sigma_s = material.sigma_s
    sigma_a = material.sigma_a
    g = material.g
    t0 = 1.0 / (sigma_s + sigma_a) # Mean free path related parameter
    
    # Initialize accumulators
    single_scatter_contribution = 0.0
    multi_scatter_contribution = 0.0
    
    # Step 1: Single Scattering
    # If indices of refraction differ, start sampling from critical angle
    # Otherwise, start from incident angle
    start_angle = get_critical_angle(material) if material.has_ior_mismatch else beam.incident_angle
    
    for i in range(num_samples):
        # Equiangular/exponential sampling for single scattering
        # Sample distance/angle along the beam
        sample = equiangular_sample(i, num_samples, start_angle)
        
        # Compute single scattering contribution at this sample point
        # Uses explicit phase function and Fresnel terms
        ss_val = compute_single_scattering_sample(beam, sample, material)
        single_scatter_contribution += ss_val
        
    # Average the single scattering samples
    single_scatter_contribution /= num_samples
    
    # Step 2: Multiple Scattering (Diffusion)
    # Use exponentially-varying set of distances
    step_size = 1.2
    max_dist = 0.01 / t0
    distances = [0.0]
    current_dist = 1.0
    while current_dist <= max_dist:
        distances.append(current_dist)
        current_dist *= step_size
        
    # Number of samples for multiple scattering (typically 3-5)
    ms_num_samples = min(num_samples, 5)
    
    for i in range(ms_num_samples):
        # Sample a point along the beam trajectory
        # View each sample as a weighted monopole
        t = sample_distance_along_beam(i, ms_num_samples, beam.length)
        monopole_pos = beam.origin + t * beam.direction
        monopole_weight = compute_monopole_weight(beam, t, material)
        
        # Compute diffusion contribution from this monopole
        # Mirror the monopole to apply boundary conditions (dipole/multipole/quadrupole)
        # Blend boundary conditions based on incidence angle (Donner and Jensen)
        ms_val = compute_diffusion_from_monopole(monopole_pos, monopole_weight, beam, material)
        multi_scatter_contribution += ms_val
        
    # Average the multiple scattering samples
    multi_scatter_contribution /= ms_num_samples
    
    # Combine and apply Fresnel exitance reshaping
    total_profile = single_scatter_contribution + multi_scatter_contribution
    final_radiance = apply_fresnel_exitance(total_profile, material)
    
    return final_radiance

def equiangular_sample(index, total_samples, start_angle):
    """Generates an equiangular sample for single scattering."""
    # Implementation details for equiangular sampling
    # Returns angle or distance along the beam
    pass

def compute_single_scattering_sample(beam, sample, material):
    """Computes the single scattering contribution for a specific sample."""
    # Uses explicit phase function and Fresnel terms
    pass

def compute_monopole_weight(beam, t, material):
    """Computes the weight of the monopole at distance t along the beam."""
    # Attenuation by medium's scattering coefficient and transmittance
    pass

def compute_diffusion_from_monopole(monopole_pos, weight, beam, material):
    """Computes the diffusion contribution from a weighted monopole."""
    # Applies improved diffusion model
    # Mirrors monopole for boundary conditions
    pass

def apply_fresnel_exitance(profile, material):
    """Reshapes the exitant radiance by a Fresnel term."""
    pass
```

**Parameters**
*   **Number of Monte Carlo samples**: Typically 3 to 5 [S1, unofficial].
*   **Step size for exponentially-varying distances**: 1.2 [S1, unofficial].
*   **Maximum distance for exponential steps**: $0.01/t$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Oblique Illumination**: Requires careful handling of boundary conditions. Blend between dipole, multipole, and quadrupole boundary conditions as proposed by Donner and Jensen [DJ07] [S1, unofficial].
*   **Index of Refraction Mismatch**: If the surface separates volumes with different indices of refraction, use a separate sample sequence for single-scattering starting from the critical angle [S1, unofficial].
*   **Precision**: The method is numerically stable, avoiding the quantization errors of QD [S1, unofficial].

## Subtopics

### Transient Photon Beams
Extends PBD to transient rendering by including time-of-flight information in the beam storage and integration. [Transient Photon Beams](3296-transient-photon-beams.md)

### Photon Marching
Uses a marching approach to evaluate the beam contributions, potentially offering different trade-offs in memory and speed. [Photon Marching](4775-photon-marching.md)

## Operating characteristics
*   **Time Complexity**: Rendering with PBD and single-scattering typically takes about 15% longer than using the classic dipole [S1, unofficial].
*   **Speedup**: PBD achieves a speedup factor of 3.1 for a full profile when comparing 5 PBD samples with 44 Gaussians (QD). In cases where only one sample of a profile is needed, the speedup is 21.6 [S1, unofficial].
*   **Memory Complexity**: Stores continuous photon beams instead of discrete points, reducing memory overhead compared to photon mapping approaches that store large numbers of photons [S1, unofficial].
*   **Strengths**: Faster, more accurate, and numerically stable than quantized diffusion. Simpler implementation. Supports finite and multi-layer materials. Supports directional incident effects.
*   **Limitations**: Assumes the exitant boundary is diffuse (though not Lambertian). Less suitable for smooth Fresnel boundaries (marble/milk) compared to Jensen et al.'s approach [S1, unofficial].
*   **When to Use**: For translucent materials like skin, wax, or food where accurate subsurface scattering is needed with efficient computation.
*   **When Not to Use**: For materials with smooth, highly reflective Fresnel boundaries where single-scattering directionality is critical (e.g., marble, milk) [S1, unofficial].

## Background
*   **Origin**: Introduced by Ralf Habel, Per H. Christensen, and Wojciech Jarosz in 2013 at the Eurographics Symposium on Rendering [S1, unofficial].
*   **Development**: Builds on the improved diffusion model [dI11] and Photon Diffusion (PD) [DJ07]. It addresses the limitations of Quantized Diffusion (QD) by replacing sum-of-Gaussians with Monte Carlo integration [S1, unofficial].
*   **Usage**: Implemented in research and production rendering systems, including Mitsuba (point-based subsurface scattering), PRMan (point-based and ray-traced subsurface scattering), and on top of a progressive photon beam tracer [S1, unofficial].

## Sources
[S1] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
