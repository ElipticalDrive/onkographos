# Approximated Gradient Back-Propagation

Also called: Aggressive Marching with Gradient Approximation

## Overview
Approximated Gradient Back-Propagation is a method for computing gradients in differentiable volume rendering that optimizes for memory efficiency by ignoring high-order gradient terms. It solves the problem of excessive memory consumption during the back-propagation of sphere tracing through implicit functions (such as SDFs) by decoupling the gradient computation from the precise dependency of sample points on the latent code. It distinguishes itself from exact gradient methods like [Differentiable Depth Rendering](1332-differentiable-depth-rendering.md) by sacrificing the exact chain rule for the spatial position dependency to enable feasible optimization of latent codes in large-scale scenes.

## How it works
The method computes gradients for a latent code $\mathbf{z}$ that defines an implicit surface function $f_\theta$. The rendering process involves marching along rays to sample the function.

**Inputs and Outputs:**
- Input: Latent code $\mathbf{z}$, camera ray origin $\mathbf{o}$, direction $\mathbf{d}$.
- Output: Rendered depth/value and gradients $\frac{\partial \text{Loss}}{\partial \mathbf{z}}$.

**Data Structures:**
- $\mathbf{p}^{(i)}$: The 3D position of the $i$-th sample point on the ray.
- $N$: Total number of marching steps.
- $\alpha$: Transmittance/weight factor for the sample.

**Numbered Steps:**
1. **Initialization**: Initialize the first point $\mathbf{p}^{(0)}$ at the intersection of the camera ray and the unit sphere. If the ray does not intersect, mark as background [S1].
2. **Forward Marching**: Perform sphere tracing or aggressive marching to generate sample points $\mathbf{p}^{(i)}$ for $i=0 \dots N-1$. These points are computed using a fixed initialization $\mathbf{z}_0$ or fixed previous positions, effectively treating the path as independent of the current $\mathbf{z}$ during the forward pass for gradient purposes [S1].
3. **Gradient Approximation**: Compute the gradient of the loss with respect to $\mathbf{z}$ by summing the direct partial derivatives of the SDF network with respect to $\mathbf{z}$ at the sampled points. The term involving the derivative of the point position with respect to $\mathbf{z}$ is ignored [S1].

**Formula:**
The full chain rule for the gradient of the SDF $f_\theta$ with respect to $\mathbf{z}$ at sample $i$ is:
$$
\frac{\partial f_\theta(\mathbf{p}^{(i)}(\mathbf{z}), \mathbf{z})}{\partial \mathbf{z}} = \frac{\partial f_\theta(\mathbf{p}^{(i)}(\mathbf{z}), \mathbf{z})}{\partial \mathbf{z}} + \frac{\partial f_\theta(\mathbf{p}^{(i)}(\mathbf{z}), \mathbf{z})}{\partial \mathbf{p}^{(i)}(\mathbf{z})} \frac{\partial \mathbf{p}^{(i)}(\mathbf{z})}{\partial \mathbf{z}}
$$
The method approximates this by ignoring the second term (the residual term $e$):
$$
\frac{\partial d'}{\partial \mathbf{z}} \approx \alpha \sum_{i=0}^{N-1} \frac{\partial f_\theta(\mathbf{p}^{(i)}(\mathbf{z}_0), \mathbf{z})}{\partial \mathbf{z}}
$$
Where:
- $d'$ is the main part of the ray distance.
- $\mathbf{p}^{(i)}$ are points computed using fixed $\mathbf{z}_0$.
- The second term $\frac{\partial f}{\partial \mathbf{p}} \frac{\partial \mathbf{p}}{\partial \mathbf{z}}$ is ignored [S1].

**Parameters:**
- $\mathbf{z}_0$: Initialization latent code or fixed path reference.
- $N$: Number of marching steps.

**Pitfalls:**
- The residual term $e$ from the full chain rule is ignored. This is an approximation that assumes high-order gradients have less impact on the optimization process [S1].
- Precision issues may arise if the marching steps are too coarse, as the gradient signal relies on the direct network output rather than the geometric consistency of the path [S1].

## Subtopics

### Efficient Gradient Computation without Computational Graph
This variant computes gradients for material and light parameters by avoiding the storage of a computational graph during the forward pass, significantly reducing memory footprint [S2, unofficial]. It distinguishes itself from standard reverse-mode automatic differentiation by casting rays with gradients from the camera during a separate backpropagation pass, rather than storing intermediate values for a single forward pass [S2, unofficial].

## Operating characteristics
- **Memory Complexity**: Significantly reduced compared to exact back-propagation because the dependency of sample points on $\mathbf{z}$ is not tracked in the computational graph [S1].
- **Time Complexity**: Similar to forward sphere tracing, but avoids the overhead of storing intermediate gradients for spatial dependencies [S1].
- **Strengths**: Enables optimization of latent codes for large resolutions (e.g., 640x480) where exact methods would exceed memory limits [S1].
- **Limitations**: The approximation may lead to suboptimal convergence if the high-order terms are significant. It is less accurate than exact gradient methods [S1].

## Background
The method was introduced in the context of rendering Deep Implicit Signed Distance Functions (DeepSDF) with differentiable sphere tracing. It addresses the memory bottleneck of back-propagating through the marching process in large-scale 3D shape representations [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
