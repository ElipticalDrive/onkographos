# Single Scattering Approximations

Also called: Single-scattering volume rendering, Single-scattering atmosphere rendering

## Overview
Single scattering approximations solve the Radiative Transfer Equation (RTE) by assuming that light undergoes at most one scattering event within the participating medium before reaching the camera [S2, S5, unofficial]. This simplification ignores multiple bounces (glow) and secondary in-scattering, reducing the computational cost significantly compared to full path tracing [S5, unofficial]. It is distinguished from multiple scattering methods by its inability to model light trapped within dense media, and from heterogeneous methods by often assuming constant coefficients (though atmospheric variants handle density falloff) [S2, S5, unofficial].

## Shared principles
All methods in this family rely on the conservation of energy for a differential cylinder of light passing through a medium [S1, unofficial]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [S1, unofficial]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [S1, unofficial].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [S1, unofficial].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray.
    *   *Out-scattering:* Light from the view ray reflected away.
In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [S1, unofficial].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [S1, unofficial].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [S1, unofficial]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [S1, unofficial].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [S1, unofficial].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [S1, unofficial].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [S1, unofficial]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [S1, unofficial].

**5. Phase Functions**
Scattering directionality is modeled by a phase function $p(\omega_i, \omega_o)$, defined in terms of $a = \cos(\theta)$, where $\theta$ is the angle between the incident direction $\omega_i$ and the scattered direction $\omega_o$ [S1, unofficial]. The choice of phase function depends on the ratio of particle radius $r$ to wavelength $\lambda$ [S1, unofficial]:
*   **Rayleigh Scattering:** Used when $r \ll \lambda$ (e.g., smoke, dust, water molecules).
    $$ P_R(a) = \frac{3}{4}(1+a)^2 $$
*   **Mie Scattering:** Used when $r \approx \lambda$ (e.g., fog, water droplets). Common variants include:
    *   Haziness: $P_{HM}(a) = 1 + 9 \left( \frac{1+a}{2} \right)^8$
    *   Murkiness: $P_{MM}(a) = 1 + 50 \left( \frac{1+a}{2} \right)^{32}$
*   **Henyey-Greenstein (HG):** An approximation to Mie scattering, widely used in graphics for its efficiency and asymmetry parameter $g$:
    $$ P_{HG}(a) = \frac{1 - g^2}{(1 + g^2 - 2ga)^{\frac{3}{2}}} $$
*   **Schlick's Approximation:** A computationally cheaper approximation to HG that avoids fractional exponentiation:
    $$ P_S(a) = \frac{1-g^2}{(1-ga)^2} $$

## Subtopics

### [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md)
This method renders participating media by solving the radiative transfer equation under the assumption that light scatters exactly once within a single, homogeneous volume before reaching the camera. It extends an absorption-only model by adding in-scattering from surface-emitting geometry, assuming monochromatic coefficients and surfaces that do not reflect or transmit light. It is distinguished from multiple scattering methods by ignoring secondary bounces within the medium and from heterogeneous methods by assuming constant extinction and scattering coefficients throughout the volume.

### [Volume Single Scattering (Monte Carlo with Point Light)](2293-volume-single-scattering-monte-carlo-with-point-light.md)
Computes the radiance contribution of a single scattering event in a homogeneous participating medium by integrating in-scattered light from a point light source along a view ray. It differs from uniform ray marching variants by using Monte Carlo distance sampling to select scattering positions, which reduces bias from fixed step discretization. It is a specific instance of single scattering approximations [5340-single-scattering-approximations.md] tailored for point lights and homogeneous media.

### [Single-Scattering Atmosphere Rendering](3289-single-scattering-atmosphere-rendering.md)
Single-scattering atmosphere rendering computes the color of the sky by numerically integrating the in-scattered light from a single distant light source (the sun) along a view ray through a spherical atmosphere shell. It solves the Radiative Transfer Equation (RTE) under the assumption that light is scattered only once before reaching the camera, accounting for Rayleigh (gas) and Mie (aerosol) scattering, as well as ozone absorption. Unlike full multiple-scattering solvers, this method is computationally efficient and suitable for real-time rendering, though it ignores light scattered between atmospheric particles. It sets itself apart from homogeneous volume methods by modeling the exponential density falloff of a planetary atmosphere and from Monte Carlo methods by using deterministic numerical integration.

### Single Scattering
#### definition
- [S1, unofficial] An approximation where in-scattered radiance is assumed to come from lights directly, similar to direct lighting, requiring integration over the spherical domain.
#### step
- [S1, unofficial] Substitute spherical integration using Next Event Estimation (directly enumerating all lights).
#### formula
- [S1, unofficial] $$ L_s(x_t, \vec w) = \int_{S^2} phase(x_t, \vec w', \vec w) L_i(x_t, \vec w') d\vec w' $$ where $L_s$ is in-scattered radiance, $phase$ is the phase function, $L_i$ is incident radiance from lights (or 0).

#### Multiple Forward/Backward Scattering Approximation
#### step
- [S2, book] Kniss et al. [30] generalized single-pass shadow algorithms to multiple forward scattering by gathering accumulated light from several sampled directions in the previous sampling plane.
- [S2, book] Zhang et al. [62] added backward scattering from the next couple of sampling planes.

## Background
Single scattering approximations originate from the fundamental solution to the Radiative Transfer Equation (RTE) in participating media [S1, unofficial]. The method is widely used in real-time rendering for atmospheric effects (sky, fog) and in offline rendering for quick previews or low-density media where multiple bounces are negligible [S5, unofficial]. The homogeneous volume variant is often taught as the first step in volumetric path tracing, providing a baseline for understanding free-path sampling and in-scattering estimation [S2, book]. Atmospheric single scattering is a staple in computer graphics for rendering realistic skies, leveraging the specific physical properties of Earth's atmosphere (Rayleigh and Mie scattering) [S5, unofficial].

## Sources
[S1] l0exy1h/VolumetricPathTracing-public, https://github.com/l0exy1h/VolumetricPathTracing-public, fetched 2026-10-26 (unofficial)
[S2] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
