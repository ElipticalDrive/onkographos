# SGGX Microflake Distribution

Also called: SGGX, Spherical Gaussian Gabor X, Spherical Gaussian Generalized X

## Overview
The SGGX (Spherical Gaussian Generalized X) microflake distribution is a normal distribution function (NDF) designed for modeling aggregated microflakes in participating media [S1]. It solves the problem of efficiently computing attenuation coefficients and sampling specular phase functions for anisotropic microflake ensembles [S1]. It is distinguished from its siblings, such as the [Simplified Model with Isotropic Self-Shadowing](4761-simplified-model-with-isotropic-self-shadowing.md), by its ability to represent complex, arbitrarily oriented anisotropic normal distributions using a single lobe, enabling exact and fast evaluation of optical properties [S1].

## How it works
The SGGX distribution models the marginal distribution of surface normals $p_N(\mathbf{n})$ for a population of microflakes. It is defined by a mean direction and a covariance matrix, allowing it to capture both isotropic and anisotropic orientations [S1].

**Inputs and Outputs:**
*   Input: Mean direction vector $\mathbf{m}$ (unit vector), covariance matrix $\mathbf{C}$ (3x3 symmetric positive-definite), and a normal direction $\mathbf{n}$ (unit vector).
*   Output: Probability density value $p_N(\mathbf{n})$.

**Data Structures:**
The distribution is parameterized by:
1.  $\mathbf{m}$: A 3D vector representing the mean orientation.
2.  $\mathbf{C}$: A 3x3 symmetric matrix representing the spread and anisotropy of the normals.

**Formulas:**
The SGGX NDF is defined as:
$$ p_N(\mathbf{n}) = \frac{1}{Z} \exp\left( -\frac{1}{2} \mathbf{n}^T \mathbf{C}^{-1} \mathbf{n} \right) $$
where $Z$ is a normalization constant ensuring the integral over the hemisphere equals 1. The matrix $\mathbf{C}$ is often derived from the second moments of the normal distribution.

**Steps for Evaluation:**
1.  Compute the inverse of the covariance matrix $\mathbf{C}^{-1}$.
2.  Calculate the quadratic form $\mathbf{n}^T \mathbf{C}^{-1} \mathbf{n}$.
3.  Compute the exponential term.
4.  Normalize by $Z$.

**Sampling:**
Efficient sampling procedures are provided for the distribution of visible normals, avoiding the inefficiency of rejection sampling in some configurations [S1].

**Edge Cases:**
*   Degenerate covariance matrices (singular $\mathbf{C}$) must be handled to avoid numerical instability in inversion.
*   Boundary conditions for normals on the hemisphere surface must be respected.

## Subtopics

### [GGX Mixture NDF](10859-ggx-mixture-ndf.md)
The GGX Mixture NDF is a compact representation of the marginal distribution of surface normals, $p_N(n)$, used to model the glossiness and anisotropy of aggregated microflake appearances. It solves the problem of representing complex, arbitrarily oriented surface normal distributions by combining multiple SGGX lobes. It is distinguished from single-lobe SGGX by its ability to capture multi-modal or highly anisotropic normal distributions through a weighted sum, building on the [SGGX Microflake Distribution](6660-sggx-microflake-distribution.md).

## Operating characteristics
The SGGX distribution supports fast and exact evaluation of attenuation coefficients, as well as efficient evaluation and sampling of specular phase functions [S1]. This makes it suitable for real-time or interactive volume rendering applications where performance is critical. The exact evaluation of attenuation avoids the approximations inherent in other microflake models, leading to higher accuracy in light transport simulation.

## Background
The SGGX distribution was introduced by Heitz et al. in [HDCD15] [S1]. It was developed to overcome limitations of previous microflake normal distributions, which often relied on inefficient sampling methods or approximations for attenuation. The distribution has been used in subsequent work, such as the simplified self-shadowing model described in [S1], and for downsampling microflakes normal distributions [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
