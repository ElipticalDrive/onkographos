# Gradient Computation and Back-Propagation

Also called: Differentiable Rasterization Gradients, Back-Propagation in 3DGS

## Overview
This article defines the mathematical and algorithmic foundations for computing gradients of the loss function with respect to 3D Gaussian parameters during the training of 3D Gaussian Splatting. It addresses the problem of differentiating through the volume rendering equation, which is non-linear and order-dependent, to enable end-to-end optimization of Gaussian attributes. It sets apart from standard differentiable rendering by handling the specific opacity-transmittance coupling and recursive accumulation inherent to the Gaussian splatting rasterizer, extending to specialized variants for Time-of-Flight signals and feature back-projection.

## Shared principles
The core mechanism relies on the chain rule applied to the volume rendering integral, discretized as a sum over sorted Gaussians. For a pixel, the rendered signal $S$ is computed via back-to-front accumulation. The gradient $\frac{\partial L}{\partial \theta_k}$ for a parameter $\theta_k$ of the $k$-th Gaussian depends on the gradient of the loss w.r.t. the final signal $\frac{\partial L}{\partial S}$ and the partial derivative of the signal w.r.t. the parameter.

The standard transmittance $T_k$ is defined as the product of non-opacities of all preceding Gaussians:
$$ T_k = \prod_{m < k} (1 - \alpha_m) $$
The rendered color $C$ is:
$$ C = \sum_{k} c_k \alpha_k T_k $$
The gradient of $C$ w.r.t. opacity $\alpha_k$ involves both the direct term and the indirect term through $T_j$ for $j > k$. In standard 3DGS, this results in a gradient proportional to $T_k(c_k - C_{accumulated})$.

For ToF signals, the accumulation model changes to include squared transmittance terms to account for non-linear intensity interactions, requiring modified recursive accumulators and gradient formulas.

For feature back-projection, the gradient of the rendered color w.r.t. the Gaussian color $\frac{\partial C}{\partial c_k} = \alpha_k T_k$ is used as a weighting factor to aggregate 2D features from training views into 3D Gaussians without iterative optimization.

## Subtopics

### [ToF Gradient Computation for Gaussian Splatting](9733-tof-gradient-computation-for-gaussian-splatting.md)
Computes gradients for Time-of-Flight image formation models. It extends the standard opacity gradient by introducing squared transmittance terms in the recursive accumulation, addressing the non-linear interaction between opacity and ToF signal intensity. Choose this when optimizing Gaussian parameters against ToF quad pixel signals.

### [Gradient-Weighted Feature Back-Projection](9769-gradient-weighted-feature-back-projection.md)
A training-free method for rendering feature fields by aggregating 2D feature embeddings into 3D Gaussians. It uses the gradient of rendered color w.r.t. Gaussian color as a weight. Choose this for semantic segmentation or affordance transfer without the cost of training a separate feature field.

### [SNS Backward Pass with Intermediate Variable Q](10040-sns-backward-pass-with-intermediate-variable-q.md)
Introduces an intermediate variable $Q = ARS$ to decouple rotation $R$ and scale $S$ in gradient computation, avoiding complex coupling in the Jacobian. Choose this when optimizing rotation and scale parameters to simplify gradient derivation.

## Background
The gradient computation for 3D Gaussian Splatting is derived from the differentiable volume rendering framework. The standard back-propagation through the rasterizer was established in the original 3DGS paper [S1]. Extensions for ToF signals and feature fields have been developed to support specialized applications like depth estimation and semantic mapping.

## Sources
[S1] Gradient Computation and Back-Propagation Notes, 2024
