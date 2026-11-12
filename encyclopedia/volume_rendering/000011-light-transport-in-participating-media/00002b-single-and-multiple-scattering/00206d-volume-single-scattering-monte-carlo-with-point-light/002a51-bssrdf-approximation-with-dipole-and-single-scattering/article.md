# BSSRDF Approximation with Dipole and Single Scattering

Also called: BSSRDF Model with Dipole Diffusion Approximation, BSSRDF Rendering with Dipole Diffusion and Single Scattering, BSSRDF approximation, Dipole BSSRDF, First-order scattering, Hanrahan and Krueger extension, Practical Model for Subsurface Light Transport, Single Scattering Term

## Overview
The BSSRDF approximation with dipole diffusion and single scattering computes subsurface light transport by evaluating the Bidirectional Surface Scattering Reflectance Distribution Function (BSSRDF) on the surface boundary, combining a dipole diffusion solution for multiple scattering with a first-order scattering term [S1, unofficial]. It replaces expensive volumetric path tracing with an analytical diffusion-based solution using real and virtual point sources, and explicitly accounts for single scattering along refracted ray segments [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Surface point $x_o$, outgoing direction $\omega_o$, incident point $x_i$, incident direction $\omega_i$, material coefficients $\sigma_a, \sigma_s, g, \eta$, incident radiance $L_i(x_i, \omega_i)$, geometric normals $n(x_o), n(x_i)$, observed distance $s_i$, and random number $\xi \in (0, 1]$ [S1, unofficial].
*   **Outputs:** Outgoing radiance $L_o(x_o, \omega_o)$ [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Parameters:**
*   $\sigma_a$: Absorption coefficient [S1, unofficial].
*   $\sigma_s$: Scattering coefficient [S1, unofficial].
*   $g$: Mean cosine of the scattering angle [S1, unofficial].
*   $\eta$: Relative index of refraction [S1, unofficial].
*   $\sigma_{t0}$: Reduced extinction coefficient, $\sigma_{t0} = \sigma_{s0} + \sigma_a$, where $\sigma_{s0} = \sigma_s (1 - g)$ [S1, unofficial].
*   $D$: Diffusion constant, $D = \frac{1}{3\sigma_{t0}}$ [S1, unofficial].
*   $\sigma_{tr}$: Effective transport coefficient, $\sigma_{tr} = \sqrt{3\sigma_a \sigma_{t0}}$ [S1, unofficial].

**Steps:**
1.  **Compute Reduced Coefficients:** Calculate $\sigma_{s0} = \sigma_s (1 - g)$, $\sigma_{t0} = \sigma_{s0} + \sigma_a$, $D = \frac{1}{3\sigma_{t0}}$, and $\sigma_{tr} = \sqrt{3\sigma_a \sigma_{t0}}$ [S1, unofficial].
2.  **Determine Source Positions (Dipole Approximation):** Place a real light source at distance $z_r = 1/\sigma_{t0}$ below $x_i$ along the inward normal, and place a virtual source according to boundary conditions [S1, unofficial].
3.  **Compute Fluence:** Evaluate fluence $\phi(x) = \frac{\Phi e^{-\sigma_{tr} r(x)}}{4\pi D r(x)}$ for a point source of power $\Phi$, where $r(x)$ is the distance to the source [S1, unofficial]. For a dipole, superimpose the real and virtual source contributions [S1, unofficial].
4.  **Calculate Single Scattering Term:** Compute the first-order scattering radiance $L_o^{(1)}(x_o, \omega_o) = \frac{\sigma_s(x_o) F p(\omega_i \cdot \omega_o)}{\sigma_{tc}} e^{-s_{0i} \sigma_t(x_i)} e^{-s_{0o} \sigma_t(x_o)} L_i(x_i, \omega_i)$ [S1, unofficial].
5.  **Compute Refracted Distances:** Sample outgoing distance $s_{0o} = \frac{\ln(\xi)}{\sigma_t(x_o)}$ along the refracted outgoing ray [S1, unofficial]. Approximate incoming refracted distance $s_{0i} = s_i \frac{|\omega_i \cdot n(x_o)|}{\sqrt{1 - \eta^2 (1 - |\omega_i \cdot n(x_i)|^2)}}$, where $n(x_o)$ is the normal at the outgoing point and $n(x_i)$ is the normal at the incoming point [S1, unofficial].
6.  **Combine Terms:** Integrate incident radiance over incoming directions and area using the BSSRDF $S$ [S1, unofficial]. Compute the diffuse BSSRDF $R_d(r) = -D \frac{(\hat{n} \cdot \nabla \phi)(x_s)}{d\Phi_i(x_i)}$ where $r = ||x_s - x_i||$ [S1, unofficial]. Apply the rational approximation for diffuse Fresnel reflectance $F_{dr} = \frac{1.440}{\eta^2} - \frac{0.710}{\eta} + 0.668 + 0.0636\eta$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def fresnel_transmission(eta, omega, n):
    """Standard Fresnel transmission coefficient Ft = 1 - Fr."""
    cos_theta = max(0.0, dot(omega, n))
    r0 = ((1.0 - eta) / (1.0 + eta)) ** 2
    fr = r0 + (1.0 - r0) * (1.0 - cos_theta) ** 5
    return 1.0 - fr

def phase_function(omega_i, omega_o):
    """Normalized phase function p(omega_i . omega_o)."""
    return 1.0 / (4.0 * math.pi)

def dipole_fluence(x_i, x_o, sigma_tr, D, z_r):
    """Fluence from dipole configuration (real and virtual sources)."""
    dx = x_o[0] - x_i[0]
    dy = x_o[1] - x_i[1]
    dz = x_o[2] - x_i[2]
    r_dr = math.sqrt(dx**2 + dy**2 + (dz + z_r)**2)
    r_dv = math.sqrt(dx**2 + dy**2 + (dz - z_r)**2)
    phi_real = math.exp(-sigma_tr * r_dr) / (4.0 * math.pi * D * r_dr)
    phi_virtual = math.exp(-sigma_tr * r_dv) / (4.0 * math.pi * D * r_dv)
    return phi_real - phi_virtual

def compute_bssrdf_dipole_single_scattering(
    x_o, omega_o, x_i, omega_i,
    sigma_a, sigma_s, g, eta,
    L_i, sigma_t,
    n_o, n_i, s_i_observed
):
    # 1. Compute Reduced Coefficients
    sigma_s0 = sigma_s * (1.0 - g)
    sigma_t0 = sigma_s0 + sigma_a
    D = 1.0 / (3.0 * sigma_t0)
    sigma_tr = math.sqrt(3.0 * sigma_a * sigma_t0)

    # 2. Single Scattering Term
    F_t_o = fresnel_transmission(eta, omega_o, n_o)
    F_t_i = fresnel_transmission(eta, omega_i, n_i)
    F = F_t_o * F_t_i
    p_val = phase_function(omega_i, omega_o)

    # Outgoing distance s_0o
    xi = random.uniform(0.0001, 1.0)
    s_0o = math.log(xi) / sigma_t(x_o)

    # Incoming distance s_0i
    # ni in formula is normal at outgoing point (n_o), n(xi) is normal at incoming point (n_i)
    dot_n_out = abs(dot(omega_i, n_o))
    dot_n_in = abs(dot(omega_i, n_i))
    denom_sq = 1.0 - (eta**2 * (1.0 - dot_n_in**2))
    if denom_sq <= 0.0:
        s_0i = s_i_observed
    else:
        s_0i = s_i_observed * dot_n_out / math.sqrt(denom_sq)

    att_in = math.exp(-s_0i * sigma_t(x_i))
    att_out = math.exp(-s_0o * sigma_t(x_o))
    sigma_tc = sigma_t(x_o) # Total cross section, often sigma_t
    single_scattering_term = (sigma_s(x_o) * F * p_val / sigma_tc) * att_in * att_out * L_i

    # 3. Diffuse BSSRDF Term (Dipole)
    z_r = 1.0 / sigma_t0
    phi = dipole_fluence(x_i, x_o, sigma_tr, D, z_r)

    # Radiant exitance approximation M ≈ phi * F_dr
    F_dr = 1.440 / (eta**2) - 0.710 / eta + 0.668 + 0.0636 * eta
    diffuse_term = phi * F_dr

    return single_scattering_term + diffuse_term
```

## Subtopics

### [BSSRDF Model](6674-bssrdf-model.md)
The BSSRDF Model approximates subsurface light transport by evaluating the transport at the surface boundary, replacing volumetric integration with an analytical diffusion-based solution using a dipole configuration of real and virtual light sources. It is distinguished from pure volume rendering methods by its surface-based evaluation and from the full BSSRDF Approximation by its specific reliance on the dipole source configuration for low-absorption regimes.

## Operating characteristics
*   **Time Complexity:** Rendering times for milk glass were 4 minutes for BSSRDF versus 2 minutes for diffuse BRDF [S1, unofficial].
*   **Memory Complexity:** Low, as it relies on analytical formulas rather than storing volumetric photon maps or extensive precomputed textures [S1, unofficial].
*   **Strengths:** Efficient approximation of subsurface scattering for translucent materials; handles complex lighting via integration over surface area [S1, unofficial].
*   **Limitations:** Hair appears darker in simulations due to the forced $1/\sigma_{t0}$ distance in the diffusion approximation [S1, unofficial]. Accuracy degrades near sharp edges where the source placement approaches $x_o$ [S1, unofficial].
*   **When to Use:** Rendering translucent materials like skin, marble, milk, and cheese where multiple scattering is significant but full volume integration is too expensive [S1, unofficial].
*   **When Not to Use:** Highly absorbing media, very thin objects, or when high precision near boundaries is required [S1, unofficial].

## Background
The model was presented in a paper for the SIGGRAPH conference proceedings, funded by NSF (IIS-0085864) and DARPA (DABT63-95-C-0085) [S1, unofficial]. It builds on the radiative transport equation and the diffusion approximation, which assumes that light distribution in highly scattering media tends to become isotropic [S1, unofficial]. The diffusion equation is derived from a two-term expansion of radiance involving fluence and vector irradiance [S1, unofficial]. The dipole solution for a semi-infinite medium allows for efficient computation of the BSSRDF [S1, unofficial].

## Sources
[S1] Practical Model for Subsurface Light Transport, https://www.scribd.com/document/904578145/A-Practical-Model-for-Subsurface-Light-Transport, fetched 2026-10-29 (unofficial)
