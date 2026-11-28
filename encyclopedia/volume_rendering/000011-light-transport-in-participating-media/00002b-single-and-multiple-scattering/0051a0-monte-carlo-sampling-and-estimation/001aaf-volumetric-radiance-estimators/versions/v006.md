# Volumetric Radiance Estimators

Also called: B-B1D, B-B2D, B-B3D, B-P2D, B-P3D, P-B2D, P-B3D, P-P3D

## Overview
Volumetric Radiance Estimators are a family of nine distinct methods introduced by Jarosz et al. [2011a] that estimate equilibrium radiance in participating media by reconstructing continuous fields from discrete samples (photons or beams) using spatial blurring kernels [S1]. They solve the problem of efficiently computing in-scattered radiance for multiple scattering events where direct path tracing is prohibitively expensive or noisy. They differ from unbiased path integral estimators by not being formulated as general path sampling techniques, and they differ from sibling methods like [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md) by relying on pre-computed photon data structures rather than on-the-fly ray marching [S1].

## Shared principles
Builds on [Single and multiple scattering](35-single-and-multiple-scattering.md).

## Subtopics

The family consists of nine estimators defined by three dimensions: the data representation (Photon Points or Photon Beams), the query type (Point or Beam), and the kernel dimension (1D, 2D, or 3D) [S1]. The naming convention follows the order: Data × Query × Kernel Dimension (e.g., B-P3D is Beam Data, Point Query, 3D Kernel) [S1]. This opposes the convention used by Jarosz et al. [2011a] [S1].

### [Point Query × Point Data, 3D Blur](2235-point-query-point-data-3d-blur.md)
Point Query × Point Data, 3D Blur estimates the in-scattered radiance at a specific 3D point by averaging the energy of nearby discrete photon points within a spherical kernel. It solves the problem of reconstructing continuous radiance fields from discrete photon samples in participating media. It differs from its sibling, the Extended Balance Heuristic, by using a fixed spherical kernel and simple density estimation rather than balancing probabilities, and from its child, Beam Query × Beam Data, 2D Blur, by blurring energy in three dimensions rather than restricting it to a 2D cross-section perpendicular to a beam direction.

### [Extended Balance Heuristic](5601-extended-balance-heuristic.md)
The Extended Balance Heuristic is a variance reduction technique used to combine estimates from a mixture of sampling techniques, specifically unbiased path sampling and extended space volumetric estimators, within the framework of volumetric radiance estimation. It differs from standard balance heuristics by incorporating an "extended" probability density function (pdf) that accounts for the blurring or kernel operations inherent in volumetric estimators, allowing for optimal weighting of samples that are not point-wise exact. It is primarily used to merge results from different radiance estimators, such as those derived from photon mapping or vertex merging, to produce a lower-variance final image.

### [Beam Query × Beam Data, 2D Blur](1402-unified-points-beams-and-paths-upbp.md)
Beam Query × Beam Data, 2D Blur is a specific estimator within the Unified Points, Beams, and Paths (UPBP) framework that uses beam data and beam queries with a 2D kernel. It differs from point-based estimators by leveraging the directional coherence of photon beams, reducing variance in scenes with strong directional lighting. It is chosen when beam-based photon mapping is employed to capture directional light transport efficiently.

### Short Beam Estimator via Russian Roulette
A technique to convert a long beam estimator into a short beam estimator by treating the beam length sampling as a Russian Roulette (RR) decision on the contribution [S1]. It differs from long beam estimators by approximating transmittance with a single step function, effectively creating "progressive deep shadow maps" [S1]. It is chosen to simplify the handling of beam termination and transmittance in complex scenes [S1]. The probability that a short beam of length $l$ contributes at distance $t_0$ is $\Pr\{l > t_0\} = \int_{t_0}^{\infty} p(t') dt'$, where $p(t)$ is the pdf used to sample the beam length [S1]. The short beam estimator is calculated as $\langle I \rangle_{\text{short}} = \langle I \rangle_{\text{long}} \frac{H(l - t)}{\Pr\{l > t\}}$, where $H$ is the Heaviside step function indicating if the beam is long enough [S1]. This approach provides an intuitive explanation and generalization of Jarosz et al.'s [2011b] unbiased progressive deep shadow maps [S1].

## Background
The family of Volumetric Radiance Estimators was introduced by Jarosz et al. in 2011 [S1]. The framework was later generalized and unified with unbiased path tracing methods in the Unified Points, Beams, and Paths (UPBP) algorithm, which allows for the combination of these estimators with unbiased path integral estimators using the Extended Balance Heuristic [S1]. The concept of "short beams" was also introduced in related work by Jarosz et al. [2011b] as "progressive deep shadow maps" [S1].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
