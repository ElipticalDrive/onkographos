# 3D Student's t Distribution Splatting

Also called: Student Splatting and Scooping, Student's t distribution mixture model with negative components, Student's t splatting

## Overview
3D Student's t Distribution Splatting (SSS) is a neural rendering method that replaces the 3D Gaussian primitives of 3D Gaussian Splatting (3DGS) with flexible Student's t distributions and extends the splatting scheme to include both positive (splatting) and negative (scooping) density components [S1]. It solves the expressivity limitations of monotonic Gaussian mixtures by allowing fat tails and non-monotonic density modeling, achieving higher parameter efficiency (reducing component counts by up to 82%) while maintaining or improving rendering quality [S1]. Unlike 3DGS, SSS requires Stochastic Gradient Hamiltonian Monte Carlo (SGHMC) for optimization to prevent mode collapse in the learnable degrees of freedom parameter $\nu$ [S1].

## How it works
**Inputs:** A set of 3D primitives, each defined by a mean position $\boldsymbol{\mu} \in \mathbb{R}^3$, a covariance matrix $\Sigma \in \mathbb{R}^{3\times3}$, a degree of freedom $\nu \in [1, 10000]$, opacity $o \in [0, 1]$, and color coefficients $c \in \mathbb{R}^{27}$ (Spherical Harmonics) [S1]. The covariance is parameterized as $\Sigma = R S S^T R^T$ where $R$ is rotation and $S$ is scaling [S1].

**Outputs:** A 2D image where each pixel color $C(u)$ is computed via alpha blending of projected 2D Student's t distributions [S1].

**Data Structures:**
Each primitive $i$ stores:
- $\boldsymbol{\mu}_i$: 3D mean (vector).
- $R_i, S_i$: Rotation matrix and scaling matrix (factorized covariance).
- $\nu_i$: Scalar degree of freedom.
- $o_i$: Opacity scalar.
- $c_i$: Color coefficients vector.
- $w_i$: Weight scalar (derived from $o_i$ and compositing, initially positive) [S1].

**Rendering Steps:**
1. **Ray Casting:** For each pixel $u$, cast a ray into the scene.
2. **Primitive Selection:** Identify all primitives $i$ intersecting the ray.
3. **Projection:** For each primitive, compute the 2D projected mean $\boldsymbol{\mu}^{2D}$ and covariance $\Sigma^{2D}$:
   - Compute intermediate matrices: $A = JW$ and $b = \boldsymbol{x} + J(\boldsymbol{d} - \boldsymbol{t})$, where $W$ is the affine view transformation, $J$ is the approximate projective Jacobian, $\boldsymbol{t}$ is camera coordinates, and $\boldsymbol{x}, \boldsymbol{d}$ are related to the ray origin and direction [S1].
   - Compute projected mean: $\boldsymbol{\mu}^{2D} = (W\boldsymbol{\mu} + \boldsymbol{t})_{1:2} / ((W\boldsymbol{\mu} + \boldsymbol{t})_3)$ [S1].
   - Compute projected covariance: $\Sigma^{2D} = (JW\Sigma W^T J^T)_{1:2,1:2}$ [S1].
   - Note: The normalization constant of the 3D distribution is dropped to facilitate learning [S1].
4. **2D Density Evaluation:** Evaluate the 2D Student's t distribution $T_i^{2D}(u)$ at pixel $u$:
   $$T_i^{2D}(u) = \left[1 + \frac{1}{\nu_i}(u - \boldsymbol{\mu}^{2D})^T(\Sigma^{2D})^{-1}(u - \boldsymbol{\mu}^{2D})\right]^{-\frac{\nu_i+2}{2}}$$
   [S1].
5. **Alpha Blending:** Composite colors using the standard alpha blending equation, adapted for the t-distribution density:
   $$C(u) = \sum_{i=1}^N c_i o_i T_i^{2D}(u) \prod_{j=1}^{i-1} (1 - o_j T_j^{2D}(u))$$
   [S1]. Here, $N$ is the number of intersecting primitives sorted by depth.

**Optimization:**
- **SGHMC:** Vanilla SGD causes $\nu$ to collapse to local minima (typically near 1). SSS uses Stochastic Gradient Hamiltonian Monte Carlo (SGHMC) to decouple parameters and explore the $\nu$ space effectively [S1].
- **Negative Components (Scooping):** SSS introduces negative components to allow subtracting density, improving expressivity. The mixture model is non-monotonic. While a squared form $P(x) = (\sum w_i T_i(x))^2$ was considered, it introduces pairwise interactions. The final implementation uses positive and negative components with a linear combination approach implied by the "splatting and scooping" description, though the exact final rendering equation for the non-monotonic mixture is not fully detailed in the provided excerpt beyond the standard alpha blending with potentially negative weights or densities [S1].
- **Low-Pass Filter:** Unlike 3DGS, SSS does not use an analytical low-pass filter for minimum scale because the convolution of two Student's t distributions lacks an analytical form [S1]. For settings using only SGD with positive t-distributions, a small value (0.3) is added to the projected covariance matrix to prevent excessive tiny components [S1].

**Gradients:**
The backward pass replaces Gaussian derivatives with t-distribution derivatives via the chain rule [S1]. Define helper function:
$$h(x) = (x - \boldsymbol{\mu}')^T (\Sigma')^{-1} (x - \boldsymbol{\mu}')$$
where $\boldsymbol{\mu}'$ and $\Sigma'$ are the projected 2D mean and covariance [S1].
- Gradient w.r.t $h(x)$:
  $$\frac{\partial T^{2D}}{\partial h(x)} = -\frac{\nu + 2}{2} \cdot \frac{1}{\nu} \cdot \left[1 + \frac{1}{\nu} h(x)\right]^{-\frac{\nu+4}{2}}$$
  [S1].
- Gradient w.r.t $\nu$:
  $$\frac{\partial T^{2D}}{\partial \nu} = \frac{\nu+2}{2} \cdot \frac{h(x)}{\nu^2} \left[ 1 + \frac{1}{\nu} h(x) \right]^{-\frac{\nu+4}{2}}$$
  [S1].
- Gradients w.r.t $\boldsymbol{\mu}$ and $\Sigma$ follow the chain rule through $h(x)$ and the projection transformations, similar to 3DGS but using the t-distribution derivatives [S1].

**Component Recycling:**
When relocating low-opacity components to high-opacity ones, distribution invariance is maintained. For opacity, the relation is $(1 - O_{new})^N = (1 - O_{old})$ [S1].

## Subtopics

### SGHMC with Adam gradient
A hybrid optimization scheme for training Student's t-splatting that uses SGHMC as the base sampler but injects Adam's adaptive gradient scaling [S1]. It differs from vanilla 3DGS by replacing standard SGD with SGHMC+Adam, and from 3DGS-MCMC by incorporating Adam's gradient scaling and supporting negative components [S1]. Hyperparameters are largely identical to original 3DGS and 3DGS-MCMC, with new parameters introduced due to the Student's t distribution, negative components, and SGHMC integration [S1].

## Operating characteristics
- **Time Complexity:** Rendering is real-time (>70fps) on an NVIDIA RTX 4090, but slower than vanilla 3DGS due to the more complex t-distribution evaluation and SGHMC sampling overhead [S1].
- **Memory Complexity:** Parameter efficiency is significantly higher; SSS can achieve comparable results to 3DGS while reducing the number of components by up to 82% [S1].
- **Strengths:** Higher expressivity due to fat tails (covering larger areas with higher densities) and non-monotonic modeling (scooping) [S1]. Better quality with fewer components [S1].
- **Limitations:** Slower training and rendering than 3DGS. SGHMC sampling is slower than vanilla SGD [S1]. Requires careful handling of $\nu$ to prevent collapse [S1].
- **When to use:** When parameter efficiency and high expressivity are prioritized over raw training speed, or when Gaussian primitives fail to capture complex details with fat tails [S1].

## Background
SSS was proposed as an improvement on the fundamental paradigm of 3D Gaussian Splatting (3DGS) [S1]. It addresses the limitations of monotonic Gaussian mixture models by introducing flexible Student's t distributions and negative density components [S1]. The method was evaluated on Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets, demonstrating state-of-the-art quality and parameter efficiency [S1]. The initial 3DGS method was presented as a neural rendering technique [S1].

## Sources
[S1] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
