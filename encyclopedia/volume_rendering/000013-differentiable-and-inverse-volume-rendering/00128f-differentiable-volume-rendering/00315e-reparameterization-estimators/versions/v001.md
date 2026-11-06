# Reparameterization Estimators

Also called: Reparameterization Trick, Score Function Estimator (contrast), Path Space Reparameterization

## Overview
Reparameterization Estimators compute gradients for differentiable volume rendering by transforming stochastic sampling operations into deterministic, differentiable functions of the underlying parameters. Unlike standard Monte Carlo estimators where gradients vanish through discrete sampling decisions, this family allows back-propagation through the sample locations themselves. It distinguishes itself from [Adjoint Method](789-adjoint-method.md) by maintaining a forward-mode computational graph for the sampling process, and from [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) by explicitly handling the non-differentiable nature of discrete ray marching via continuous density approximations.

## Shared principles
The core mechanism relies on the **reparameterization trick**: expressing a random variable $Z$ as a deterministic function $g(\epsilon; \theta)$ of a parameter $\theta$ and an auxiliary noise variable $\epsilon \sim p(\epsilon)$ [S1]. This allows the gradient of the expectation $\mathbb{E}_{Z \sim p_\theta}[f(Z)]$ to be computed as $\mathbb{E}_{\epsilon}[ \nabla_\theta f(g(\epsilon; \theta)) ]$, shifting the differentiation burden from the probability density to the transformation function.

In volume rendering, this typically involves:
1.  **CDF Inversion:** Defining the sampling distribution via the Cumulative Distribution Function (CDF) of the density field, $F(t) = 1 - \exp(-\int \sigma)$.
2.  **Differentiable Inversion:** Solving $F(t) = u$ for $t$ given uniform noise $u \in [0,1]$. The solution $t(u, \sigma)$ must be differentiable with respect to $\sigma$.
3.  **Smooth Densities:** Ensuring $\sigma$ is a smooth function (e.g., using Softplus) to avoid zero gradients associated with ReLU activations [S1].

## Subtopics

### [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md)
Uses piecewise linear density approximation to derive a closed-form, differentiable inverse CDF (quadratic solver). It avoids auxiliary networks, offering lower variance than uniform sampling.

### [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md)
Applies the reparameterization trick to shape parameters $\theta$ by mapping interior volume integrals to boundary area integrals via the divergence theorem. It eliminates the need for interior tetrahedralization, solving the problem of expensive interior mapping evaluations.

### [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md)
Estimates radiance by sampling directly from the density-induced distribution. It differs from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) by potentially relying on hierarchical schemes or auxiliary coarse models for efficiency, whereas the latter learns sampling points end-to-end.

## Background
The reparameterization trick was introduced by [S1] in the context of variational autoencoders and has been adapted for differentiable rendering to handle the non-differentiable nature of ray-sampling intersections. In volume rendering, it addresses the discontinuity in gradients when sample points jump between intervals in discrete ray marching. The Boundary Path Integral variant extends this to geometric optimization, allowing shape gradients without volumetric mesh derivatives [S1].

## Sources
[S1] Reparameterized Volume Sampling notes, URL, date fetched
