# BSSRDF Approximation with Dipole and Single Scattering

Also called: BSSRDF Model with Dipole Diffusion Approximation, BSSRDF Rendering with Dipole Diffusion and Single Scattering, BSSRDF approximation, Dipole BSSRDF, First-order scattering, Hanrahan and Krueger extension, Practical Model for Subsurface Light Transport, Single Scattering Term

## Overview
This method approximates subsurface light transport by combining a dipole diffusion model for multiple scattering with a single-scattering term, evaluating the Bidirectional Surface Scattering Reflectance Distribution Function (BSSRDF) at the surface boundary [S1, unofficial]. It solves the problem of computationally expensive volumetric path tracing in participating media under rough dielectric surfaces by replacing volumetric integration with an analytical diffusion-based solution [S1, unofficial]. It sets itself apart from its siblings, such as [Ray Marching with In/Out-Scattering and Density Scaling](2318-ray-marching-with-in-out-scattering-and-density-scaling.md) and [Analytical Approximation of Single Scattering Radiance](4244-analytical-approximation-of-single-scattering-radiance.md), by explicitly modeling the spatial variation of light entry and exit points via the dipole configuration, whereas siblings often assume uniform density or precomputed terms without the dipole's specific boundary handling [S1, unofficial].

## How it works
The method computes outgoing radiance by integrating incident radiance over the surface area, using the BSSRDF to relate incident flux at point $x_i$ to outgoing radiance at point $x_o$ [S1, unofficial]. The BSSRDF $S$ is approximated using a dipole solution to the diffusion equation for the multiple scattering component, supplemented by a single-scattering term for near-surface effects [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Surface geometry: Points $x_i$ (incident) and $x_o$ (outgoing), normals $n(x_i)$ and $n(x_o)$.
- Material properties: Absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, mean cosine of scattering angle $g$, relative index of refraction $\eta$.
- Lighting: Incident radiance $L_i(x_i, \omega_i)$, incident direction $\omega_i$.
- Sampling: Random number $\xi \in ]0, 1]$ for single scattering distance.

**Outputs:**
- Outgoing radiance $L_o(x_o, \omega_o)$.

### Data Structures
The method relies on scalar and vector quantities derived from the radiative transfer equation. No complex volumetric data structures are required; only surface points and material constants are needed [S1, unofficial].

Key parameters:
- $\sigma_t = \sigma_s + \sigma_a$: Extinction coefficient.
- $\sigma_{s0} = \sigma_s (1 - g)$: Reduced scattering coefficient.
- $\sigma_{t0} = \sigma_{s0} + \sigma_a$: Reduced extinction coefficient.
- $D = \frac{1}{3\sigma_{t0}}$: Diffusion constant.
- $\sigma_{tr} = \sqrt{3\sigma_a \sigma_{t0}}$: Effective transport coefficient.

### Formulas
1. **BSSRDF Definition:**
   $$ dL_o(x_o, \omega_o) = S(x_i, \omega_i; x_o, \omega_o) d\Phi_i(x_i, \omega_i) $$
   $$ L_o(x_o, \omega_o) = \int_A \int_{2\pi} S(x_i, \omega_i; x_o, \omega_o) (n \cdot \omega_i) L_i(x_i, \omega_i) d\omega_i dA(x_i) $$
   [S1, unofficial]

2. **Single Scattering Term:**
   The single scattering radiance $L_o^{(1)}$ is computed as:
   $$ L_o^{(1)}(x_o, \omega_o) = \frac{\sigma_s(x_o) F p(\omega_i \cdot \omega_o)}{\sigma_{tc}} e^{-s_{0i} \sigma_t(x_i)} e^{-s_{0o} \sigma_t(x_o)} L_i(x_i, \omega_i) $$
   Where:
   - $F = F_t(\eta, \omega_o) F_t(\eta, \omega_i)$ is the product of Fresnel transmission terms.
   - $p(\omega_i \cdot \omega_o)$ is the phase function.
   - $\sigma_{tc}$ is the total cross section (often equivalent to $\sigma_t$ in this context).
   - $s_{0i}$ is the refracted distance through the material.
   - $s_{0o}$ is the distance along the outgoing ray.
   [S1, unofficial]

3. **Refracted Distances:**
   - **Outgoing distance ($s_{0o}$):**
     $$ s_{0o} = \frac{\ln(\xi)}{\sigma_t(x_o)} $$
     where $\xi \in ]0, 1]$ is a uniformly distributed random number.
     [S1, unofficial]

   - **Incoming distance ($s_{0i}$):**
     Approximated using Snell's law, assuming the light source is far away compared to the mean free path:
     $$ s_{0i} = s_i \frac{|\omega_i \cdot n_i|}{\sqrt{1 - (\eta^2 (1 - |\omega_i \cdot n(x_i)|^2))}} $$
     Where $s_i$ is the observed distance, $\eta$ is the index of refraction, $n_i$ is the normal at the outgoing point, and $n(x_i)$ is the normal at the incoming point.
     [S1, unofficial]

4. **Dipole Diffusion Approximation (Multiple Scattering):**
   The diffuse BSSRDF $R_d$ is derived from the dipole solution to the diffusion equation. The fluence $\phi(x)$ for a dipole configuration (real source at $x_r$ and virtual source at $x_v$) is:
   $$ \phi(x) = \frac{\Phi}{4\pi D} \left( \frac{e^{-\sigma_{tr} r_r(x)}}{r_r(x)} - \frac{e^{-\sigma_{tr} r_v(x)}}{r_v(x)} \right) $$
   Where:
   - $\Phi$ is the source power.
   - $r_r(x) = ||x - x_r||$ is the distance to the real source.
   - $r_v(x) = ||x - x_v||$ is the distance to the virtual source.
   - The real source is placed at distance $z_r = 1/\sigma_{t0}$ below the surface at $x_i$.
   - The virtual source is placed symmetrically or according to boundary conditions (often $-z_r$ or adjusted for index mismatch).
   - $R_d(r) = -D \frac{\partial \phi}{\partial n} \big|_{surface} / d\Phi_i$.
   [S1, unofficial]

   For a plane-parallel medium, the diffuse reflectance $F_{dr}$ can be approximated analytically or via rational approximation:
   $$ F_{dr} = 1.440 \left( \frac{1}{\eta^2} - 1 \right) + 0.710 \left( \frac{1}{\eta^2} + 1 \right) + 0.668 + 0.0636 \eta $$
   [S1, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_bssrdf_approximation(xi, xo, wi, wo, n_i, n_o, material, light):
    """
    Computes outgoing radiance using BSSRDF approximation with Dipole and Single Scattering.
    
    Args:
        xi: Incident point on surface.
        xo: Outgoing point on surface.
        wi: Incident direction (normalized).
        wo: Outgoing direction (normalized).
        n_i: Normal at xi.
        n_o: Normal at xo.
        material: Object containing sigma_a, sigma_s, g, eta.
        light: Object containing Li (incident radiance) and si (observed distance).
        
    Returns:
        Lo: Outgoing radiance.
    """
    # Extract material properties
    sigma_a = material.sigma_a
    sigma_s = material.sigma_s
    g = material.g
    eta = material.eta
    
    # Compute derived coefficients
    sigma_t = sigma_s + sigma_a
    sigma_s0 = sigma_s * (1 - g)
    sigma_t0 = sigma_s0 + sigma_a
    D = 1.0 / (3.0 * sigma_t0)
    sigma_tr = math.sqrt(3.0 * sigma_a * sigma_t0)
    
    # 1. Single Scattering Term
    # Sample distance for single scattering
    xi_rand = random.uniform(0.0, 1.0)
    if xi_rand <= 0: xi_rand = 1e-9 # Guard against log(0)
    s0o = math.log(xi_rand) / sigma_t
    
    # Approximate refracted incoming distance
    # Note: si is observed distance, assume light is far away
    si = light.si
    dot_wi_ni = math.dot(wi, n_i)
    dot_wi_nxi = math.dot(wi, n_i) # Assuming n(xi) is n_i for local flatness or passed separately
    
    # Denominator for Snell's law approximation
    denom_sq = 1.0 - (eta**2 * (1.0 - dot_wi_nxi**2))
    if denom_sq <= 0:
        s0i = 0.0 # Degenerate case
    else:
        s0i = si * abs(dot_wi_ni) / math.sqrt(denom_sq)
        
    # Fresnel transmission (simplified approximation or lookup)
    # F = Ft(eta, wo) * Ft(eta, wi)
    # Using a simplified Fresnel transmission for demonstration
    Ft_wi = 1.0 - fresnel_reflectance(eta, math.acos(abs(math.dot(wi, n_i))))
    Ft_wo = 1.0 - fresnel_reflectance(eta, math.acos(abs(math.dot(wo, n_o))))
    F = Ft_wi * Ft_wo
    
    # Phase function p(wi . wo)
    p = phase_function(math.dot(wi, wo))
    
    # Single scattering radiance
    # L_o(1) = [sigma_s * F * p / sigma_tc] * exp(-s0i * sigma_t) * exp(-s0o * sigma_t) * Li
    # Assuming sigma_tc = sigma_t
    term_single = (sigma_s * F * p / sigma_t) * math.exp(-s0i * sigma_t) * math.exp(-s0o * sigma_t) * light.Li
    
    # 2. Multiple Scattering Term (Dipole BSSRDF)
    # Distance between xi and xo
    r = distance(xi, xo)
    
    # Dipole source positions
    # Real source at 1/sigma_t0 below xi
    z_r = 1.0 / sigma_t0
    # Virtual source position depends on boundary conditions. 
    # For simplicity, assuming symmetric dipole or standard Hanrahan-Krueger setup.
    # Standard dipole: real at z_r, virtual at -z_r (or adjusted for eta).
    # Here we use the fluence gradient method for Rd.
    
    # Calculate fluence gradient at surface for Rd
    # Rd(r) = -D * d(phi)/dn / dPhi_i
    # Approximation for Rd using the dipole solution derivative
    # This part is complex; often Rd is precomputed or approximated by the Fdr formula scaled by geometry.
    # Using the Fdr approximation for diffuse reflectance component
    Fdr = 1.440 * (1.0/eta**2 - 1.0) + 0.710 * (1.0/eta**2 + 1.0) + 0.668 + 0.0636 * eta
    
    # The BSSRDF S is often approximated as Rd(r) * F_t(wo) * F_t(wi) * p(wi, wo) / (|n.wi| + |n.wo|) for the single scattering-like term
    # Or the full BSSRDF is Rd(r) for the diffuse part.
    # Combining: The total radiance is often the sum of single scattering and the diffuse BSSRDF contribution.
    # L_o = L_o(1) + L_o(diffuse)
    
    # Diffuse contribution approximation
    # L_o(diffuse) ~ Rd(r) * Li * ... (simplified integration)
    # For a point light, the diffuse term is often:
    # L_o(diffuse) = Rd(r) * Li * (n_i . wi) * (n_o . wo) / (|n.wi| + |n.wo|) ? 
    # The source notes give: fr(1) = alpha * F * p / (|n.wi| + |n.wo|)
    # Let's use the provided single scattering BRDF approximation formula for the "single scattering term" 
    # and assume the dipole handles the rest, or that the formula provided IS the approximation for the BSSRDF term in this context.
    
    # Re-reading source [S1, unofficial]: "Single scattering BRDF approximation fr(1)... The outscattered radiance L(1)o... is computed as..."
    # It seems the method primarily focuses on the single scattering term L(1)o and the dipole for the diffuse BSSRDF Rd.
    # The final output is the sum.
    
    # Diffuse BSSRDF term (Rd)
    # Rd(r) is the diffuse reflectance. 
    # L_o(diffuse) = Rd(r) * Li * (n_i . wi) / pi ? (Standard diffuse)
    # The source does not give a explicit L_o(diffuse) formula, only Rd.
    # We will assume the single scattering term L(1)o is the primary contribution for the "Single Scattering" part of the title,
    # and the Dipole is used to compute Rd for the multiple scattering part.
    
    # Approximate Rd(r) using the dipole solution's exitance
    # Rd(r) = (1/pi) * ... (complex)
    # Using the Fdr approximation as a proxy for the diffuse component scaling
    term_diffuse = Fdr * light.Li * abs(math.dot(wi, n_i)) # Simplified
    
    Lo = term_single + term_diffuse
    
    return Lo

def fresnel_reflectance(eta, theta_i):
    """Simplified Fresnel reflectance for dielectric."""
    sin_theta_i = math.sin(theta_i)
    cos_theta_i = math.cos(theta_i)
    sin_theta_t = sin_theta_i / eta
    cos_theta_t = math.sqrt(max(0, 1 - sin_theta_t**2))
    
    Rs = ((eta * cos_theta_i - cos_theta_t) / (eta * cos_theta_i + cos_theta_t))**2
    Rp = ((eta * cos_theta_t - cos_theta_i) / (eta * cos_theta_t + cos_theta_i))**2
    return (Rs + Rp) / 2

def phase_function(mu):
    """Simple isotropic phase function."""
    return 1.0 / (4.0 * math.pi)

def distance(p1, p2):
    return math.sqrt(sum((a-b)**2 for a, b in zip(p1, p2)))
```
[G] The pseudocode includes a simplified Fresnel function and isotropic phase function as standard graphics primitives not detailed in the source notes but required for implementation. The diffuse term `term_diffuse` is a simplified placeholder for the full dipole integration, as the source notes provide the formula for $R_d$ but not the explicit integration into $L_o$ for a point light in a single line. In practice, $L_o^{diffuse}$ involves integrating $R_d$ over the surface area.

### Parameters
- $\sigma_a$: Absorption coefficient. Not given in the sources.
- $\sigma_s$: Scattering coefficient. Not given in the sources.
- $g$: Mean cosine of scattering angle. Not given in the sources.
- $\eta$: Index of refraction. Not given in the sources.
- Typical values: Not given in the sources.

### Edge Cases and Pitfalls
- **Hair Darkness:** The hair at the back of the head appears slightly darker in the BSSRDF simulation compared to full Monte Carlo. This is attributed to the forced $1/\sigma_{t0}$ distance in the diffusion approximation [S1, unofficial].
- **Sharp Edges:** Special care must be taken in the presence of highly curved surfaces. The method handles this by always evaluating the diffusion term with a minimum distance of $1/\sigma_{t0}$ to eliminate singularities at sharp edges where the source can be placed arbitrarily close to $x_o$ [S1, unofficial].
- **Boundary Conditions:** The diffusion equation does not have an analytical solution for finite media. The method assumes a semi-infinite plane-parallel medium [S1, unofficial].

## Subtopics

### [BSSRDF Model](6674-bssrdf-model.md)
The BSSRDF Model approximates subsurface light transport by formulating the problem as a surface integral, avoiding expensive volumetric path tracing in participating media under rough dielectric surfaces. It replaces volumetric integration with an analytical diffusion-based solution using a dipole configuration of real and virtual light sources. It builds on [BSSRDF Approximation with Dipole and Single Scattering](4750-bssrdf-approximation-with-dipole-and-single-scattering.md) by evaluating the transport at the surface boundary.

### [Tabulated BSSRDF](13778-tabulated-bssrdf.md)
The Tabulated BSSRDF method approximates the spatially varying reflectance distribution function (BSSRDF) for subsurface scattering by precomputing a lookup table of the single-scattering profile $S_r$ as a function of albedo $\rho$ and optical radius $r_{optical}$. It solves the problem of the high dimensionality of the full BSSRDF, which depends on five parameters ($\eta, g, \rho, \sigma_t, r$), by fixing the refractive index $\eta$ and asymmetry coefficient $g$ and reducing the remaining variables to a 2D table. This approach sets it apart from analytical approximations like the Dipole model by trading storage space for potentially higher accuracy across a wider range of material parameters, while avoiding the computational cost of evaluating complex analytical functions per sample.

## Operating characteristics
- **Time Complexity:** Significantly faster than full volumetric Monte Carlo. Rendering time for a milk glass example was 4 minutes for BSSRDF vs. 2 minutes for diffuse BRDF, and much faster than the 1250 minutes for full Monte Carlo [S1, unofficial].
- **Memory Complexity:** Low, as it does not require storing volumetric photon maps or extensive precomputed tables (unless using Tabulated BSSRDF).
- **Strengths:** Captures the true appearance of translucent materials (marble, skin, milk, etc.) with much lower computational cost than volumetric path tracing. Handles spatial variation of light entry and exit.
- **Limitations:** Approximation errors near sharp edges or highly curved surfaces (mitigated by minimum distance clamp). "Hair darkness" artifact due to diffusion approximation. Assumes semi-infinite medium.
- **When to use:** When rendering translucent materials where performance is critical and full volumetric scattering is too expensive.
- **When not to use:** When high accuracy near boundaries or complex geometry is required without the minimum distance hack, or when the medium is not highly scattering (diffusion approximation fails).

## Background
The model was presented in a paper for the SIGGRAPH conference proceedings, funded by NSF (IIS-0085864) and DARPA (DABT63-95-C-0085) [S1, unofficial]. It builds on the radiative transfer equation and the diffusion approximation, extending the work of Hanrahan and Krueger [S1, unofficial]. The method is used for rendering translucent materials such as marble, cloth, paper, skin, milk, cheese, bread, meat, fruits, plants, fish, ocean water, and snow [S1, unofficial].

## Sources
[S1] Practical Model for Subsurface Light Transport, https://www.scribd.com/document/904578145/A-Practical-Model-for-Subsurface-Light-Transport, fetched 2026-10-29 (unofficial)
