# Adjoint Method > Gradient w.r.t Ray Direction

Also called: Gradient of Projection Weight w.r.t Ray Direction, xrt_ad_n (Geometry Gradient w.r.t Ray Direction)

## Overview
This method computes the gradient of the projection weight $P$ with respect to the ray direction $\mathbf{n}$, denoted as $\partial P / \partial \mathbf{n}$. It solves the problem of backpropagating gradients through the geometric ray tracing step in differentiable volume rendering, specifically accounting for the implicit dependence of the entry point $\mathbf{p}_a$ on the ray direction $\mathbf{n}$. It builds on [Adjoint Method](789-adjoint-method.md) by handling the geometric parameter space. Unlike [Inversion Trick](964-inversion-trick.md) which may approximate geometric sensitivities, this method computes the exact geometric gradient required for the adjoint pass.

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - `ray_t`: Array of ray origins and t-values (shape `[L, D]` or similar, where $L$ is the number of rays/segments and $D$ is dimensionality 2 or 3).
  - `ray_n`: Array of ray directions (shape `[L, D]`).
  - `knot_spec`: Specification of the grid structure (start, step, num).
  - `data`: The scalar field data (e.g., density) stored in the grid.
  - `order`: Interpolation order (0, 1, or 2).
  - `buffer`: Optional pre-allocated output array of size $L$.
- **Outputs:**
  - `proj`: Array of projection values $\mathbf{xrt}[f] \in \mathbb{R}$ and their gradients with respect to $\mathbf{n}$.

**Data Structures:**
- `ray_t`: 2D array of floats.
- `ray_n`: 1D or 2D array of floats representing direction vectors.
- `knot_spec`: Contains `start` (offset), `step` (spacing), and `num` (count) arrays.
- `bbox_ll`, `bbox_ur`: Bounding box limits derived from knot spec.

**Numbered Steps:**
1. **Initialization:** Determine the dimensionality $D$ from `ray_t`. Validate that $D \in \{2, 3\}$. Calculate the number of rays $L$ as $\max(\text{ray\_t.shape}[1], \text{ray\_n.shape}[1])$. Initialize `buffer` if not provided.
2. **Bounding Box Calculation:** Compute the lower-left (`bbox_ll`) and upper-right (`bbox_ur`) bounds of the grid using:
   $$ \text{bbox\_ll} = \text{knot\_start} - (\text{knot\_step} / 2) $$
   $$ \text{bbox\_ur} = \text{bbox\_ll} + (\text{knot\_num} \times \text{knot\_step}) $$
3. **Ray-Grid Intersection:** For each ray, determine the entry point $\mathbf{p}_a$ and exit point. This involves intersecting the ray $\mathbf{o} + t\mathbf{n}$ with the bounding box. The entry point $\mathbf{p}_a$ is a function of $\mathbf{n}$.
4. **Gradient Accumulation:** Compute the total gradient of the projection weight $P$ with respect to the ray direction $\mathbf{n}$ at index `idx`. The gradient accounts for two terms:
   - The direct gradient of the projection with respect to $\mathbf{n}$ (if the projection depends explicitly on direction, though typically it depends on the path length and density along the path).
   - The indirect gradient via the entry point $\mathbf{p}_a$.
   
   The formula is:
   $$ \frac{\partial P}{\partial \mathbf{n}_{idx}} = \text{ray\_n.grad}[idx] + \text{fac\_t} \times \mathbf{p}_a.\text{grad}[idx] $$
   Where:
   - `ray_n.grad[idx]` is the direct gradient component.
   - `fac_t` is a scalar factor accounting for the implicit dependence of the entry point $\mathbf{p}_a$ on $\mathbf{n}$. This factor arises from the chain rule when $\mathbf{p}_a$ changes as $\mathbf{n}$ changes.
   - $\mathbf{p}_a.\text{grad}[idx]$ is the gradient of the entry point position with respect to $\mathbf{n}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_wrt_ray_direction(ray_t, ray_n, knot_spec, data, order, buffer=None):
    # 1. Setup
    D = ray_t.shape[1] # Assuming 2D or 3D
    L = max(ray_t.shape[0], ray_n.shape[0])
    if buffer is None:
        buffer = zeros(L)
    
    # 2. Bounding Box
    knot_start = knot_spec.start
    knot_step = knot_spec.step
    knot_num = knot_spec.num
    bbox_ll = knot_start - (knot_step / 2)
    bbox_ur = bbox_ll + (knot_num * knot_step)
    
    # 3. Forward Pass (Implicit in the gradient calculation context)
    # Calculate entry point p_a for each ray
    # p_a is the intersection of ray(o, n) with bbox_ll/ur
    # This step determines p_a(n)
    
    # 4. Backward Pass / Gradient Calculation
    # Initialize gradients
    ray_n_grad = zeros_like(ray_n)
    p_a_grad = zeros_like(ray_n) # Gradient of entry point w.r.t n
    
    # Compute direct gradient component
    # Note: In many volume rendering formulations, the projection P depends on n
    # through the path length and the density sampling.
    # The source specifies: ray_n.grad[idx] + fac_t * p_a.grad[idx]
    
    for idx in range(L):
        # Calculate fac_t: factor for implicit dependence of p_a on n
        # This requires the derivative of the intersection point w.r.t direction
        # For a box intersection, if the ray hits a face, the derivative depends on the face normal.
        fac_t = compute_fac_t(ray_n[idx], bbox_ll, bbox_ur) # Implementation detail for fac_t
        
        # Get gradients from upstream (projection loss)
        # proj_grad is the gradient of the loss w.r.t the projection value P
        # This is assumed to be propagated into ray_n.grad and p_a.grad
        
        # Accumulate total gradient
        # The source formula: ray_n.grad[idx] + fac_t * p_a.grad[idx]
        # Here, ray_n.grad[idx] represents the direct contribution
        # and fac_t * p_a.grad[idx] represents the geometric contribution
        
        buffer[idx] = ray_n_grad[idx] + fac_t * p_a_grad[idx]
        
    return buffer
```

**Edge Cases and Pitfalls:**
- **Precision:** Intersection calculations near box edges can suffer from precision issues.
- **Degenerate Input:** Rays parallel to box faces may have undefined or infinite derivatives for the entry point.
- **Boundary Conditions:** Ensure `fac_t` is correctly computed for rays entering different faces of the bounding box.

## Subtopics

### [Gradient w.r.t. Initial Velocity](23227-gradient-w-r-t-initial-velocity.md)
This method computes the gradient of the ray position $\mathbf{x}$ with respect to the initial velocity $\mathbf{v}_0$ in differentiable volume rendering using symplectic integration. It is required to propagate gradients through the ray tracing step when the refractive index or medium properties depend on parameters that influence the initial velocity. It builds on [Adjoint Method > Gradient w.r.t Ray Direction](2891-adjoint-method-gradient-w-r-t-ray-direction.md) by handling the velocity parameter space. The method distinguishes between Type-1 connections (where the ray hits an implicit surface) and Type-2 connections (where the ray connects two points inside the medium).

## Operating characteristics
- **Time Complexity:** Linear in the number of rays $L$, assuming constant-time bounding box intersection.
- **Memory Complexity:** Linear in $L$ for storing gradients and intermediate values.
- **Strengths:** Provides exact geometric gradients, essential for high-precision differentiable rendering.
- **Limitations:** Requires careful handling of intersection derivatives. Performance may degrade if the bounding box intersection logic is complex.
- **When to Use:** When differentiating through the ray tracing step itself, particularly when the entry/exit points are sensitive to ray direction.

## Background
This method is part of the `differentiable_xrt` library, which implements differentiable X-ray tomography and volume rendering. It is designed to compute gradients for inverse problems where the ray geometry is a function of parameters.

## Sources
[S1] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
