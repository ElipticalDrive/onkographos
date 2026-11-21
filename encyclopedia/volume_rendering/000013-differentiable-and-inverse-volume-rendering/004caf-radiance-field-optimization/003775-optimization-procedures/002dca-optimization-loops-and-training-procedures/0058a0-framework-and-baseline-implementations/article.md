# Framework and Baseline Implementations


## Overview
This family encompasses the software architectures and training protocols used to execute differentiable volume rendering optimization. It provides the computational infrastructure for evaluating the rendering equation and propagating gradients to scene parameters, serving as the operational layer for specific methods like NeRF and Gaussian Splatting. It is distinct from its siblings by defining the low-level execution model (e.g., graph-based vs. eager-mode) and the high-level training schedule (e.g., distractor separation, multi-stage optimization) rather than the mathematical loss functions or camera models themselves.

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [DressiAD Optimization Loop](10728-dressiad-optimization-loop.md)
A define-and-run differentiable rendering optimization procedure that constructs a computational graph, marks variables for gradient tracking, and executes forward/backward passes with Stochastic Gradient Descent (SGD). It is distinct from eager-mode frameworks by explicitly managing graph traversal, packing the computation into Vulkan command buffers, and handling dirty-flag-based graph rebuilding.

### [Training & Optimization Protocol](12939-training-optimization-protocol.md)
A protocol for decomposed Gaussian splatting methods, specifically targeting distractor-free rendering by separating static scene geometry from dynamic or erroneous "distractor" elements. It differs from standard Gaussian Splatting by explicitly initializing and optimizing distractor points alongside static points, using specific learning rate schedules and regularization terms to suppress artifacts.

### 4DGS Baseline Implementation
This variant optimizes multi-view video sequences using a two-stage process. First, the first frame is optimized for 3,000 iterations. Second, the full multi-view video sequence is optimized for 14,000 iterations [S1].

## Background
The implementation of differentiable rendering optimization loops has evolved from manual gradient derivation to automated graph-based systems. Early implementations relied on analytic inversion to manage memory [44-radiance-field-optimization.md]. Modern frameworks like DressiAD [10728-dressiad-optimization-loop.md] integrate these principles with GPU compute APIs (Vulkan) for efficient execution. In the context of Gaussian Splatting, protocols like the one for DeSplat [12939-training-optimization-protocol.md] have been developed to handle specific artifacts like distractors, while variants like 4DGS [S1] adapt these loops for temporal consistency in video data.

## Sources
[S1] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
