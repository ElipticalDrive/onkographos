# Photon-based Methods

Also called: Photon Mapping for Participating Media, Photon Beam Tracing

## Overview
Photon-based Methods solve the Radiative Transfer Equation (RTE) by decoupling light transport simulation from the final image formation, estimating radiance at sample points using stored photon data rather than tracing paths to the camera [S1, 2521-photon-based-estimation.md]. Unlike direct path tracing which accumulates radiance along camera rays, these methods first trace photons through the participating medium to build a spatial representation of light energy, then estimate the in-scattered radiance at rendering time by querying this representation [S1, 2521-photon-based-estimation.md]. This family is distinguished from [Photon Beam Methods](4976-photon-beam-methods.md) by its use of point-based or volume-based density estimation rather than directional beam structures, and from [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md) and [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by the specific storage and querying mechanisms employed (e.g., BVHs, 3D textures, or point clouds) to handle the spatial distribution of photons.

## Shared principles
All methods in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [S1]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [S1]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [S1].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [S1].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray.
    *   *Out-scattering:* Light from the view ray reflected away.
In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [S1].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [S1].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [S1]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [S1].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [S1].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [S1].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [S1]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [S1].

**5. Phase Functions**
Scattering directionality is modeled by a phase function $p(\omega_i, \omega_o)$, defined in terms of $a = \cos(\theta)$, where $\theta$ is the angle between the incident direction $\omega_i$ and the scattered direction $\omega_o$ [S1]. The choice of phase function depends on the ratio of particle radius $r$ to wavelength $\lambda$ [S1]:
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

### [Photon-based Estimation](2521-photon-based-estimation.md)
Photon-based Estimation methods solve the RTE by decoupling light transport simulation from the final image formation, estimating radiance at sample points using stored photon data rather than tracing paths to the camera. Unlike direct path tracing which accumulates radiance along camera rays, these methods first trace photons through the participating medium to build a spatial representation of light energy, then estimate the in-scattered radiance at rendering time by querying this representation. This family is distinguished from [Photon Beam Methods](4976-photon-beam-methods.md) by its use of point-based or volume-based density estimation rather than directional beam structures, and from [Compute Shader Photon Integration](2111-compute-shader-photon-integration.md) and [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by the specific storage and querying mechanisms employed (e.g., BVHs, 3D textures, or point clouds) to handle the spatial distribution of photons.

### [Photon Beam Methods](4976-photon-beam-methods.md)
Photon Beam Methods are a family of numerical techniques for rendering participating media and translucent materials that treat incident light as a continuous beam of photons rather than discrete points or rays. Unlike [Photon-based Estimation](2521-photon-based-estimation.md), which typically deposits photons as point sources for density estimation, Photon Beam Methods maintain the directional continuity of light transport, allowing for more efficient integration of scattering profiles along beam trajectories. This approach mitigates variance in complex scenes by combining beam tracing with density estimation or diffusion approximations, avoiding the high computational overhead of standard Monte Carlo sampling for every ray-beam intersection.

### Multiple Scattering with Photon Marching
Photon marching combines single scattering photon beam tracing with random-walk sampling to handle multiple scattering [S1].

#### step
1. For a discrete beam $b_0$, choose a random propagation distance $t_s$ for the scattering event location [S1].
2. In homogeneous media, compute $t_s = -\log(1-\xi)/\sigma_t$ [S1].
3. In heterogeneous media, use inversion method or delta-tracking [S1].
4. Initiate a new discrete beam $b_1$ with origin $x_{b1} = x_{b0} + t_s \tilde{\omega}_{b0}$, direction $\tilde{\omega}_{b1}$ sampled from $f(\theta)$, and starting power $\Phi_{b1} = (\sigma_s/\sigma_t) \Phi_{b0}$ [S1].
5. Continue photon marching of the parent beam $b_0$ to the media's boundary [S1].

#### formula
$$ t_s = -\log(1-\xi)/\sigma_t $$
with pdf$(t_s) = \sigma_t e^{-\sigma_t t_s}$, used in homogeneous media to determine scattering distance [S1].

#### parameter
$\sigma_s/\sigma_t$: The scattering albedo, used for Russian roulette. If accepted, the starting power simplifies to $\Phi_{b1} = \Phi_{b0}$ in grey media [S1].

## Background
Photon-based methods for participating media were introduced by Jensen et al. in 2001, extending the photon mapping technique from surface rendering to volumetric effects [S1]. The method was developed to address the high variance and computational cost of direct path tracing in highly scattering media, where many bounces are required to capture indirect illumination. The approach was later refined to include photon beams, which provide a more continuous representation of light transport, improving the accuracy of in-scattered radiance estimation [S1]. These methods are widely used in offline rendering and high-quality visual effects where accurate simulation of light transport in fog, smoke, and translucent materials is critical.

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
