# Tabulated Importance Sampling for Anisotropic Scattering

Also called: Adaptive tabulation of geometry and phase functions, Tabulated unidirectional/bidirectional factorization

## Overview
A method to efficiently tabulate target probability density functions (PDFs) for unidirectional and bidirectional factorizations to handle anisotropic scattering in participating media [S1]. It solves the problem of analytically unavailable conditional PDFs for double-scattering paths by precomputing accurate approximations in a compact 2D table, avoiding the limited accuracy and overhead of on-the-fly tabulation [S1]. Unlike [Unidirectional Importance Sampling for Double-Scattering](2151-unidirectional-importance-sampling-for-double-scattering.md) which relies on analytic formulations for isotropic media, this method supports general anisotropic phase functions by exploiting symmetries to reduce dimensionality [S1].

## How it works
**Inputs:**
- Ray origin $c$ and direction $\omega_{cb}$ [S1].
- Previous vertex $a$ and incoming direction $\omega_{dc}$ (encoded in $\Xi$) [S1].
- Phase function $\rho(\cdot)$, assumed to be circularly-symmetric 1D (depending only on deflection angle); isotropic scattering is the special case $\rho = 1/4\pi$ [S1].

**Outputs:**
- Sampled scattering distance $t_{cb}$ and direction $\omega_{cb}$ (for unidirectional factorization) [S1].
- Corresponding PDF values for importance sampling [S1].

**Data Structures:**
- **2D Lookup Table:** Indexed by canonical spherical coordinates $(\theta_{la}, \phi_{la})$ of the direction vector $\omega_{la}$ [S1].
- **Table Entries:** Each entry is a piecewise-linear approximation of a 1D PDF over $\theta \in [0, \pi]$ [S1].

**Table Construction (Unidirectional Factorization):**
1. **Canonical Coordinate System:** For a given ray $(c, \omega_{cb})$ and vertex $a$: rescale the problem such that the distance between the line defined by $(c, \omega_{cb})$ and vertex $a$ is one [S1]; place the origin at $a$ [S1]; align the $z$-axis with the ray direction $\omega_{cb}$ [S1]; align the $x$-axis in the plane containing $a$ and the ray [S1].
2. **Indexing:** The table is indexed by the direction vector $\omega_{la}$ (the direction from $a$ to the scattering point $b$), represented by spherical coordinates $(\theta_{la}, \phi_{la})$ in the canonical system [S1].
3. **PDF Definition:** The target PDF for the distance $t_{cb}$ is proportional to the product of geometry and scattering terms: $p(t_{cb} \mid \omega_{cb}, t_{dc}, \Xi) \propto G(a,b) \rho(a) \rho(b)$, where $G(a,b)$ is the geometry term and $\rho$ is the phase function [S1].
4. **Reparameterization:** To handle sharp peaks or singularities in the geometry term, the infinite line is mapped to a finite angular domain using $\theta = \text{arccot } u$, where $u$ is the signed distance from the projection of $a$ onto the line [S1].
5. **Tabulated Function:** Each entry stores a piecewise-linear approximation of $p(\theta(u)) = p(u) \left| \frac{du}{d\theta} \right| = p(u) \frac{1}{G(a,b(u))} \propto \rho(a) \rho(b(u))$ [S1]. This transformation analytically eliminates geometric variations, improving tabulation accuracy [S1].
6. **Adaptive Refinement:** The 1D PDF $p(\theta)$ is approximated using adaptive refinement over $\theta \in [0, \pi]$ [S1].

**Sampling (Unidirectional Factorization):**
1. **Coordinate Setup:** Construct the canonical coordinate system for the specific ray $(c, \omega_{cb})$ and vertex $a$ as described in construction [S1].
2. **Table Lookup:** Index the 2D table using the canonical spherical coordinates of $\omega_{la}$ [S1].
3. **Angle Sampling:** Draw an angle $\theta_b$ from the retrieved tabulated PDF, constrained to the interval $\theta_b \in [0, \text{arccot } u_c]$, where $u_c$ is the canonical distance from the projection of $a$ to the ray origin $c$ [S1].
4. **Distance Conversion:** Transform the sampled angle to a canonical distance: $u_b = \cot \theta_b$ [S1]. Compute the scattering distance: $t_{cb} = -u_c + u_b$ [S1].
5. **PDF Transformation:** Transform the PDF from the angular measure to the Euclidean length measure for the final importance sampling weight: $p(u_b) = p(\theta_b) \left| \frac{d\theta_b}{du_b} \right| = p(\theta_b) [G(a,b) h_{ca^\perp}]$, where $h_{ca^\perp}$ is the distance between $a$ and the ray [S1].

**Pitfalls:**
- **Geometric Singularities:** Geometric terms can have sharp peaks or singularities, which reduce tabulation accuracy; the reparameterization $\theta = \text{arccot } u$ is critical to analytically eliminate these variations [S1].
- **Precision:** The mapping from infinite line to finite angular domain requires careful handling of boundaries, particularly near $\theta = 0$ and $\theta = \pi$ [S1].

## Operating characteristics
- **Time Complexity:** Precomputation is done once per configuration family; sampling is $O(1)$ table lookup and inversion, significantly faster than on-the-fly tabulation [S1].
- **Memory Complexity:** Compact 2D table of 1D PDFs; dimensionality is reduced by exploiting symmetries, storing only the necessary angular parameters [S1].
- **Accuracy:** Achieves nearly ideal importance sampling of the product of geometry and scattering terms, outperforming on-the-fly methods [S1].
- **Strengths:** Handles anisotropic scattering; avoids analytic formulation requirements; high accuracy with low sampling overhead [S1].
- **Limitations:** Requires precomputation; assumes circularly-symmetric phase functions [S1].

## Background
Introduced in [S1] to tabulate target PDFs for anisotropic scattering without requiring analytic formulations, extending on-the-fly tabulation approaches [S1]. The method is part of a broader effort to efficiently sample double-scattering paths in participating media, building on [Unidirectional Importance Sampling for Double-Scattering](2151-unidirectional-importance-sampling-for-double-scattering.md) but generalizing to anisotropic cases [S1].

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
