# SGGX Microflake Distribution


## Overview
The SGGX microflake distribution is a normal distribution function for modeling microflake orientations [S1]. It solves the problem of efficiently computing attenuation coefficients and sampling specular phase functions for anisotropic microflake ensembles [S1]. It serves as the foundational distribution for the [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md) and its simplified self-shadowing variant [S1].

## How it works
**Inputs and Outputs:** Not given in the sources.
**Data Structures:** Not given in the sources.
**Formulas:** Not given in the sources.
**Steps for Evaluation:** # Not given in the sources.
**Parameters:** Not given in the sources.
**Edge Cases:** Not given in the sources.
**Sampling:** The method provides efficient sampling procedures for the distribution of visible normals, avoiding the inefficiency of rejection sampling in some configurations [S1].

## Subtopics

### [GGX Mixture NDF](10859-ggx-mixture-ndf.md)
The GGX Mixture NDF is a compact representation of the marginal distribution of surface normals, $p_N(n)$, used to model the glossiness and anisotropy of aggregated microflake appearances. It solves the problem of representing complex, arbitrarily oriented surface normal distributions by combining multiple SGGX lobes. It is distinguished from single-lobe SGGX by its ability to capture multi-modal or highly anisotropic normal distributions through a weighted sum, building on the [SGGX Microflake Distribution](6660-sggx-microflake-distribution.md).

## Operating characteristics
The SGGX distribution supports fast and exact evaluation of attenuation coefficients, as well as efficient evaluation and sampling of specular phase functions [S1].

## Background
The SGGX distribution was introduced by Heitz et al. in [HDCD15] [S1]. It was developed to overcome limitations of previous microflake normal distributions, which relied on inefficient sampling methods or approximations for attenuation [S1]. The distribution has been used for downsampling microflakes normal distributions [S1] and for implementing the simplified self-shadowing model [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
