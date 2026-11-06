# Volumetric Radiance Estimators

Also called: B-B1D, B-B2D, B-B3D, B-P2D, B-P3D, P-B2D, P-B3D, P-P3D

## Overview
Volumetric Radiance Estimators are a family of nine distinct methods introduced by Jarosz et al. [2011a] that estimate in-scattered radiance by aggregating energy from discrete photon samples (points or beams) using spatial blurring [S1]. They solve the problem of reconstructing continuous radiance fields from discrete light transport samples, differing from unbiased path tracing siblings like [Volumetric Path Tracing](1863-volumetric-path-tracing.md) by relying on biased density estimation rather than direct Monte Carlo integration of the path contribution function [S1].

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

The family consists of nine estimators distinguished by the representation of equilibrium radiance (Photon Points or Photon Beams), the query type (Point or Beam), and the dimension of the blur kernel (1D, 2D, or 3D) [S1]. The naming convention follows the order: Data Source × Query Type × Kernel Dimension (e.g., B-P3D refers to Beam data, Point query, 3D kernel) [S1]. This opposes the convention used by Jarosz et al. [2011a] [S1].

### [Point Query × Point Data, 3D Blur](2235-point-query-point-data-3d-blur.md)
Estimates radiance at a specific 3D query point by aggregating nearby photon points using a 3D spherical kernel. It is distinguished by using point-based data and 3D blurring, whereas beam variants use extended data and lower-dimensional blurring.

### [Extended Balance Heuristic](5601-extended-balance-heuristic.md)
A weighting method for mixing unbiased path sampling and extended space volumetric estimators to reduce variance, derived from the balance heuristic.

### [Beam Query × Beam Data, 2D Blur](4555-beam-query-beam-data-2d-blur.md)
Uses extended beam data and 2D blurring to reduce bias along the ray direction compared to point-based methods.

### Short Beam Estimator via Russian Roulette
A technique to convert a long beam estimator into a short beam estimator by treating the beam length sampling as a Russian Roulette (RR) decision on the contribution [S1]. Long beams extend until the closest surface, with transmittance appearing in the estimator, whereas short beams are of finite extent where transmittance vanishes (replaced by a constant 1) via a single step function approximation, coined 'progressive deep shadow maps' [S1]. To implement this, one makes a zero contribution whenever the beam length $l$ is less than the distance to the query point $t$, and otherwise divides the original long beam estimator result by the RR probability $\Pr\{l > t\}$ [S1]. The probability that a short beam of length $l$ contributes at distance $t_0$ is $\Pr\{l > t_0\} = \int_{t_0}^{\infty} p(t') dt'$, where $p(t)$ is the pdf used to sample the beam length [S1]. The short beam estimator is calculated as $\langle I \rangle_{\text{short}} = \langle I \rangle_{\text{long}} \frac{H(l - t)}{\Pr\{l > t\}}$, where $H$ is the Heaviside step function [S1]. This approach provides an intuitive explanation and generalization of Jarosz et al.'s [2011b] unbiased progressive deep shadow maps [S1].

## Background
The family of nine volumetric radiance estimators was introduced by Jarosz et al. [2011a] to address the reconstruction of continuous radiance from discrete photon samples in participating media [S1]. They were generalized for light and eye subpaths of arbitrary length in subsequent work [S1]. The concept of "progressive deep shadow maps" using short beams was introduced by Jarosz et al. [2011b] [S1]. These estimators are evaluated on segments along light and eye subpaths and can be fused with unbiased path sampling techniques in algorithms such as [Unified Points, Beams, and Paths (UPBP)](1402-unified-points-beams-and-paths-upbp.md) [S1].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
