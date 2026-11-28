# Differentiable Rendering

Also called: Differentiable Splatting, Gradient-Based Rendering

## Overview
Differentiable rendering treats the image formation process as a differentiable function, enabling the computation of gradients with respect to scene parameters (geometry, appearance, pose) via backpropagation [G]. This allows for gradient-based optimization of 3D representations directly from 2D image observations [G]. It sets itself apart from standard rendering by exposing internal state gradients to upstream optimizers, facilitating tasks like pose estimation, shape reconstruction, and sequential affordance reasoning for robotic planning [S1, unofficial][S2]. It builds on [Splatting](23-splatting.md) by differentiating the rasterization and compositing steps.

## Shared principles

## Subtopics

### [Differentiable Forward Warping with Average Splatting](8921-differentiable-forward-warping-with-average-splatting.md)
Solves the "many-to-one" mapping issue in temporal deformation by fusing source voxel values into a target grid using trilinear kernel weights, addressing density collisions during contraction.

### [Learning and Training Methods](9068-learning-and-training-methods.md)
Focuses on optimizing primitive parameters and network weights through differentiable rendering pipelines to learn 3D representations from 2D images [G].

### [Differentiable Robot Rendering (Dr. Robot)](15698-differentiable-robot-rendering-dr-robot.md)
Integrates kinematics-aware deformation with 3D Gaussian Splatting using Implicit Linear Blend Skinning (LBS) to provide accurate gradients for robotic control and planning.

### SeqAffordSplat
Performs sequential affordance reasoning on the 3D Gaussian Splatting (3DGS) representation, where rendered feature fields encode interaction probabilities over time, allowing robots to plan actions in the rendered 3D scene [S1, unofficial].

### Differentiable Rendering (Forward Pass)
For Fourier splatting, the forward pass implements a tile-based rasterizer in polar coordinates [S2]. For each pixel $j$ with polar coordinates $(\rho_j, \theta_j)$ in the tangent plane of primitive $s_i$, the signed distance to the boundary is computed as $\varphi_j = \rho_j - r_i(\theta_j)$ [S2]. Points with $\varphi_j \ge 0$ are discarded [S2]. The per-pixel opacity is calculated as $\alpha_{i,j} = o_i \cdot \max\left(0, \frac{r_i(\theta_j) - \rho_j}{r_i(\theta_j)}\right)^{\sigma_i}$ [S2]. Colors are accumulated via front-to-back compositing [S2].

## Background
Differentiable rendering has evolved from volume rendering techniques to support modern neural graphics [G]. Early work focused on volume rendering equations, where opacity $\alpha$ is computed by integrating density along the ray: $\alpha_i = 1 - \exp\left(-\int G_i(x) dx\right)$ [S1, unofficial]. Recent advancements integrate these principles with Gaussian Splatting, enabling applications like robot learning and 3D reconstruction [S1, unofficial][S2]. Fourier Splatting extends this by using generalized Fourier encoded primitives, adapting power window functions to polar coordinates for scalable radiance fields [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Fourier Splatting: Generalized Fourier encoded primitives for scalable radiance fields, https://doi.org/10.48550/arXiv.2603.19834, fetched 2026-10-30
