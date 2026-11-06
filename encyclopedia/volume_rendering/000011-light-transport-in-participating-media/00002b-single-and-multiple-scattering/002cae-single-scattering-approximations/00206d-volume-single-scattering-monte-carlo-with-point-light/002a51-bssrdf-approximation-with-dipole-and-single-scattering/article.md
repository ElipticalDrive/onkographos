# BSSRDF Approximation with Dipole and Single Scattering

Also called: BSSRDF Model with Dipole Diffusion Approximation, BSSRDF Rendering with Dipole Diffusion and Single Scattering, BSSRDF approximation, Dipole BSSRDF, First-order scattering, Hanrahan and Krueger extension, Practical Model for Subsurface Light Transport, Single Scattering Term

## Overview
This method renders translucent materials by combining a dipole diffusion approximation for multiple scattering with a first-order single scattering term, addressing the problem of light entering and exiting a medium at different positions [S1, unofficial]. It sets itself apart from pure surface BRDF models by accounting for subsurface diffusion and from full volumetric path tracing by using an analytical approximation to compute diffuse reflectance, offering a balance between physical accuracy and computational efficiency [S1, unofficial]. It builds on [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md) by replacing the pure Monte Carlo single-scattering integration with an analytical single-scattering term and adding a dipole-based multiple scattering component.

## How it works
The method computes outgoing radiance $L_o(x_o, \omega_o)$ at a surface point $x_o$ in direction $\omega_o$ by integrating the incident radiance over the surface area and incoming directions, using the BSSRDF $S$ [S1, unofficial]:

$$
L_o (x_o , \omega_o) = \int_A \int_{2\pi} S(x_i , \omega_i ; x_o , \omega_o ) (\omega_i \cdot \omega_o) L_i (x_i , \omega_i ) d\omega_i dA(x_i )
$$

The BSSRDF $S$ relates outgoing radiance at $x_o$ to incident flux at $x_i$, allowing light to enter and exit at different positions [S1, unofficial]. The model approximates $S$ using a dipole diffusion solution for the diffuse component and an analytical term for single scattering.

### Inputs and Outputs
**Inputs:**
*   Surface point $x_o$ and outgoing direction $\omega_o$.
*   Incident point $x_i$ and incident direction $\omega_i$ (for the single scattering term).
*   Material properties: Absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, mean cosine of scattering angle $g$, relative index of refraction $\eta$.
*   Incident radiance $L_i(x_i, \omega_i)$.
*   Random number $\xi \in ]0, 1]$ for single scattering distance sampling.

**Outputs:**
*   Outgoing radiance $L_o(x_o, \omega_o)$.

### Data Structures
No specific complex data structures are required beyond standard material property storage. The key parameters are:
*   $\sigma_a$: Absorption coefficient.
*   $\sigma_s$: Scattering coefficient.
*   $g$: Mean cosine of scattering angle.
*   $\eta$: Relative index of refraction.
*   Derived parameters:
    *   Reduced scattering coefficient: $\sigma_{s0} = \sigma_s (1 - g)$.
    *   Reduced extinction coefficient: $\sigma_{t0} = \sigma_{s0} + \sigma_a$.
    *   Diffusion constant: $D = \frac{1}{3\sigma_{t0}}$.
    *   Effective transport coefficient: $\sigma_{tr} = \sqrt{3\sigma_a \sigma_{t0}}$.

### Steps
1.  **Compute Diffuse BSSRDF ($R_d$) using Dipole Approximation:**
    The diffuse reflectance $R_d$ is computed using a dipole source configuration with a real source at distance $z_r = 1/\sigma_{t0}$ below the surface and a virtual source at distance $z_v$ [S1, unofficial]. The fluence $\phi$ at a point $x$ is given by:
    $$
    \phi(x) = \frac{\Phi}{4\pi D} \left( \frac{e^{-\sigma_{tr} r(x)}}{r(x)} - \frac{e^{-\sigma_{tr} r_v(x)}}{r_v(x)} \right)
    $$
    where $\Phi$ is the source power, $r(x)$ is the distance to the real source, and $r_v(x)$ is the distance to the virtual source. The diffuse reflectance $R_d$ is derived from the radiant exitance at the surface, which is related to the gradient of the fluence [S1, unofficial]. For a flat surface, $R_d$ can be approximated using a rational function of $\eta$ [S1, unofficial]:
    $$
    F_{dr} = \frac{1.440}{\eta^2} + \frac{0.710}{\eta} + 0.668 + 0.0636\eta
    $$
    The final $R_d$ value depends on the distance between $x_i$ and $x_o$ and the material parameters.

2.  **Compute Single Scattering Term:**
    For the single scattering component, pick a random distance $s_{0o}$ along the refracted outgoing ray:
    $$
    s_{0o} = \frac{\log(\xi)}{\sigma_t(x_o)}
    $$
    where $\xi \in ]0, 1]$ is a uniformly distributed random number [S1, unofficial].

3.  **Approximate Refracted Distance for Incoming Ray:**
    Approximate the refracted distance $s_{0i}$ for shadow rays that do not refract at the surface (assuming the light source is far away compared to the mean free path) using Snell's law:
    $$
    s_{0i} = s_i \frac{|\omega_i \cdot n_i|}{\sqrt{1 - (\eta^2 (1 - |\omega_i \cdot n(x_i)|^2))}}
    $$
    Here $s_i$ is the observed distance, $\eta$ is the index of refraction, $n_i$ is the normal at the outgoing point, and $n(x_i)$ is the normal at the incoming point [S1, unofficial].

4.  **Compute Outgoing Radiance:**
    The outscattered radiance $L_o^{(1)}(x_o, \omega_o)$ is computed as:
    $$
    L_o^{(1)}(x_o, \omega_o) = \left[ \frac{\sigma_s(x_o) F p(\omega_i \cdot \omega_o)}{\sigma_{tc}} \right] e^{-s_{0i} \sigma_t(x_i)} e^{-s_{0o} \sigma_t(x_o)} L_i(x_i, \omega_i)
    $$
    where $\sigma_s$ is the scattering coefficient, $F = F_t(\eta, \omega_o) F_t(\eta, \omega_i)$ is the product of Fresnel transmission terms, $p$ is the phase function, $\sigma_{tc}$ is the total cross section, $s_{0i}$ is the refracted distance through the material, $s_{0o}$ is the distance along the outgoing ray, and $L_i$ is the incident radiance [S1, unofficial].

5.  **Combine Terms:**
    The total outgoing radiance is the sum of the diffuse BSSRDF contribution and the single scattering term. The BSSRDF contribution is integrated over the surface area, while the single scattering term is evaluated for specific light paths.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_bssrdf_dipole(sigma_a, sigma_s, g, eta, xi, xo, xo_normal, xi_pos, xo_pos):
    """
    Computes the diffuse BSSRDF component using the dipole approximation.
    """
    sigma_s0 = sigma_s * (1 - g)
    sigma_t0 = sigma_s0 + sigma_a
    D = 1.0 / (3.0 * sigma_t0)
    sigma_tr = math.sqrt(3.0 * sigma_a * sigma_t0)
    
    # Distance between xi and xo
    r = math.norm(xi_pos - xo_pos)
    
    # Real source distance
    zr = 1.0 / sigma_t0
    
    # Virtual source distance (simplified dipole model)
    # In the full Hanrahan-Krueger model, zv is derived from boundary conditions.
    # Here we use a simplified approximation for the dipole term.
    # The fluence at the surface is approximated by the difference of two exponentials.
    
    # Note: The exact calculation of Rd involves integrating the fluence gradient.
    # A common approximation for Rd(r) is:
    # Rd(r) = (1 / (4 * pi)) * ( (1 + sigma_tr * dr) * exp(-sigma_tr * dr) / dr^2 
    #                           - (1 + sigma_tr * dv) * exp(-sigma_tr * dv) / dv^2 )
    # where dr and dv are distances to real and virtual sources.
    
    dr = math.sqrt(r**2 + zr**2)
    # Virtual source position is typically at -zv below the surface.
    # For a semi-infinite medium, zv is often approximated.
    # Using a simplified virtual source distance for demonstration.
    zv = 1.0 / sigma_tr # Approximation
    dv = math.sqrt(r**2 + zv**2)
    
    term1 = (1 + sigma_tr * dr) * math.exp(-sigma_tr * dr) / (dr**2)
    term2 = (1 + sigma_tr * dv) * math.exp(-sigma_tr * dv) / (dv**2)
    
    Rd = (1.0 / (4.0 * math.pi)) * (term1 - term2)
    
    return Rd

def compute_single_scattering(sigma_s, sigma_t, eta, p, xi, xo, xo_normal, xi_normal, Li, xi_pos, xo_pos, ray_length):
    """
    Computes the single scattering term.
    """
    # Fresnel transmission
    Ft_o = fresnel_transmission(eta, xo_normal, xo)
    Ft_i = fresnel_transmission(eta, xi_normal, xi)
    F = Ft_o * Ft_i
    
    # Phase function
    phase = p(xi, xo)
    
    # Refracted distances
    # s0o is sampled randomly
    xi_rand = random.uniform(0.001, 1.0)
    s0o = math.log(xi_rand) / sigma_t
    
    # s0i is approximated using Snell's law
    # Assuming si is the distance to the light source or intersection
    si = ray_length 
    dot_ni = abs(dot(xi, xi_normal))
    dot_nxo = abs(dot(xi, xo_normal))
    
    denominator = math.sqrt(1 - (eta**2 * (1 - dot_nxo**2)))
    if denominator == 0:
        s0i = si
    else:
        s0i = si * dot_ni / denominator
        
    # Attenuation
    attenuation = math.exp(-s0i * sigma_t) * math.exp(-s0o * sigma_t)
    
    # Single scattering radiance
    # Note: sigma_tc is total cross section, often same as sigma_t in this context
    sigma_tc = sigma_t
    
    Lo_single = (sigma_s * F * phase / sigma_tc) * attenuation * Li
    
    return Lo_single

def fresnel_transmission(eta, normal, direction):
    """
    Simplified Fresnel transmission approximation.
    """
    # Use Schlick's approximation for reflectance, then 1 - R for transmission
    cos_theta = abs(dot(normal, direction))
    R = schlick_reflectance(eta, cos_theta)
    return 1.0 - R

def schlick_reflectance(eta, cos_theta):
    """
    Schlick's approximation for Fresnel reflectance.
    """
    r0 = ((1 - eta) / (1 + eta))**2
    return r0 + (1 - r0) * (1 - cos_theta)**5

def render_bssrdf(xo, xo_normal, xi_pos, xi_normal, Li, material_params):
    """
    Main rendering function for BSSRDF with dipole and single scattering.
    """
    sigma_a = material_params['sigma_a']
    sigma_s = material_params['sigma_s']
    g = material_params['g']
    eta = material_params['eta']
    
    # Compute diffuse BSSRDF component
    Rd = compute_bssrdf_dipole(sigma_a, sigma_s, g, eta, None, xo, xo_normal, xi_pos, xo_pos)
    
    # Compute single scattering component
    # Note: In a full implementation, single scattering is integrated over light paths.
    # Here we assume a point light and a specific shadow ray.
    Lo_single = compute_single_scattering(
        sigma_s, sigma_a + sigma_s, eta, phase_function, 
        xi, xo, xo_normal, xi_normal, Li, xi_pos, xo_pos, ray_length
    )
    
    # Combine terms
    # The BSSRDF term is integrated over the surface, so Rd is multiplied by incident flux.
    # For a point light, this simplifies to Rd * Li * cos(theta) * solid_angle.
    # The single scattering term is added directly.
    
    # Simplified combination for demonstration
    Lo = Rd * Li + Lo_single
    
    return Lo
```

### Parameters
*   $\sigma_a$: Absorption coefficient. Typical values depend on the material (e.g., skin, wax).
*   $\sigma_s$: Scattering coefficient. Typical values depend on the material.
*   $g$: Mean cosine of scattering angle. Typical values for skin are around 0.7-0.9.
*   $\eta$: Relative index of refraction. Typical value for skin is around 1.4.
*   $\xi$: Random number for single scattering distance sampling.

### Edge Cases and Pitfalls
*   **Hair Darkness:** The hair at the back of the head appears slightly darker in the BSSRDF simulation compared to full Monte Carlo, attributed to the forced $1/\sigma_{t0}$ distance in the diffusion approximation [S1, unofficial].
*   **Highly Curved Surfaces:** Special care must be taken in the presence of highly curved surfaces. The diffusion term is evaluated with a minimum distance of $1/\sigma_{t0}$ to eliminate singularities at sharp edges where the source can be placed arbitrarily close to $x_o$ [S1, unofficial].
*   **Precision:** The exponential terms in the dipole approximation can lead to numerical instability if not handled carefully, especially for large distances or high absorption coefficients.

## Subtopics

### [BSSRDF Model](6674-bssrdf-model.md)
The BSSRDF (Bidirectional Surface Scattering Reflectance Distribution Function) model approximates subsurface light transport by treating the surface as a boundary where light enters, diffuses within the medium, and exits at a different location. It solves the problem of rendering translucent materials (like skin, wax, or marble) where light penetrates the surface, scatters, and re-emerges. It sets itself apart from pure surface BRDF models by accounting for subsurface diffusion and from full volumetric path tracing by using an analytical approximation (typically the dipole model) to compute the diffuse reflectance, offering a balance between physical accuracy and computational efficiency.

## Operating characteristics
*   **Time Complexity:** The method is significantly faster than full Monte Carlo simulation. For example, rendering a milk glass took 4 minutes with BSSRDF compared to 1250 minutes for full Monte Carlo [S1, unofficial]. Diffuse (BRDF) rendering took 2 minutes, while BSSRDF renderings took 4 minutes each, including caustics and global illumination [S1, unofficial].
*   **Memory Complexity:** The method does not require storing photons or a full 3D description of the material, unlike photon mapping [S1, unofficial].
*   **Strengths:** Fast rendering time, physically plausible appearance for translucent materials, handles subsurface diffusion.
*   **Limitations:** Approximation errors in highly curved surfaces or near sharp edges, potential darkness artifacts in specific regions (e.g., hair), limited to single scattering for the non-diffuse component.
*   **When to Use:** Suitable for rendering translucent materials like skin, wax, marble, and milk where real-time or near-real-time performance is needed, and full volumetric path tracing is too slow.
*   **When Not to Use:** Not suitable for highly anisotropic scattering or when high accuracy in complex geometries is required without approximation errors.

## Background
The model was presented in a paper for the SIGGRAPH conference proceedings, funded by NSF (IIS-0085864) and DARPA (DABT63-95-C-0085) [S1, unofficial]. It extends the work of Hanrahan and Krueger on subsurface scattering, providing a practical model for subsurface light transport. The diffusion approximation is based on the observation that the light distribution in highly scattering media tends to become isotropic, allowing the radiance to be approximated by a two-term expansion involving the radiant fluence and the vector irradiance [S1, unofficial].

## Sources
[S1] Practical Model for Subsurface Light Transport, https://www.scribd.com/document/904578145/A-Practical-Model-for-Subsurface-Light-Transport, fetched 2026-10-29 (unofficial)
