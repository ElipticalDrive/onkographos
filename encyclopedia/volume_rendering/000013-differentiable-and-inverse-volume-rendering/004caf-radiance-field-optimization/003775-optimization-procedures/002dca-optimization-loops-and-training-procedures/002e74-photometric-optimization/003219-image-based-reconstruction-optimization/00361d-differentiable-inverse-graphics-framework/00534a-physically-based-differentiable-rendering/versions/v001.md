# Physically Based Differentiable Rendering
Also called: Differentiable Path Tracing

## Overview
Physically Based Differentiable Rendering (PBDR) is a rendering approach that differentiates through physically based integrators, such as path tracing, to preserve derivatives of light transport operations [S1, unofficial]. It solves the inverse rendering problem by enabling gradient-based optimization to estimate latent 3D scene parameters (geometry, materials, lighting) from observed images [S1, unofficial]. Unlike rasterization-based differentiable renderers, PBDR preserves the physical accuracy of the rendering equation, allowing for the optimization of complex material properties and lighting conditions [S1, unofficial].

## Shared principles
PBDR relies on the principle that the rendering function $f(\pi)$, which maps scene parameters $\pi$ to images, is differentiable with respect to $\pi$ [S1, unofficial]. The core mechanism involves minimizing a reconstruction loss $\mathcal{L}$ between the rendered image $f(\pi)$ and an observed image $y$ (or a target latent representation) by backpropagating gradients through the light transport simulation [S1, unofficial].

The image formation is governed by the rendering equation [S1, unofficial]:
$$ L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{S^2} f_s(x, \omega_i, \omega_o) L_i(x, \omega_i) \cos \theta_i d\omega_i $$
where $L_o$ is outgoing radiance, $L_e$ is emitted radiance, $f_s$ is the BSDF, $L_i$ is incident radiance, and $\theta_i$ is the angle between the surface normal and incoming direction [S1, unofficial].

The optimization objective is defined as:
$$ \pi^* = \arg \min_\pi \mathcal{L}(f(\pi), y) $$
where $\pi^*$ are the estimated scene parameters [S1, unofficial].

The gradient computation $\frac{d\pi}{d\pi}$ includes contributions from:
1.  BSDF parameters.
2.  Geometric changes affecting surface points $x$, normal directions, and $\cos \theta_i$.
3.  Derivatives of incident radiance $L_i$.
4.  Visibility terms [S1, unofficial].

A key challenge in PBDR is handling visibility discontinuities that occur at object boundaries and shadow edges. These discontinuities must be mitigated via techniques such as soft visibility, edge sampling, or regularization to ensure stable gradient computation [S1, unofficial].

## Subtopics

### [Metamers Rendered Differentiably (MRD)](14966-metamers-rendered-differentiably-mrd.md)
A method that uses PBDR to find 3D scene parameters that produce equivalent latent representations in a frozen neural network, rather than matching pixel values. It optimizes physical scene parameters to probe vision models' implicit understanding of 3D properties.

### [Differentiable Re-rendering Module](14687-differentiable-re-rendering-module.md)
(No details provided in sources for this sibling).

## Background
PBDR emerged as a third-generation differentiable rendering technique, building upon earlier methods that differentiated through rasterization [S1, unofficial]. Early work in 2014 allowed the reconstruction of scene parameters such as shape and material [S1, unofficial]. Modern PBDR implementations include Mitsuba3 and psdr, which can differentiate through path tracing integrators while preserving derivatives [S1, unofficial]. This distinguishes them from PyTorch3D and nvdiffrast, which primarily focus on rasterization [S1, unofficial]. The field continues to address technical challenges related to discontinuities and computational tractability [S1, unofficial].

## Sources
[S1] MRD: Using Physically Based Differentiable Rendering to Probe Vision Models for 3D Scene Understanding, https://doi.org/10.48550/arXiv.2512.12307, fetched 2026-10-29 (unofficial)
