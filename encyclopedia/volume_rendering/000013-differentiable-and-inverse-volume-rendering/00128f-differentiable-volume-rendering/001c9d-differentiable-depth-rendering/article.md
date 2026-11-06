# Differentiable Depth Rendering

Also called: Differentiable Depth Rendering via Implicit Function Theorem, IFT-based gradients, Differentiable Depth via Implicit Function Theorem (IFT) Surrogate, IFT Surrogate, IFT-based differentiable rendering, surrogate gradient for depth

## Overview
Differentiable Depth Rendering computes exact gradients of rendered depth values with respect to implicit function parameters by applying the Implicit Function Theorem (IFT) [S2, unofficial]. It solves the problem of numerical instability and high memory cost associated with differentiating through iterative root-finding algorithms (such as bisection or sphere tracing) directly [S2, unofficial]. Unlike standard reverse-mode automatic differentiation which backpropagates through the entire trace history, this method uses a surrogate expression for the backward pass that yields the correct analytical derivative while allowing the forward pass to remain a robust, non-differentiable numerical trace [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Ray origins $\mathbf{o} \in \mathbb{R}^3$ and directions $\mathbf{d} \in \mathbb{R}^3$ for each pixel.
    *   An implicit function $f(\mathbf{x}, \boldsymbol{\theta})$ mapping 3D coordinates and parameters $\boldsymbol{\theta}$ to a scalar value (e.g., signed distance), where $f(\mathbf{x}, \boldsymbol{\theta}) = 0$ defines the surface [S2, unofficial].
    *   A forward tracing result $t_{\text{det}}$ (the depth along the ray) obtained via a robust numerical method (e.g., bisection) [S2, unofficial].
*   **Outputs:**
    *   A depth tensor containing the surrogate depth values $t_{\text{surr}}$ which match $t_{\text{det}}$ in value but carry correct gradients via autograd [S2, unofficial].

**Data Structures:**
*   `hit_idx`: Integer array of indices corresponding to pixels where a surface intersection was found [S2, unofficial].
*   `t_det`: Float array of forward-detected depths, detached from the computation graph [S2, unofficial].
*   `p_star`: Float tensor of shape $(N, 3)$ containing the 3D surface points $\mathbf{p}^* = \mathbf{o} + t_{\text{det}}\mathbf{d}$ for hit pixels, requiring gradient tracking [S2, unofficial].
*   `f_vals`: Float tensor of shape $(N,)$ containing $f(\mathbf{p}^*, \boldsymbol{\theta})$ [S2, unofficial].
*   `grad_x`: Gradient tensor of shape $(N, 3)$ containing $\nabla_{\mathbf{x}} f$ evaluated at $\mathbf{p}^*$ [S2, unofficial].

**Parameters:**
*   `grad_clamp`: Minimum absolute value for the directional derivative to prevent division by zero (grazing rays). Default $10^{-6}$ [S2, unofficial].
*   `dtype`: Data type for tensors; `float64` is recommended for numerical stability [S2, unofficial].

**Steps:**
1.  **Forward Pass (Non-Differentiable):** Use a robust numerical algorithm (e.g., bisection) to find the root $t$ such that $f(\mathbf{o} + t\mathbf{d}, \boldsymbol{\theta}) = 0$. Store the result as $t_{\text{det}}$ detached from the autograd graph [S2, unofficial].
2.  **Identify Hits:** Extract indices of pixels where the surface was hit: `hit_idx = np.where(hit_mask)[0]` [S2, unofficial].
3.  **Compute Surface Points:** Calculate the 3D coordinates of the intersection points:
    $$ \mathbf{p}^* = \mathbf{o}_{\text{hit}} + t_{\text{det, hit}} \cdot \mathbf{d}_{\text{hit}} $$
    Convert $\mathbf{p}^*$ to a PyTorch tensor with `requires_grad=True` [S2, unofficial].
4.  **Evaluate Function and Gradient:**
    *   Evaluate the implicit function at the surface points: $f_{\text{vals}} = f(\mathbf{p}^*, \boldsymbol{\theta})$. Note: Ideally $f_{\text{vals}} \approx 0$, but numerical errors may exist [S2, unofficial].
    *   Compute the spatial gradient $\nabla_{\mathbf{x}} f$ using automatic differentiation:
        $$ (\nabla_{\mathbf{x}} f) = \frac{\partial f(\mathbf{p}^*, \boldsymbol{\theta})}{\partial \mathbf{p}^*} $$
        This is done via `torch.autograd.grad(f_vals.sum(), p_star, create_graph=True)` [S2, unofficial].
5.  **Compute Directional Derivative:** Calculate the dot product of the spatial gradient and the ray direction:
    $$ \text{dot} = \nabla_{\mathbf{x}} f \cdot \mathbf{d} $$
    This represents $\frac{\partial f}{\partial t}$ [S2, unofficial].
6.  **Clamp Directional Derivative:** Prevent division by zero for grazing rays where $\nabla_{\mathbf{x}} f \perp \mathbf{d}$:
    $$ \text{dot}_{\text{clamped}} = \begin{cases} \text{grad\_clamp} & \text{if } \text{dot} \ge 0 \text{ and } |\text{dot}| < \text{grad\_clamp} \\ -\text{grad\_clamp} & \text{if } \text{dot} < 0 \text{ and } |\text{dot}| < \text{grad\_clamp} \\ \text{dot} & \text{otherwise} \end{cases} $$
    [S2, unofficial].
7.  **Compute Surrogate Depth:** Apply the IFT formula to create the differentiable surrogate:
    $$ t_{\text{surr}} = t_{\text{det}} - \frac{f_{\text{vals}}}{\text{dot}_{\text{clamped}}} $$
    [S2, unofficial].
8.  **Scatter Results:** Scatter $t_{\text{surr}}$ back into the full depth tensor at `hit_idx`. Non-hit pixels retain background values [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def differentiable_depth_rendering(
    origins: torch.Tensor,
    directions: torch.Tensor,
    hit_mask: torch.Tensor,
    t_det: torch.Tensor,
    implicit_func,
    grad_clamp: float = 1e-6
) -> torch.Tensor:
    """
    Compute differentiable depth using IFT surrogate.
    
    Args:
        origins: (H, W, 3) ray origins.
        directions: (H, W, 3) ray directions.
        hit_mask: (H, W) boolean mask of pixels hitting the surface.
        t_det: (H, W) forward-detected depths (detached from graph).
        implicit_func: Callable f(p, theta) -> value.
        grad_clamp: Clamping threshold for directional derivative.
        
    Returns:
        depth: (H, W) tensor with IFT gradients.
    """
    # 1. Identify hit pixels
    hit_indices = torch.nonzero(hit_mask, as_tuple=False).squeeze()
    if hit_indices.dim() == 0:
        hit_indices = hit_indices.unsqueeze(0)
        
    if hit_indices.numel() == 0:
        return t_det.detach()

    # 2. Gather ray data for hit pixels
    # Note: Assuming origins/directions are (H, W, 3)
    # Reshape to (N, 3) for gathering
    H, W, _ = origins.shape
    flat_origins = origins.view(-1, 3)
    flat_dirs = directions.view(-1, 3)
    flat_t_det = t_det.view(-1)
    
    # Convert indices to flat indices
    flat_hit_indices = hit_indices[:, 0] * W + hit_indices[:, 1]
    
    o_hit = flat_origins[flat_hit_indices]
    d_hit = flat_dirs[flat_hit_indices]
    t_hit = flat_t_det[flat_hit_indices]

    # 3. Compute surface points p*
    # p* = o + t * d
    p_star = o_hit + t_hit.unsqueeze(1) * d_hit
    p_star.requires_grad_(True)

    # 4. Evaluate f(p*) and compute gradient
    # implicit_func should return a scalar or tensor compatible with autograd
    f_vals = implicit_func(p_star) 
    
    # Compute spatial gradient grad_x = df/dp*
    # We sum f_vals to create a scalar loss for grad
    (grad_x,) = torch.autograd.grad(f_vals.sum(), p_star, create_graph=True)

    # 5. Compute directional derivative dot = grad_x . d
    dot = (grad_x * d_hit).sum(dim=1)

    # 6. Clamp directional derivative
    abs_dot = torch.abs(dot)
    dot_clamped = torch.where(
        abs_dot < grad_clamp,
        torch.where(dot >= 0, torch.tensor(grad_clamp, device=dot.device), 
                    torch.tensor(-grad_clamp, device=dot.device)),
        dot
    )

    # 7. Compute IFT surrogate depth
    # t_surr = t_det - f(p*) / (df/dt)
    t_surr = t_hit - f_vals / dot_clamped

    # 8. Scatter back to full tensor
    depth = t_det.detach().clone().view(-1)
    depth[flat_hit_indices] = t_surr
    
    return depth.view(H, W)
```

**Edge Cases and Pitfalls:**
*   **Grazing Rays:** If the ray direction is perpendicular to the surface normal, $\nabla_{\mathbf{x}} f \cdot \mathbf{d} \approx 0$, causing division by zero. Clamping `dot` is required [S2, unofficial].
*   **Numerical Residuals:** The forward bisection may not find the exact root, leaving $f(\mathbf{p}^*) \neq 0$. The IFT formula corrects this residual in the gradient computation [S2, unofficial].
*   **Precision:** Use `float64` for intermediate calculations to maintain stability, especially for small gradients [S2, unofficial].
*   **Non-Implementation in Core Loop:** Some libraries implement the forward pass in NumPy for robustness and only provide the PyTorch IFT wrapper for the backward pass, requiring careful integration [S2, unofficial].

## Subtopics

### [Depth from Ray Distance](5665-depth-from-ray-distance.md)
Computes Euclidean distance from camera center to intersection, converting ray distance to camera-space depth ($z_c$). It provides exact gradients via IFT, avoiding the instability of differentiating through iterative bisection directly.

### [Approximated Gradient Back-Propagation](5666-approximated-gradient-back-propagation.md)
Reduces memory in differentiable sphere tracing by ignoring high-order gradient terms (dependence of sample points on latent code). It sacrifices gradient accuracy for lower memory overhead compared to the exact IFT method.

## Operating characteristics
*   **Time Complexity:** The forward pass is dominated by the numerical root-finding (e.g., bisection), which is $O(k)$ per ray where $k$ is the number of steps. The backward pass is $O(1)$ per hit pixel, involving only function evaluation and gradient computation at the surface point.
*   **Memory Complexity:** $O(n)$ where $n$ is the number of pixels, as it does not store intermediate states for every marching step, unlike standard reverse-mode AD which requires $O(kn)$ [S5, unofficial].
*   **Strengths:** Exact gradients; robust forward pass (can use non-differentiable algorithms); low memory usage.
*   **Limitations:** Requires the implicit function to be differentiable; sensitive to clamping parameters for grazing rays; requires the surface to be well-defined ($f=0$).
*   **When to Use:** When optimizing implicit function parameters (e.g., neural fields, SDFs) and exact depth gradients are needed without the memory cost of full backpropagation through the tracer.
*   **When Not to Use:** When the implicit function is not differentiable or when approximate gradients are sufficient and memory is extremely constrained (though IFT is already memory efficient).

## Background
The method was introduced as part of the `isoray` library, providing PyTorch integration for robust iso-surface rendering [S2, unofficial]. It leverages the Implicit Function Theorem to decouple the robustness of numerical root-finding from the differentiability required for optimization [S2, unofficial]. This approach addresses the limitations of end-to-end differentiable rendering that backpropagates through bisection, which can be numerically unstable and memory-intensive [S2, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
