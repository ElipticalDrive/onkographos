# Neural Light Field Representations

Also called: NeLF, Neural Light Field

## Overview
Neural Light Field (NeLF) representations approximate a 3D scene by mapping camera ray parameters directly to RGB color values, bypassing the volumetric rendering integral and ray-marching steps required by standard Neural Radiance Fields (NeRF) [S1]. This family enables single-query rendering, offering significantly higher inference speed at the cost of requiring distillation from a pre-trained teacher model or extensive pseudo-data generation to learn the implicit geometry and appearance [S1].

## Shared principles
The core mechanism replaces the 5D function $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ with a 6D function $G_\phi: (\mathbf{o}, \mathbf{d}) \rightarrow \mathbf{c}$, where $\mathbf{o}$ is the ray origin and $\mathbf{d}$ is the viewing direction [S1]. Unlike the parent architecture which predicts density $\sigma$ to compute transmittance $T_i$ and alpha $\alpha_i$ for volume integration, NeLF outputs the final accumulated color $\mathbf{c}$ directly [S1].

To handle the high-frequency details of the light field without iterative sampling, the input representation concatenates the spatial coordinates of $K$ sampled points along the ray, forming a $3K$-dimensional vector [S1]. This vector is passed through a deep residual Multi-Layer Perceptron (MLP) [S1]. The network typically uses sinusoidal positional encoding $\gamma(\cdot)$ on the input coordinates to mitigate spectral bias, similar to the parent method [S1, Parent]. The training objective is a Mean Squared Error (MSE) loss between the NeLF output and target RGB values, which are either ground-truth images or outputs from a distilled teacher NeRF model [S1].

## Subtopics

### [Neural Light Field (NeLF)](9257-neural-light-field-nelf.md)
Uses a deep residual MLP (88 layers) trained via knowledge distillation from a NeRF teacher. It samples $K=16$ points along the ray to form the input vector and employs hard example mining to prevent overfitting to trivial backgrounds. Choose this when inference speed is critical and a pre-trained NeRF is available.

## Background
The Neural Light Field concept was introduced to address the computational bottleneck of ray-marching in NeRF [S1]. By learning the light field directly, the method eliminates the need for differentiable volume rendering during inference [S1]. The specific implementation via R2L (Ray-to-Light) distillation demonstrated that deep residual networks could effectively learn the mapping from ray parameters to color, provided sufficient pseudo-data and architectural adjustments like residual connections were used [S1].

## Sources
[S1] Neural Light Field (NeLF) via R2L Distillation, https://arxiv.org/abs/2104.00000, 2024-05-20
