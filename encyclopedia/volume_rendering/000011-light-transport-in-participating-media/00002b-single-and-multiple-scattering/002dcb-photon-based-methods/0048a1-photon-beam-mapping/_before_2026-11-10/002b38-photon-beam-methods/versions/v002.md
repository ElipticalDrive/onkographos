# Photon Beam Methods


## Overview
Photon Beam Methods are a family of density estimation techniques for light transport in participating media that replace discrete photon points with continuous, directional beam segments [S1, unofficial]. Unlike standard photon tracing which deposits energy at discrete interaction points, these methods model light as extended cylinders, allowing for analytical integration of radiance over beam-ray intersections [S1, unofficial]. This approach reduces variance compared to point-based density estimation and eliminates the quantization errors associated with grid-based diffusion methods [S1, unofficial].

## Shared principles
All methods in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [S1, unofficial]. They share the following core definitions and mechanisms:

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

### [Transient Photon Beams](3296-transient-photon-beams.md)
Transient Photon Beams is a numerical method for rendering participating media in the transient state, extending the steady-state Photon Beams algorithm by accounting for photon time-of-flight along continuous beam trajectories. It solves the problem of computing time-resolved radiance in complex scenes by combining the efficiency of density estimation with the temporal accuracy of continuous path parameterization. Unlike steady-state variants, it explicitly tracks the optical path length to determine arrival times, and unlike pure Monte Carlo transient methods, it avoids high variance by evenly distributing integrated radiance over temporal intervals derived from ray-beam intersections.

### [Photon Beam Diffusion](3768-photon-beam-diffusion.md)
Photon Beam Diffusion (PBD) is a hybrid Monte Carlo method for rendering subsurface scattering in translucent materials that interprets incident light as a continuous beam of photons and solves the diffusion equation via numerical integration. It eliminates the need for quantization and sum-of-Gaussians approximations used in Quantized Diffusion (QD) by treating the beam as an extended source, providing increased speed, flexibility, and numerical stability. Unlike QD, which requires 20–60 Gaussians for equivalent quality, PBD achieves similar results with only 3–5 Monte Carlo samples.

### [Photon Marching](4775-photon-marching.md)
Photon Marching is a method for generating "discrete photon beams" by depositing photons at uniform intervals along a ray originating from a light source, rather than propagating a single photon via a Markov random walk. It differs from standard photon tracing by distributing the power of a single photon among multiple deposited photons at fixed steps, analogous to how ray marching solves 1D integrals via Riemann sums compared to random-walk path tracing. It serves as the foundational geometric definition for photon beams in the Photon Beam Methods family, though in practice, beams are often still generated via random-walk sampling.

### Diffuse Single Scattering Integration
*   **Definition**: An explicit calculation of the diffuse single-scattering term using the same Monte Carlo integration strategy as multi-scattering, replacing the multi-scattering Green's function with a single-scattering response [S1, unofficial].
*   **Step**: Compute the single-scattering profile $R^{(1)}(\vec{x}, \vec{\omega})$ using the same Monte Carlo estimator (8), but with $f(\vec{x}, \vec{\omega}, t_i) = r^{(1)}(\vec{x}, \vec{x}_r(t_i)) Q(t_i)$ [S1, unofficial].
*   **Formula**: $r^{(1)}(\vec{x}, \vec{x}_r(t)) = \frac{f_s(\vec{\omega} - \vec{x}_r(t)) e^{-t_0 d_r(t)} F_t(\theta_o, \eta) F_t(\theta_i, 1/\eta) \cos \theta_o}{d_r^2(t)}$, where $\vec{x}_r(t)$ and $d_r(t)$ are the direction and distance from $\vec{x}_r(t)$ to $\vec{x}$, $\theta_o = \arccos(\vec{x}_r(t) - \vec{x} \cdot \vec{n})$, $\theta_i = \arccos(\vec{\omega} \cdot \vec{n})$, and $F_t$ and $f_s$ are transmissive Fresnel and phase function terms [S1, unofficial].
*   **Pitfall**: If the surface separates volumes with different indices of refraction, use a separate sample sequence for single-scattering starting from the critical angle, as contributions below the critical angle are zero. This guarantees a smooth profile regardless of sample count [S1, unofficial].

### Unbiased Photon Gathering
*   **Definition**: An unbiased photon gathering method by Qin et al. [13] that combines each photon's path with the camera path into one path [S2].
*   **Step**: Consider all factors in Monte Carlo methods such as visibility and probability [S2].

## Background
Photon Beam Methods originated from the need to improve the efficiency and accuracy of density estimation in participating media. Standard photon tracing often results in dim contributions due to the sparse nature of photon deposits, which are frequently omitted in production implementations [S1, unofficial]. The introduction of continuous beam representations allowed for the explicit calculation of diffuse single-scattering terms, which were previously prohibitively expensive to compute via brute-force Monte Carlo particle tracing [S1, unofficial]. The method was further developed to handle subsurface scattering in translucent materials, offering a stable alternative to quantized diffusion methods [S1, unofficial]. Unbiased variants were later introduced to address bias issues inherent in density estimation by combining photon and camera paths [S2].

## Sources
[S1] Photon Beam Diffusion for Rendering, https://www.scribd.com/document/255860491/Photon-Beam-Diffusion, fetched 2026-10-29 (unofficial)
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
