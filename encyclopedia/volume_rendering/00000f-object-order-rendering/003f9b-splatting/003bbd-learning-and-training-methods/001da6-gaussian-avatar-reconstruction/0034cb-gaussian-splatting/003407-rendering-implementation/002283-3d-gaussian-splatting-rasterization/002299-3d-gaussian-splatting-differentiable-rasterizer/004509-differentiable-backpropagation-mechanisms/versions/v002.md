# Differentiable Backpropagation Mechanisms

Also called: Gradient computation for Gaussian Splatting, Backpropagation in 3DGS

## Overview
This family defines the mathematical and algorithmic mechanisms for computing gradients of the rendering loss with respect to 3D Gaussian parameters (position, scale, rotation, opacity, color) during the training of Gaussian Splatting models. It solves the problem of enabling end-to-end optimization of scene representations by providing differentiable paths from the final pixel color to the underlying 3D attributes. It differs from standard rasterization by explicitly deriving chain-rule gradients for alpha compositing and basis function evaluations, and from non-differentiable methods by supporting arbitrary gradient estimation strategies including per-splat parallelization and stochastic estimation.

## Shared principles
All mechanisms in this family rely on the differentiability of the forward rasterization pipeline defined in [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md). The core principle is the application of the chain rule to the alpha compositing equation:
$$ C_j = \sum_{s \in \mathcal{O}_j} c_s \alpha_s \prod_{k \in \mathcal{O}_j, k < s} (1 - \alpha_k) $$
where $C_j$ is the final color of pixel $j$, $c_s$ is the color of splat $s$, $\alpha_s$ is its opacity at pixel $j$, and $\mathcal{O}_j$ is the set of splats sorted by depth contributing to pixel $j$. Gradients flow back through this accumulation. The specific implementation details (parallelization strategy, basis function derivatives, or stochastic approximation) vary by child, but all must correctly propagate $\frac{\partial L}{\partial C_j}$ to $\frac{\partial L}{\partial \theta}$ where $\theta$ represents the Gaussian parameters.

## Subtopics

### [Backpropagation with Per-Splat Parallelization](3533-backpropagation-with-per-splat-parallelization.md)
A GPU kernel optimization that eliminates atomic contention by parallelizing the backward pass over 2D splats rather than pixels, using intra-warp shuffling for state exchange. Choose this for high-performance training on modern GPUs where memory bandwidth and atomic operations are bottlenecks.

### [DARBF Backpropagation](10254-darbf-backpropagation.md)
Computes gradients for Decaying Anisotropic Radial Basis Functions (DARBFs) by replacing standard Gaussian derivatives with specific terms derived from the DARBF activation function. Choose this when using non-Gaussian basis functions that require custom gradient formulas.

### [Stochastic Gradient Estimation for Differentiable Rasterization](11381-stochastic-gradient-estimation-for-differentiable-rasterizat.md)
Estimates gradients on a per-pixel basis using finite differences with random sign vectors, avoiding the need for a differentiable framework. Choose this when integrating with non-differentiable rasterizers or when reducing the dimensionality of the optimization problem is critical.

## Background
The concept of differentiable rendering has been explored since the 1990s, with early work on differentiable ray tracing and rasterization. The specific application to 3D Gaussian Splatting emerged with the original 3DGS paper [Kerbl et al., 2023], which introduced the differentiable rasterizer. Subsequent works have optimized this backpropagation for performance [Radl et al., 2024] or extended it to alternative basis functions [DARB-Splatting] and stochastic estimation techniques [S1].

## Sources
[S1] Stochastic Gradient Estimation for Differentiable Rasterization, https://example.com/stochastic-grad, 2024-05-20
[S2] Backpropagation with Per-Splat Parallelization, https://example.com/per-splat-backprop, 2024-06-15
