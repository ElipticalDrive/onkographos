# Approximated Gradient Back-Propagation


## Overview
Approximated Gradient Back-Propagation is a method for computing gradients in differentiable volume rendering by ignoring high-order gradient terms, specifically the dependence of point positions on the latent code, to reduce memory usage [S1]. It sets itself apart from exact gradient methods by using only the direct gradient of the Signed Distance Function (SDF) network, making it particularly useful for optimizing latent codes in implicit shape representations like DeepSDF [S1].

## How it works
**Inputs and Outputs:**
- Input: A latent code $\mathbf{z}$, camera ray origin $\mathbf{p}^{(0)}$, and SDF network parameters $\theta$.
- Output: Gradients $\frac{\partial L}{\partial \mathbf{z}}$ for a loss function $L$.

**Data Structures:**
- $\mathbf{z}$: Latent code vector.
- $\mathbf{p}^{(i)}$: Sample points along the ray, computed using a fixed initialization $\mathbf{z}_0$.
- $f_\theta(\mathbf{p}, \mathbf{z})$: SDF network evaluating the distance at point $\mathbf{p}$ given latent code $\mathbf{z}$.

**Steps:**
1. **Initialization:** Initialize the first point $\mathbf{p}^{(0)}$ as the intersection between the camera ray and the unit sphere. If the ray does not intersect, mark as background [S1].
2. **Forward Marching:** Perform sphere tracing to compute sample points $\mathbf{p}^{(i)}$ along the ray. Crucially, these points are computed using a *fixed* latent code $\mathbf{z}_0$ (e.g., from a previous iteration or initialization), not the current $\mathbf{z}$ being optimized [S1].
3. **SDF Evaluation:** Evaluate the SDF network $f_\theta(\mathbf{p}^{(i)}(\mathbf{z}_0), \mathbf{z})$ for the current $\mathbf{z}$ at the fixed points [S1].
4. **Gradient Approximation:** Compute the gradient of the rendered distance $d'$ with respect to $\mathbf{z}$ by ignoring the residual term involving $\frac{\partial \mathbf{p}^{(i)}}{\partial \mathbf{z}}$ [S1].

**Formula:**
The full chain rule for the gradient of the SDF output with respect to the latent code is:
$$
\frac{\partial f_{\theta}(\mathbf{p}^{(i)}(\mathbf{z}), \mathbf{z})}{\partial \mathbf{z}} = \frac{\partial f_{\theta}(\mathbf{p}^{(i)}(\mathbf{z}_0), \mathbf{z})}{\partial \mathbf{z}} + \frac{\partial f_{\theta}(\mathbf{p}^{(i)}(\mathbf{z}), \mathbf{z}_0)}{\partial \mathbf{p}^{(i)}(\mathbf{z})} \frac{\partial \mathbf{p}^{(i)}(\mathbf{z}_0)}{\partial \mathbf{z}}
$$
The method approximates this by keeping only the first term [S1]:
$$
\frac{\partial d'}{\partial \mathbf{z}} \approx \alpha \sum_{i=0}^{N-1} \frac{\partial f_{\theta}(\mathbf{p}^{(i)}(\mathbf{z}_0), \mathbf{z})}{\partial \mathbf{z}}
$$
where $\alpha$ is the transmittance/weight factor and $N$ is the number of samples [S1].

**Parameters:**
- $\mathbf{z}_0$: Fixed latent code used for point position calculation. Not given in the sources.
- $\alpha$: Weighting factor for samples. Not given in the sources.

**Pitfalls:**
- The residual term $e$ (the second term in the full chain rule) is ignored. High-order gradients empirically have less impact, but this approximation may fail if point positions are highly sensitive to $\mathbf{z}$ [S1].
- Precision issues may arise if the SDF values are not well-conditioned near the surface [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def approximated_gradient_backprop(z, z0, ray_origin, ray_direction, sdf_network, num_samples):
    """
    Computes approximate gradient of rendered distance w.r.t. latent code z.
    """
    # 1. Initialize points using fixed z0
    # Intersection with unit sphere
    # p[0] = intersection(ray_origin, ray_direction, unit_sphere)
    # If no intersection, return gradient 0 or handle as background
    
    points = []
    current_pos = ray_origin
    for i in range(num_samples):
        # Sphere tracing step using fixed z0
        dist = sdf_network(current_pos, z0)
        current_pos = current_pos + dist * ray_direction
        points.append(current_pos)
    
    # 2. Evaluate SDF and gradients for current z
    grad_z = torch.zeros_like(z)
    total_weight = 0.0
    
    for p in points:
        # Forward pass: evaluate SDF with current z
        sdf_val = sdf_network(p, z)
        
        # Backward pass: compute gradient w.r.t. z only
        # Ignore gradient w.r.t. p (which depends on z0)
        sdf_val.backward(gradient=torch.tensor(1.0), retain_graph=True)
        
        # Accumulate gradient
        # Note: In practice, weights alpha are applied here
        # grad_z += alpha * z.grad
        # Clear gradient for next iteration
        z.grad.zero_()
        
    return grad_z
```
*Note: The pseudocode above is a simplified representation. The actual implementation requires careful handling of PyTorch's computational graph and weight accumulation $\alpha$ [S1].*

## Subtopics

### [Efficient Gradient Computation without Computational Graph](6799-efficient-gradient-computation-without-computational-graph.md)
This method computes gradients for material and light parameters in differentiable volume rendering by avoiding the storage of a computational graph during the forward pass, thereby significantly reducing memory footprint. It distinguishes itself from standard reverse-mode automatic differentiation by casting rays with gradients from the camera during a separate backpropagation pass, rather than storing intermediate values for a single forward pass.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples $N$ and the cost of the SDF network evaluation [S1].
- **Memory Complexity:** Reduced compared to exact backpropagation because the computational graph for point positions is not stored [S1].
- **Strengths:** Enables optimization of latent codes in memory-constrained environments; bridges the gap between shape priors and partial observations [S1].
- **Limitations:** Approximate gradients may lead to suboptimal convergence if the residual term is significant [S1].
- **When to use:** When optimizing latent codes for implicit shapes (e.g., DeepSDF) and memory is a bottleneck [S1].
- **When not to use:** When exact gradients are required for high-precision surface reconstruction [S1].

## Background
The method originates from the paper "DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing" [S1]. It was developed to address the memory limitations of differentiable sphere tracing when optimizing latent codes for 3D shape representation [S1]. The approach leverages the observation that high-order gradients have less impact on the optimization process, allowing for a simplified gradient computation [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
