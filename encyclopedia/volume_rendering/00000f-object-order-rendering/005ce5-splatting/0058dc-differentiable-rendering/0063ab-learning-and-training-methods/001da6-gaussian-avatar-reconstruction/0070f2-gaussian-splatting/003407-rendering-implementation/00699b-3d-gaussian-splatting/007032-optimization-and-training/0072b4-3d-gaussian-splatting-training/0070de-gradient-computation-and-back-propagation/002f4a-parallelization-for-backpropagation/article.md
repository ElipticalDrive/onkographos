# Parallelization for Backpropagation
Also called: Per-splat Backpropagation
## Overview
Parallelization for Backpropagation is a training optimization for 3D Gaussian Splatting that changes the backward pass parallelization strategy from per-pixel to per-splat to address training speed obstacles [S1]. It derives faster, numerically equivalent solutions for gradient computation and attribute updates, yielding a 4–5× reduction in training time and lower compute and memory requirements compared to the standard approach [S1].

## How it works
The method restructures the backward pass to iterate over Gaussians rather than pixels, accumulating gradients directly per Gaussian to avoid synchronization bottlenecks [S1].

**Inputs and Outputs:**
- Input: The set of $N$ 3D Gaussians $\mathcal{G}$, camera parameters, and the gradient of the loss with respect to the rendered pixel colors $\frac{\partial \mathcal{L}}{\partial \mathbf{c}}$ [G].
- Output: The accumulated gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all Gaussian parameters $\theta$ (position, covariance, opacity, spherical harmonic coefficients) [G].

**Data Structures:**
- **Gaussian-to-Pixel Mapping:** A list or sparse structure storing the set of screen-space pixels covered by each Gaussian, generated during the forward rasterization pass [G].
- **Gradient Accumulator:** A buffer matching the parameter layout of the Gaussians, initialized to zero for each processing unit [G].

**Steps:**
1. **Forward Rasterization:** Project Gaussians to screen space and determine the set of pixels each Gaussian covers, storing the mapping for the backward pass [G].
2. **Per-Splat Iteration:** Iterate over each Gaussian $g$ in the dataset.
3. **Pixel Gradient Retrieval:** For each pixel $p$ covered by $g$, retrieve the pre-computed pixel loss gradient $\frac{\partial \mathcal{L}}{\partial \mathbf{c}_p}$ from the forward pass output [G].
4. **Jacobian Computation:** Compute the derivative of the alpha-blended color at pixel $p$ with respect to $g$'s parameters, $\frac{\partial \mathbf{c}_p}{\partial \theta_g}$, using the differentiable rendering equations [G].
5. **Gradient Accumulation:** Update the gradient accumulator for $g$ using the chain rule: $\frac{\partial \mathcal{L}}{\partial \theta_g} \leftarrow \frac{\partial \mathcal{L}}{\partial \theta_g} + \frac{\partial \mathcal{L}}{\partial \mathbf{c}_p} \frac{\partial \mathbf{c}_p}{\partial \theta_g}$ [G].
6. **Numerical Optimization:** Apply faster, numerically equivalent formulations for the Jacobian and attribute updates to reduce compute overhead [S1].

**Formulas:**
The core update for a Gaussian $g$ with parameters $\theta_g$ is:
$$ \frac{\partial \mathcal{L}}{\partial \theta_g} = \sum_{p \in \text{covered}(g)} \frac{\partial \mathcal{L}}{\partial \mathbf{c}_p} \frac{\partial \mathbf{c}_p}{\partial \theta_g} $$
Where $\frac{\partial \mathbf{c}_p}{\partial \theta_g}$ is the derivative of the alpha-blended color at pixel $p$ with respect to the Gaussian's parameters. The per-splat approach computes this sum by iterating over $g$ and then $p$, whereas the per-pixel approach iterates over $p$ and then $g$, requiring atomic adds for $\frac{\partial \mathcal{L}}{\partial \theta_g}$ when multiple Gaussians contribute to the same pixel [G].

**Parameters:**
- Bucket size: Not given in the sources.

**Edge Cases:**
- **Partial Buckets:** If $N$ is not a multiple of the processing unit size, the final iteration handles fewer Gaussians; handled gracefully by standard loop bounds [G].
- **Precision:** The method maintains numerical equivalence to the original backpropagation, so standard float32 precision is preserved [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_splat_backward(gaussians, pixel_gradients, camera_params):
    # gaussians: list of objects with 'params' dict and 'covered_pixels' list
    # pixel_gradients: 2D array of shape (H, W, 3) containing dL/dc
    # camera_params: camera intrinsics and extrinsics
    
    global_grads = []
    
    for g in gaussians:
        # Initialize local gradient accumulator for this Gaussian's parameters
        local_grad = {k: [0.0] * len(v) for k, v in g.params.items()}
        
        # Iterate over all pixels covered by this Gaussian
        for p in g.covered_pixels:
            # Retrieve pre-computed pixel loss gradient
            dL_dc_p = pixel_gradients[p.y, p.x]
            
            # Compute Jacobian: derivative of alpha-blended color w.r.t Gaussian parameters
            # Follows standard differentiable splatting chain rule through opacity, 
            # 2D covariance/conic matrix, 3D covariance, and spherical harmonics [G]
            dC_dTheta = compute_jacobian(g.params, camera_params, p)
            
            # Accumulate gradient using chain rule
            for param_name in local_grad:
                local_grad[param_name] = [a + b * dL_dc_p for a, b in zip(local_grad[param_name], dC_dTheta[param_name])]
        
        global_grads.append(local_grad)
        
    return global_grads

def compute_jacobian(params, camera_params, pixel):
    # Standard differentiable rendering Jacobian computation
    # Computes dC/dTheta via chain rule through alpha blending and SH evaluation [G]
    # Implementation matches the parent article's gradient flow for opacity, 
    # covariance, and spherical harmonics.
    pass
```

## Subtopics

### [Per-Gaussian Backward Pass](9590-per-gaussian-backward-pass.md)
The Per-Gaussian Backward Pass is a backward pass optimization for 3D Gaussian Splatting that parallelizes gradient computation over Gaussians rather than pixels to reduce atomic contention. It addresses the performance bottleneck of standard pixel-parallel backpropagation, where splats contributing to arbitrary numbers of pixels require frequent atomic operations for accumulation. By computing gradients per Gaussian instead of per tile, it reduces false sharing and atomic contention during z-order sorting.

## Operating characteristics
- **Time Complexity:** Reduces training time by 4–5× compared to the standard per-pixel backpropagation [S1].
- **Memory Complexity:** Requires lower compute and memory requirements than the standard approach [S1].
- **Strengths:** Provides faster, numerically equivalent solutions for gradient computation and attribute updates [S1].
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.

## Background
The method was introduced in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" (2024) as part of a broader effort to optimize the 3D Gaussian Splatting training pipeline [S1]. It addresses the scalability issues of the original 3DGS method, which suffered from slow training times due to the inefficiencies of the per-pixel backward pass [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
