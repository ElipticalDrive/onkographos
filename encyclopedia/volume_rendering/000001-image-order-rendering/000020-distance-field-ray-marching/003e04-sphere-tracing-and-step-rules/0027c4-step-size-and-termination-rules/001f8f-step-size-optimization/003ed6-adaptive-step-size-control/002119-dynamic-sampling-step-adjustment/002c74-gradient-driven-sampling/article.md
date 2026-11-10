# Gradient-driven Sampling

Also called: Gradient-based Step Size Adjustment

## Overview
Gradient-driven Sampling is a dynamic step size adjustment technique for distance-field ray marching that refines traversal steps according to the magnitude of the scalar field gradient $|\nabla s|$. It solves the problem of balancing rendering accuracy against computational cost in volumes with sharp boundaries, such as CT angiography data, by taking smaller steps in high-gradient regions to capture transitions accurately and larger steps in smooth interiors. It sets itself apart from uniform sampling by adapting to local feature sharpness, achieving 2-3x efficiency gains without aliasing artifacts [S1].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
- Scalar field volume $V$ with associated gradient field $\nabla V$.
- Minimum step size $h_{min}$ and maximum step size $h_{max}$.
- Gradient threshold $\tau$ (controls sensitivity to feature sharpness).

**Outputs:**
- Sequence of sample points along the ray for compositing.

**Data Structures:**
- The scalar field $V$ is typically stored as a 3D texture or grid.
- The gradient field $\nabla V$ is stored as a 3D vector field, where each voxel contains a 3D vector $(\frac{\partial s}{\partial x}, \frac{\partial s}{\partial y}, \frac{\partial s}{\partial z})$.
- If the gradient field is not pre-computed, it is derived from the scalar field using central differences or analytical derivatives if available.

**Steps:**
1. **Initialize:** Start at ray origin $\mathbf{p} = \mathbf{o}$. Set current step size $h = h_{max}$.
2. **Sample Gradient:** At current position $\mathbf{p}$, sample the scalar value $s$ and the gradient vector $\mathbf{g} = \nabla V(\mathbf{p})$.
3. **Compute Gradient Magnitude:** Calculate $|\mathbf{g}| = \sqrt{g_x^2 + g_y^2 + g_z^2}$.
4. **Adjust Step Size:**
   - If $|\mathbf{g}| > \tau$, the region is considered high-gradient (sharp feature). Reduce step size: $h = \max(h_{min}, h_{max} \cdot \frac{\tau}{|\mathbf{g}|})$. [G: This inverse proportionality ensures smaller steps for larger gradients, though the exact scaling factor may vary by implementation. The source states steps are "dynamically adjusted based on the gradient direction and a threshold," implying the magnitude drives the size.]
   - If $|\mathbf{g}| \le \tau$, the region is smooth. Increase step size towards $h_{max}$: $h = \min(h_{max}, h + \Delta h)$ or simply $h = h_{max}$. [G: A simple clamp to $h_{max}$ is sufficient if the previous step was small, or a gradual increase can be used. The source emphasizes bounding by min/max.]
5. **Bound Step Size:** Ensure $h_{min} \le h \le h_{max}$.
6. **Advance Ray:** Move along the ray direction: $\mathbf{p}_{new} = \mathbf{p} + h \cdot \mathbf{d}$.
7. **Sample and Composite:** Sample the scalar value at $\mathbf{p}_{new}$. Perform compositing (e.g., front-to-back) using the accumulated opacity and color.
8. **Repeat:** Update $\mathbf{p} = \mathbf{p}_{new}$. Repeat steps 2-7 until the ray exits the volume or a termination condition is met.

**Edge Cases and Pitfalls:**
- **Zero Gradient:** In homogeneous regions, $|\mathbf{g}| = 0$. The step size should default to $h_{max}$ to maximize efficiency.
- **Discontinuities:** Sharp discontinuities in the scalar field (e.g., hard edges) can cause large gradient magnitudes. Ensure $h_{min}$ is small enough to prevent missing thin structures.
- **Gradient Noise:** Noisy scalar fields can produce erratic gradient magnitudes, leading to unstable step sizes. Smoothing the scalar field or gradient field may be necessary.
- **Precision:** Floating-point precision errors in gradient calculation can affect step size stability. Use consistent interpolation methods for sampling $V$ and $\nabla V$.

## Subtopics

### [Dynamic Sampling Strategy](9081-dynamic-sampling-strategy.md)
A procedure to dynamically determine the number and locations of sample points in a participating medium to accurately reproduce shading variations and minimize rendering errors. It adapts sample locations based on shading error, unlike prior methods that sample based on density distribution. It builds on [Gradient-driven Sampling](5281-gradient-driven-sampling.md) by using a specific gradient-based error metric to drive a recursive splitting process.

### [FTLE-based Quad Size Estimation](12446-ftle-based-quad-size-estimation.md)
A method to determine conservative quad sizes for isosurface rendering that guarantees complete covering of isosurfaces by estimating the separation factor between seeds using Finite-Time Lyapunov Exponent (FTLE) concepts from dynamical systems theory. Unlike standard FTLE which measures predictability over a fixed time interval, this method adapts the separation metric to the distance along gradient field lines to the target isosurface, ensuring quads are large enough to cover the surface but small enough to avoid gaps.

## Operating characteristics
- **Time Complexity:** Adaptive. In smooth regions, complexity approaches $O(1)$ per unit distance (large steps). In high-gradient regions, complexity increases to $O(N)$ where $N$ is the number of samples required to resolve the feature. Overall, it is more efficient than uniform sampling for volumes with sparse opaque features.
- **Memory Complexity:** Requires storage for both the scalar field and the gradient field, doubling the memory requirement compared to scalar-only methods.
- **Measured Performance:** Achieves efficiency gains of 2-3x over uniform sampling in boundary-heavy volumes like CT angiography data [S1].
- **Strengths:** Excellent detail preservation in boundary-heavy volumes; reduces aliasing artifacts; efficient in smooth interiors.
- **Limitations:** Requires pre-computation or on-the-fly computation of gradients, increasing memory and computational overhead per sample; sensitive to gradient noise.
- **When to Use:** Volumes with sharp boundaries and sparse opaque features (e.g., medical scans, CT angiography).
- **When Not to Use:** Homogeneous volumes where uniform sampling is sufficient; memory-constrained environments where storing gradient fields is prohibitive.

## Background
Gradient-driven sampling is a technique used in volume ray casting to optimize the sampling rate along rays based on the local structure of the scalar field. It is particularly useful for datasets with sparse opaque features, such as medical scans, where it can reduce rendering time by factors of 1.3 to 2.2 [S1]. The method enhances detail preservation in boundary-heavy volumes while coarsening traversal in smooth interiors [S1].

## Sources
[S1] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
