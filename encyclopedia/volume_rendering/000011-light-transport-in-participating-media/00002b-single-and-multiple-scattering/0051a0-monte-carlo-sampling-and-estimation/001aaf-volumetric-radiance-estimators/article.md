# Volumetric Radiance Estimators

Also called: B-B1D, B-B2D, B-B3D, B-P2D, B-P3D, P-B2D, P-B3D, P-P3D

## Overview
Volumetric Radiance Estimators are a family of nine distinct methods introduced by Jarosz et al. [2011a] that estimate equilibrium radiance in participating media by reconstructing continuous fields from discrete samples (photons or beams) using spatial blurring kernels [S1]. They differ from unbiased path integral estimators by not being formulated as general path sampling techniques, and they differ from sibling methods like [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md) by relying on pre-computed photon data structures rather than on-the-fly ray marching [S1].

## Shared principles
Builds on [Single and multiple scattering](35-single-and-multiple-scattering.md) [G].

## Subtopics

The family consists of nine estimators defined by three dimensions: the data representation (Photon Points or Photon Beams), the query type (Point or Beam), and the kernel dimension (1D, 2D, or 3D) [S1]. The naming convention follows the order: Data × Query × Kernel Dimension (e.g., B-P3D is Beam Data, Point Query, 3D Kernel) [S1]. This opposes the convention used by Jarosz et al. [2011a] [S1]. Kd denotes a normalized d-dimensional kernel used for blurring, where 1D uses a line, 2D uses a square, and 3D uses a cube [S1]. The scattering function ρ at a query location xj is evaluated with the direction of the photon beam or photon, which may not pass through this location [S1].

### [Point Query × Point Data, 3D Blur](2235-point-query-point-data-3d-blur.md)
Estimates the in-scattered radiance at a specific query point by aggregating the energy of discrete photon points stored within a surrounding three-dimensional volume. It solves the problem of approximating the integral of incoming radiance from all directions by converting a continuous density estimation problem into a discrete summation over a spherical kernel. It differs from its sibling, the Extended Balance Heuristic, by relying on a fixed spherical kernel and standard density estimation rather than balance heuristics to combine multiple estimators. It differs from its child, Beam Query × Beam Data, 2D Blur, by using discrete point data and blurring in 3D, which introduces bias along the light transport direction that the 2D blur variant avoids. Choose it when point-based photon mapping is employed and 3D spatial blurring is acceptable.

### [Extended Balance Heuristic](5601-extended-balance-heuristic.md)
A variance reduction technique used to weight estimates from a mixture of sampling techniques, combining unbiased path sampling with extended space volumetric estimators. It generalizes the standard balance heuristic by incorporating kernel density estimation properties into the weighting scheme, allowing for optimal combination of disparate estimators such as photon mapping and path tracing. It sets itself apart from siblings like Point Query × Point Data by explicitly accounting for the kernel support volume in the probability density function (pdf) calculation for volumetric techniques. Choose it when fusing results from different radiance estimators to produce a lower-variance final image.

### Short Beam Estimator via Russian Roulette
A technique to convert a long beam estimator into a short beam estimator by treating the beam length sampling as a Russian Roulette (RR) decision on the contribution [S1]. Long beams extend until the closest surface, with transmittance along the beam appearing as part of the estimator [S1]. Short beams are of finite extent where transmittance vanishes (replaced by a constant 1) via a single step function approximation, coined "progressive deep shadow maps" [S1]. The probability that a short beam of length $l$ contributes at distance $t_0$ is $\Pr\{l > t_0\} = \int_{t_0}^{\infty} p(t') dt'$, where $p(t)$ is the pdf used to sample the beam length [S1]. The short beam estimator is calculated as $\langle I \rangle_{\text{short}} = \langle I \rangle_{\text{long}} \frac{H(l - t)}{\Pr\{l > t\}}$, where $H$ is the Heaviside step function indicating if the beam is long enough [S1]. This approach provides an intuitive explanation and generalization of Jarosz et al.'s [2011b] unbiased progressive deep shadow maps [S1]. Choose it to simplify the handling of beam termination and transmittance in complex scenes [S1].

## Background
The family of Volumetric Radiance Estimators was introduced by Jarosz et al. in 2011 [S1]. The framework was later generalized and unified with unbiased path tracing methods in the Unified Points, Beams, and Paths (UPBP) algorithm, which allows for the combination of these estimators with unbiased path integral estimators using the Extended Balance Heuristic [S1]. The concept of "short beams" was also introduced in related work by Jarosz et al. [2011b] as "progressive deep shadow maps" [S1].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
