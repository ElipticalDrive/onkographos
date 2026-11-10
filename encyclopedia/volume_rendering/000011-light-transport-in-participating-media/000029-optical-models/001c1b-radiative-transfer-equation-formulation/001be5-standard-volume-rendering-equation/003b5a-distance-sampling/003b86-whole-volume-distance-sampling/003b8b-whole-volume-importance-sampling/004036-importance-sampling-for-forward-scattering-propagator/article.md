# Importance Sampling for Forward Scattering Propagator

Also called: Global Importance Sampling, MIS Sampling Scheme

## Overview
This method provides a Monte Carlo path tracing strategy to evaluate the rendering equation for the forward scattering propagator $G_f^b$, specifically addressing the transport of light through participating media where forward scattering dominates [S1]. It differs from standard local importance sampling (which samples phase functions and exponential depth independently at each vertex) by constructing a 7-dimensional global sampling strategy that incorporates the analytical structure of the propagator, resulting in faster convergence for infinite-dimensional scattering problems [S1].

## How it works
The method evaluates the rendering equation for the propagator $G_f^b$ by importance sampling a query point $\vec{x}_i$ on the surface, directions $\hat{\omega}_e^i$ and $\hat{\omega}_i^i$, and a path length $s$ [S1]. The integral to be solved is:

$$
L_i(\vec{x}_0, \dot{\omega}_0^i) = \int \mathrm{d}s \mathrm{d}A(\vec{x}_i) \mathrm{d}^2 \dot{\omega}_i^c G_0^b(s, \vec{x}_0, -\dot{\omega}_0^i, \vec{x}_i, \dot{\omega}_i^c) |\dot{\omega}_i^c \cdot \hat{n}(\vec{x}_i)| \times \left[ L_e(\vec{x}_i, \dot{\omega}_i^c) + \int \mathrm{d}^2 \dot{\omega}_i^i L_i(\vec{x}_i, \dot{\omega}_i^i) f_s(\vec{x}_i, \dot{\omega}_i^i \leftrightarrow \dot{\omega}_i^c) |\dot{\omega}_i^i \cdot \hat{n}(\vec{x}_i)| \right]
$$

where $\vec{x}_0$ is the camera position, $\dot{\omega}_0^i$ is the incoming direction, $\vec{x}_i$ is the surface query point, $\dot{\omega}_i^c$ is the camera direction from the surface, $L_e$ is emitted radiance, $f_s$ is the surface BSDF, and $\hat{n}$ is the surface normal [S1]. The sampling space is 7-dimensional: $\vec{x}_i$ (2D surface coordinates), $s$ (1D path length), $\hat{\omega}_e^i$ (2D outgoing direction), and $\hat{\omega}_i^i$ (2D incident direction) [S1].

The algorithm uses a one-sample Multiple Importance Sampling (MIS) weighted combination of two main strategies: an **Indirect** approach (analytical approximations) and a **Direct** approach (next event estimation) [S1].

### Data Structures
*   **Surface Point $\vec{x}_i$**: 3D coordinates on the object surface.
*   **Path Length $s$**: Scalar float representing distance traveled in the medium.
*   **Directions $\hat{\omega}_e^i, \hat{\omega}_i^i$**: Unit 3D vectors.
*   **MIS Weights**: Standard balance heuristic weights computed from the PDFs of the two sampling strategies [S1].

### Steps

1.  **Sample Surface Point $\vec{x}_i$**:
    Use a one-sample MIS combination of two methods to select $\vec{x}_i$ on the surface [S1]:
    *   **Method A (Offset Projection)**: Sample an offset in a plane passing through the origin point $\vec{x}_o$ (camera/previous vertex) and project this point onto the surface geometry [S1].
    *   **Method B (Ballistic Peak)**: Sample a direction for the vector $\vec{R}$ based on the marginalization of the short length weight of the propagator (Eq. 10 in source) and shoot a ray along this direction to capture the initial ballistic peak [S1].
    *   *Note*: This step uses local information and ray-traced projection. It is performed first while the propagator marginalized over other dimensions is still broad and smooth [S1].

2.  **Indirect Sampling (Analytical Approximation)**:
    Given $\vec{x}_i$, sample the remaining variables using analytical limits of the directionally marginalized $G_f$ [S1]:
    *   **Sample $s$**: Use an MIS combination of analytical limit results for $G_f$ in the regimes of large and small $ps$ (where $p$ is a parameter related to the dipole model) [S1].
    *   **Sample $\hat{\omega}_e^i$**: Sample based on the exponential weight of $G_f$ (Eq. 9 in source) [S1].
    *   **Sample $\hat{\omega}_i^i$**: Use the boundary BSDF $f_s$ with the sampled $\hat{\omega}_e^i$ to sample the incident direction $\hat{\omega}_i^i$. Note that $\hat{\omega}_i^i$ may point back into the medium if internal reflection occurs [S1].

3.  **Direct Sampling (Next Event Estimation)**:
    This approach is MIS-weighted against the Indirect approach [S1]:
    *   **Sample $\hat{\omega}_i^i$**: Directly sample a direction towards a light source [S1].
    *   **Sample $\hat{\omega}_e^i$**: Use the adjoint BSDF with the sampled $\hat{\omega}_i^i$ to sample the direction $\hat{\omega}_e^i$ into the medium [S1].
    *   **Sample $s$**: Use a specialized path length sampler that utilizes knowledge of $\hat{\omega}_e^i$ to generate a suitable length $s$ [S1].

4.  **Throughput Management**:
    If the sampled direction reflects back into the medium, apply a simple path splitting approach based on the current path throughput to keep throughput near unity [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_forward_scattering_propagator(x0, omega0_i, surface, bsdf, lights, dipole_model):
    """
    Evaluates the rendering equation for the forward scattering propagator G_f^b.
    
    Args:
        x0: Camera/previous vertex position (vec3)
        omega0_i: Incoming direction (vec3)
        surface: Surface geometry object with intersection and projection methods
        bsdf: Surface BSDF object with sample_pdf and eval methods
        lights: List of light sources
        dipole_model: Forward scattering dipole model providing G_f approximations
        
    Returns:
        radiance: Estimated radiance L_i
    """
    
    # Step 1: Sample surface point x_i using MIS of two methods
    # Method A: Offset projection
    offset_a = sample_offset_in_plane_through(x0)
    x_i_a, valid_a = project_offset_to_surface(offset_a, surface)
    pdf_a = compute_pdf_offset_projection(offset_a, surface)
    
    # Method B: Ballistic peak sampling
    # Marginalize short length weight of Eq. 10 to sample direction R
    direction_R = sample_direction_from_short_length_weight(dipole_model)
    ray = Ray(x0, direction_R)
    hit_b = surface.intersect(ray)
    x_i_b = hit_b.point if hit_b else None
    pdf_b = compute_pdf_ballistic_sampling(direction_R, dipole_model)
    
    # Combine using MIS (one-sample balance heuristic)
    if valid_a and x_i_b is not None:
        # Both valid, choose one based on weights
        weight_a = pdf_a / (pdf_a + pdf_b)
        if random() < weight_a:
            x_i = x_i_a
            pdf_x = pdf_a
            strategy = "A"
        else:
            x_i = x_i_b
            pdf_x = pdf_b
            strategy = "B"
    elif valid_a:
        x_i = x_i_a
        pdf_x = pdf_a
        strategy = "A"
    elif x_i_b is not None:
        x_i = x_i_b
        pdf_x = pdf_b
        strategy = "B"
    else:
        return 0.0 # No intersection found

    normal = surface.normal(x_i)
    
    # Step 2 & 3: Sample s, omega_e_i, omega_i_i using Indirect and Direct strategies
    
    # --- Indirect Strategy ---
    # Sample s based on MIS of large and small ps limits
    s_indirect = sample_s_from_dipole_limits(dipole_model, x_i, x0)
    
    # Sample omega_e_i based on exponential weight of G_f (Eq. 9)
    omega_e_indirect = sample_direction_from_exponential_weight(dipole_model, x_i, x0, s_indirect)
    
    # Sample omega_i_i from BSDF given omega_e_indirect
    # Note: omega_i_i might point back into medium
    omega_i_indirect, pdf_bsdf_indirect = bsdf.sample(omega_e_indirect, x_i)
    
    # Evaluate indirect contribution
    # G_0^b is the ballistic propagator, G_f is the forward scattering propagator
    # The full integrand involves G_0^b and the BSDF term
    radiance_indirect = evaluate_indirect_path(
        x0, omega0_i, x_i, omega_e_indirect, omega_i_indirect, 
        s_indirect, normal, bsdf, dipole_model, lights
    )
    pdf_indirect = compute_pdf_indirect(s_indirect, omega_e_indirect, omega_i_indirect, dipole_model, bsdf)
    
    # --- Direct Strategy (Next Event Estimation) ---
    # Sample direction towards a light source
    light, omega_i_direct, pdf_light = sample_light_direction(x_i, lights)
    
    # Sample omega_e_i into medium using adjoint BSDF
    omega_e_direct, pdf_bsdf_direct = bsdf.sample_adjoint(omega_i_direct, x_i)
    
    # Sample s using specialized sampler given omega_e_direct
    s_direct = sample_s_specialized(dipole_model, x_i, x0, omega_e_direct)
    
    # Evaluate direct contribution
    radiance_direct = evaluate_direct_path(
        x0, omega0_i, x_i, omega_e_direct, omega_i_direct,
        s_direct, normal, bsdf, dipole_model, light
    )
    pdf_direct = compute_pdf_direct(s_direct, omega_e_direct, omega_i_direct, dipole_model, bsdf, pdf_light)
    
    # Combine Indirect and Direct using MIS
    # Total PDF for the 7D sample is the sum of the two strategy PDFs
    total_pdf = pdf_indirect + pdf_direct
    
    if total_pdf == 0:
        return 0.0
        
    # Weighted sum of contributions
    weight_indirect = pdf_indirect / total_pdf
    weight_direct = pdf_direct / total_pdf
    
    radiance = (radiance_indirect * weight_indirect / pdf_indirect) + \
               (radiance_direct * weight_direct / pdf_direct)
               
    # Handle back-scattering into medium via path splitting if necessary
    if is_direction_back_into_medium(omega_i_direct, normal):
        radiance = radiance_split_throughput(radiance, path_throughput_so_far)
        
    return radiance

def evaluate_indirect_path(x0, omega0_i, x_i, omega_e, omega_i, s, normal, bsdf, dipole_model, lights):
    """Evaluates the integrand for the indirect sampling strategy."""
    # G_0^b(s, x0, -omega0_i, x_i, omega_e) is the ballistic propagator
    G_ballistic = dipole_model.ballistic_propagator(s, x0, -omega0_i, x_i, omega_e)
    
    # Surface term: L_e + integral over incoming directions
    # For indirect, we approximate the incoming radiance L_i using the BSDF and potentially lights
    # Simplified: assume L_e is 0 or sampled, and L_i comes from BSDF evaluation
    # The term is: [ L_e(x_i, omega_e) + integral L_i(x_i, omega_i) f_s(...) |omega_i . n| ]
    
    # Evaluate BSDF
    fs = bsdf.eval(omega_i, omega_e, x_i)
    cos_theta = abs(dot(omega_i, normal))
    
    # Approximate L_i(x_i, omega_i) - in a full path tracer, this would be recursive
    # Here we assume the direct lighting contribution or previous path history
    # For the propagator evaluation, we often look at the direct term from lights
    L_i = 0.0
    for light in lights:
        if light.is_visible(x_i, omega_i):
            L_i += light.eval_radiance(x_i, omega_i)
            
    surface_term = L_i + fs * cos_theta # Simplified surface term
    
    # Integrand value
    integrand = G_ballistic * abs(dot(omega_e, normal)) * surface_term
    
    return integrand

def evaluate_direct_path(x0, omega0_i, x_i, omega_e, omega_i, s, normal, bsdf, dipole_model, light):
    """Evaluates the integrand for the direct sampling strategy."""
    G_ballistic = dipole_model.ballistic_propagator(s, x0, -omega0_i, x_i, omega_e)
    
    fs = bsdf.eval(omega_i, omega_e, x_i)
    cos_theta = abs(dot(omega_i, normal))
    
    # Direct lighting from the sampled light
    L_e_light = light.eval_radiance(x_i, omega_i)
    
    surface_term = L_e_light + fs * cos_theta
    
    integrand = G_ballistic * abs(dot(omega_e, normal)) * surface_term
    
    return integrand
```

### Parameters and Edge Cases
*   **MIS Weights**: The balance heuristic is used for the one-sample MIS [S1].
*   **Back-scattering**: If a sampled direction reflects back into the medium, use path splitting to maintain throughput near unity [S1].
*   **Approximation**: The method relies on analytical approximations of the dipole model (large and small $ps$ limits) rather than a perfect importance sampling of $G_f$ [S1].

## Operating characteristics
*   **Convergence**: Converges significantly faster than standard local importance sampling (phase function + exponential depth) because it accounts for the global geometry of the medium and the analytical form of $G_f$ [S1].
*   **Complexity**: The sampling space is 7-dimensional, which is tractable compared to the infinite-dimensional standard path tracing problem [S1].
*   **Limitations**: The method is limited to analytical sampling schemes based on asymptotic approximations of the dipole model [S1]. It requires the ability to project offsets to surfaces and sample directions based on marginalization of the propagator's short length weight [S1].
*   **Use Case**: Preferred for rendering with forward scattering dipole models where global transport effects are significant and local sampling is suboptimal [S1].

## Background
This method was introduced in the context of a forward scattering dipole model derived from a functional integral approximation [S1]. It addresses the limitations of standard path tracers for participating media by leveraging the specific structure of the forward scattering propagator $G_f^b$ [S1]. The approach is detailed in the supplementary material of the source paper due to space limitations, with an overview provided in the main text [S1].

## Sources
[S1] A forward scattering dipole model from a functional integral approximation, https://doi.org/10.1145/3072959.3073681, fetched 2026-10-26
