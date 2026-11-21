# Differentiable Volume Rendering with Analytical Gradients

Also called: Analytical Differentiable Rendering, DiffRenderer

## Overview
A rendering technique that computes radiance and its analytical derivatives with respect to volume density directly, avoiding the overhead of propagating gradients through long computational graphs inherent in automatic differentiation [S1]. It solves the problem of inefficient optimization for dynamic volumetric densities, such as in fluid reconstruction, by providing a closed-form gradient calculation [S1]. Unlike [Differentiable Volume Rendering via Box Splines](2449-differentiable-volume-rendering-via-box-splines.md), which relies on basis function smoothness, this method operates on the standard emission-absorption model with explicit step-wise differentiation [S1].

## How it works
**Inputs:** Ray origin $\mathbf{x}_a$, ray direction $\boldsymbol{u}$, volume density field $\sigma(\mathbf{x})$, step size $\triangle d$, and ray exit boundary $\mathbf{x}_b$ [G].
**Outputs:** Radiance $L$ and its analytical gradient with respect to density parameters [G].

**Data Structures:**
*   Ray state: Current position $x_u$, accumulated transmittance $T$, accumulated radiance $L$ [G].
*   Step history: Lists storing $\sigma_i$, $\alpha_i$, $\mathbf{c}_i$, and $T_i$ for each sampled step [G].

**Steps:**
1.  Initialize $L \leftarrow 0$, $T \leftarrow 1$, $x_u \leftarrow \mathbf{x}_a$ [G].
2.  Iterate along the ray from $x_a$ to $x_b$ with step size $\triangle d$:
    a.  Sample density $\sigma_i$ and color $\mathbf{c}_i$ at $x_u$.
    b.  Compute step transmittance attenuation: $\alpha_i = 1 - \exp(-\sigma_i \triangle d)$.
    c.  Compute emitted radiance contribution: $C_i = T \cdot \alpha_i \cdot \mathbf{c}_i$.
    d.  Update accumulated radiance: $L \leftarrow L + C_i$.
    e.  Update transmittance: $T \leftarrow T \cdot (1 - \alpha_i)$.
    f.  Store step data $(\sigma_i, \alpha_i, \mathbf{c}_i, T_i)$ for gradient computation.
    g.  Advance ray position: $x_u \rightarrow x_u + \boldsymbol{u} \triangle d$ [S1].
3.  Compute analytical gradient $\frac{\partial L}{\partial \sigma_k}$ for each step $k$ using a backward pass:
    a.  Initialize backward accumulator $B \leftarrow 0$.
    b.  Iterate steps $k$ from $N$ down to $1$:
        i.  Direct term: $D_k = T_k \triangle d \exp(-\sigma_k \triangle d) \mathbf{c}_k$.
        ii. Indirect term: $I_k = \triangle d \cdot B$.
        iii. Gradient: $\frac{\partial L}{\partial \sigma_k} = D_k - I_k$.
        iv. Update accumulator: $B \leftarrow B (1 - \alpha_k) + T_k \alpha_k \mathbf{c}_k$.

**Formulas:**
The standard emission-absorption model computes color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as:
$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$
where $\sigma_i$ is density, $\delta_i$ is step size, $\mathbf{c}_i$ is color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is transmittance [G].

The analytical gradient with respect to density $\sigma_k$ is derived by differentiating the sum term-by-term [S1].
$$
\frac{\partial \hat{C}}{\partial \sigma_k} = T_k \triangle d \exp(-\sigma_k \triangle d) \mathbf{c}_k - \triangle d \sum_{j=k+1}^N T_j (1 - \exp(-\sigma_j \triangle d)) \mathbf{c}_j
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def analytical_gradient_rendering(ray_origin, ray_dir, density_field, step_size, ray_exit):
    x_u = ray_origin.copy()
    T = 1.0
    L = 0.0
    steps = []
    
    # Forward pass: sample and accumulate radiance
    while True:
        # Guard: check if current position is within volume bounds [G]
        if not is_inside_volume(x_u, ray_origin, ray_dir, ray_exit):
            break
            
        sigma_i = density_field.sample(x_u)
        c_i = density_field.color(x_u)
        alpha_i = 1.0 - math.exp(-sigma_i * step_size)
        
        contribution = T * alpha_i * c_i
        L += contribution
        T *= (1.0 - alpha_i)
        
        # Store history for backward gradient pass
        steps.append({'sigma': sigma_i, 'alpha': alpha_i, 'color': c_i, 'T': T / (1.0 - alpha_i) if alpha_i < 1.0 else 0.0})
        
        # Advance ray position
        x_u = x_u + ray_dir * step_size
        
    # Backward pass: compute analytical gradients
    grad_sigma = np.zeros_like(density_field.data)
    B = 0.0
    for k in range(len(steps) - 1, -1, -1):
        s = steps[k]
        direct = s['T'] * step_size * math.exp(-s['sigma'] * step_size) * s['color']
        indirect = step_size * B
        grad_sigma += direct - indirect
        B = B * (1.0 - s['alpha']) + s['T'] * s['alpha'] * s['color']
        
    return L, grad_sigma

def is_inside_volume(pos, origin, direction, exit):
    # Standard ray-box intersection guard [G]
    t_min = np.max((origin - pos) / (direction + 1e-8))
    t_max = np.min((exit - pos) / (direction + 1e-8))
    return t_min <= t_max
```

## Subtopics

### [Differentiable Moment Bounds](8855-differentiable-moment-bounds.md)
Computes continuously differentiable upper and lower bounds on cumulative mass consistent with power moments. It solves the truncated Hamburger moment problem. It is distinguished by providing robust, numerically stable bounds for differentiable rendering pipelines where gradients w.r.t. scene parameters are required.

## Operating characteristics
*   **Performance:** Improves performance for optimizing dynamic volumetric densities, such as in fluid reconstruction tasks, by avoiding gradient tracking and redundant operations [S1].
*   **Strengths:** Efficient closed-form gradient computation; scales linearly with ray steps without storing full computational graphs [G].
*   **Limitations:** Requires explicit analytical derivation of the rendering equation; less flexible than automatic differentiation for arbitrary, non-differentiable operations [G].

## Background
Originated in the context of physics-based differentiable rendering for fluid modeling [S1]. The work builds upon insights that automatic differentiation fundamentally relies on propagating gradients through long computational graphs, which carries inherent overhead compared to direct analytical derivatives [S1].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
