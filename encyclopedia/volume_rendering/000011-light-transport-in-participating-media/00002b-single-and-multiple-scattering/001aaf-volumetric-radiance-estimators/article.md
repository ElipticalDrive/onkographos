# Volumetric Radiance Estimators

Also called: B-B1D, B-B2D, B-B3D, B-P2D, B-P3D, P-B2D, P-B3D, P-P3D

## Overview
Volumetric Radiance Estimators are a family of nine distinct methods introduced by Jarosz et al. [2011a] that estimate equilibrium radiance in participating media by reconstructing values from discrete samples (photon points or photon beams) using spatial blurring [S1]. They differ from unbiased path integral estimators by lacking a general path contribution function and pdf, instead relying on specific radiometric quantities and kernel dimensions to approximate the radiance field [S1].

## Shared principles
All methods in this family are derived from the conservation of energy for a differential cylinder of light passing through a medium [G]. They share the following core definitions and mechanisms:

**1. Fundamental Interactions**
Any volumetric effect is composed of three fundamental characteristics [G]:
*   **Absorption:** The loss of radiant energy along a ray due to energy conversion (e.g., to heat). It is described by the absorption coefficient $\sigma_a$ (units: $m^{-1}$) [G].
*   **Emission:** The addition of radiant energy where the medium is luminous (e.g., flames). It is described by the emissive term $L_e(\mathbf{p}, \boldsymbol{\omega})$, representing radiance emitted per unit length [G].
*   **Scattering:** The redirection of light rays upon collision with particles. It is split into:
    *   *In-scattering:* Light from other directions reflected into the view ray [G].
    *   *Out-scattering:* Light from the view ray reflected away [G].
In most lighting calculations, out-scattering is folded into the **extinction** term because its net effect on the view ray is identical to absorption [G].

**2. Notation**
*   $\mathbf{p}$: Position of interaction.
*   $\boldsymbol{\omega}$: Ray direction (unit vector).
*   $L$: Radiance quantity.
*   $L_i$: Incoming radiance.
*   $L_o$: Outgoing radiance.
*   $\sigma_t$: Total extinction coefficient, $\sigma_t = \sigma_a + \sigma_s$ [G].

**3. Differential Forms**
The change in radiance $dL$ over a differential path length $dt$ is governed by the balance of loss and gain terms [G]:
*   **Absorption Model:** $dL = -\sigma_a L_i dt$ [G].
*   **Emission Model:** $dL = \sigma_e dt$ (where $\sigma_e$ is the emission coefficient) [G].
*   **General RTE:** Combines extinction (loss) and source terms (gain from emission and in-scattering) [G].

**4. Transmittance**
The fraction of light surviving transmission through a medium is defined by the Beer-Lambert law [S3]:
$$ T = e^{-\tau} $$
where $\tau$ is the optical depth (integrated extinction) [S3].

## Subtopics

The naming convention for estimators (e.g., B-P3D) indicates radiance data (Photon Points or Photon Beams), query type (Point or Beam), and kernel dimension (1D, 2D, 3D) in that order [S1]. This opposes the convention used by Jarosz et al. [2011a] [S1].

### [Point Query × Point Data, 3D Blur](2235-point-query-point-data-3d-blur.md)
Estimates in-scattered radiance at a specific query point by averaging photon energy over a 3D spherical kernel. It uses a fixed-density spatial average rather than a balance heuristic.

### [Extended Balance Heuristic](5601-extended-balance-heuristic.md)
A variance reduction technique that combines estimates from a mixture of sampling techniques by incorporating kernel support size into the weighting function.

### Short Beam Estimator via Russian Roulette
A technique to convert a long beam estimator into a short beam estimator by treating the beam length sampling as a Russian Roulette (RR) decision on the contribution [S1]. A zero contribution is made whenever the beam length $l$ is less than the distance to the query point $t$; otherwise, the original long beam estimator result is divided by the RR probability $\Pr\{l > t\}$ [S1]. The probability that a short beam of length $l$ contributes at distance $t_0$ is $\Pr\{l > t_0\} = \int_{t_0}^{\infty} p(t') dt'$, where $p(t)$ is the pdf used to sample the beam length [S1]. The short beam estimator is calculated as $\langle I \rangle_{\text{short}} = \langle I \rangle_{\text{long}} \frac{H(l - t)}{\Pr\{l > t\}}$, where $H$ is the Heaviside step function [S1]. This approach provides an intuitive explanation and generalization of Jarosz et al.'s [2011b] unbiased progressive deep shadow maps [S1].

## Background
The family of nine distinct volumetric radiance estimators was introduced by Jarosz et al. [2011a]. They differ in the representation of equilibrium radiance, the radiometric quantities estimated, and the dimension of blur used to reconstruct the desired quantity from samples [S1]. The estimators are evaluated on segments along light and eye subpaths [S1]. Jarosz et al. [2011b] later proposed an unbiased approximation of transmittance using step functions, coined "progressive deep shadow maps," which corresponds to the short beam variants [S1].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
