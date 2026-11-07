# Differentiable Depth Rendering

Also called: Differentiable Depth Rendering via Implicit Function Theorem, IFT-based gradients, Differentiable Depth via Implicit Function Theorem (IFT) Surrogate, IFT Surrogate, IFT-based differentiable rendering, surrogate gradient for depth

## Overview
Differentiable Depth Rendering via the Implicit Function Theorem (IFT) computes exact gradients of rendered depth values with respect to implicit scene parameters (e.g., neural network weights or latent codes) without differentiating through the forward ray-marching or bisection algorithm [S2, unofficial]. It solves the problem of high memory usage and instability associated with backpropagating through iterative solvers by replacing the backward pass with an analytic surrogate derived from the implicit surface equation $f(\mathbf{x}, \boldsymbol{\theta}) = 0$ [S2, unofficial]. Unlike [Differentiable Direct Volume Rendering (DVR)](8871-differentiable-direct-volume-rendering-dvr.md) which integrates emission-absorption, this method targets iso-surface extraction where depth is defined by the zero-crossing of a Signed Distance Function (SDF) [S1][S2, unofficial].

## How it works
The method separates the forward pass (robust, non-differentiable tracing) from the backward pass (analytic gradient computation via IFT).

### Inputs and Outputs
*   **Inputs:**
    *   Ray origins $\mathbf{o} \in \mathbb{R}^3$ and directions $\mathbf{d} \in \mathbb{R}^3$ (normalized).
    *   Implicit function $f(\mathbf{x}; \boldsymbol{\theta})$ returning value and optionally gradient, parameterized by $\boldsymbol{\theta}$.
    *   Forward depth estimate $t_{\text{det}}$ obtained via a robust solver (e.g., bisection or sphere tracing).
*   **Outputs:**
    *   Differentiable depth tensor $\hat{t}$ compatible with autograd frameworks.
    *   Gradients $\frac{\partial \hat{t}}{\partial \boldsymbol{\theta}}$ computed during the backward pass.

### Data Structures
*   **Hit Mask:** Boolean array identifying pixels where the ray intersects the surface (SDF value crosses zero).
*   **Surface Points:** $\mathbf{p}^* = \mathbf{o} + t_{\text{det}} \mathbf{d}$, the 3D coordinates of the intersection.
*   **Tensors:** PyTorch tensors with `requires_grad=True` for $\mathbf{p}^*$ to enable automatic differentiation of $f(\mathbf{p}^*)$ [S2, unofficial].

### Algorithm Steps
1.  **Forward Tracing (Detached):**
    Compute the intersection distance $t_{\text{det}}$ using a standard, non-differentiable algorithm (e.g., bisection or sphere tracing). This value is detached from the computation graph to prevent backpropagation through the solver [S2, unofficial].
    $$ t_{\text{det}} = \text{Solver}(\mathbf{o}, \mathbf{d}, f) $$

2.  **Surface Point Evaluation:**
    Calculate the 3D position of the surface intersection:
    $$ \mathbf{p}^* = \mathbf{o} + t_{\text{det}} \mathbf{d} $$
    Convert $\mathbf{p}^*$ to a tensor requiring gradients.

3.  **Implicit Function Evaluation:**
    Evaluate the implicit function and its spatial gradient at $\mathbf{p}^*$:
    $$ v = f(\mathbf{p}^*; \boldsymbol{\theta}) $$
    $$ \nabla_{\mathbf{x}} f = \nabla_{\mathbf{x}} f(\mathbf{p}^*; \boldsymbol{\theta}) $$
    Note: Due to the solver, $v$ is approximately zero but may have small residual error.

4.  **IFT Gradient Computation:**
    According to the Implicit Function Theorem, for $f(\mathbf{o} + t\mathbf{d}, \boldsymbol{\theta}) = 0$, the derivative of $t$ with respect to parameters $\boldsymbol{\theta}$ is:
    $$ \frac{dt}{d\boldsymbol{\theta}} = -\frac{\frac{\partial f}{\partial \boldsymbol{\theta}}}{\nabla_{\mathbf{x}} f \cdot \mathbf{d}} $$
    In practice, using automatic differentiation on the scalar field value $v$:
    $$ \text{grad}_x = \nabla_{\mathbf{p}^*} v $$
    Compute the directional derivative (denominator):
    $$ \text{dot} = \text{grad}_x \cdot \mathbf{d} $$

5.  **Clamping:**
    To prevent division by zero for grazing rays ($\nabla_{\mathbf{x}} f \perp \mathbf{d}$), clamp the directional derivative:
    $$ \text{dot}_{\text{clamped}} = \begin{cases} \text{grad\_clamp} & \text{if } \text{dot} \ge 0 \text{ and } |\text{dot}| < \text{grad\_clamp} \\ -\text{grad\_clamp} & \text{if } \text{dot} < 0 \text{ and } |\text{dot}| < \text{grad\_clamp} \\ \text{dot} & \text{otherwise} \end{cases} $$
    Typical value for `grad_clamp` is $10^{-6}$ [S2, unofficial].

6.  **Surrogate Depth Construction:**
    Construct the differentiable depth $\hat{t}$ using the surrogate formula. This ensures the forward value matches $t_{\text{det}}$ (approximately) but the gradient follows the IFT rule:
    $$ \hat{t} = t_{\text{det}} - \frac{v}{\text{dot}_{\text{clamped}}} $$
    Here, $t_{\text{det}}$ is detached, so $\frac{\partial \hat{t}}{\partial \boldsymbol{\theta}} = -\frac{\partial}{\partial \boldsymbol{\theta}} \left( \frac{v}{\text{dot}_{\text{clamped}}} \right)$ [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def differentiable_depth_rendering(
    origins: torch.Tensor,       # (N, 3)
    directions: torch.Tensor,    # (N, 3), normalized
    t_detached: torch.Tensor,    # (N,), forward depth from bisection/sphere tracing
    implicit_fn: callable,       # f(x, theta) -> (values, gradients_or_None)
    grad_clamp: float = 1e-6
) -> torch.Tensor:
    """
    Computes differentiable depth using IFT surrogate.
    
    Args:
        origins: Ray origins.
        directions: Ray directions.
        t_detached: Non-differentiable depth from forward solver.
        implicit_fn: Function evaluating SDF. Must support autograd.
        grad_clamp: Minimum absolute directional derivative.
        
    Returns:
        Differentiable depth tensor.
    """
    # 1. Compute surface points
    # t_detached is detached from graph to avoid backprop through solver
    p_star = origins + t_detached.unsqueeze(1) * directions
    
    # 2. Enable gradient tracking for p_star to compute grad of f
    p_star.requires_grad_(True)
    
    # 3. Evaluate implicit function
    # implicit_fn should return (values, gradients) or just values
    # If it returns gradients, use them; otherwise compute via autograd
    f_vals, f_grads = implicit_fn(p_star)
    
    # If implicit_fn does not return gradients, compute them
    if f_grads is None:
        # f_vals is (N,), sum to create scalar for backward
        (grad_x,) = torch.autograd.grad(f_vals.sum(), p_star, create_graph=True)
    else:
        grad_x = f_grads
        
    # 4. Compute directional derivative: dot = grad_x . d
    dot = torch.sum(grad_x * directions, dim=1)
    
    # 5. Clamp directional derivative to avoid division by zero
    abs_dot = torch.abs(dot)
    dot_clamped = torch.where(
        abs_dot < grad_clamp,
        torch.where(dot >= 0, grad_clamp, -grad_clamp),
        dot
    )
    
    # 6. Compute IFT surrogate depth
    # t_surr = t_det - f(p*) / dot
    # Note: t_detached is detached, so gradient flows only through f_vals and dot
    t_surr = t_detached - f_vals / dot_clamped
    
    return t_surr
```

### Parameters
*   `grad_clamp`: Minimum absolute value for the directional derivative $\nabla_{\mathbf{x}} f \cdot \mathbf{d}$. Prevents exploding gradients for grazing rays. Default: $10^{-6}$ [S2, unofficial].
*   `dtype`: Data type for tensors. `float64` is recommended for numerical stability in the division step [S2, unofficial].

### Edge Cases and Pitfalls
*   **Grazing Rays:** When the ray direction is perpendicular to the surface normal, $\nabla_{\mathbf{x}} f \cdot \mathbf{d} \approx 0$. The clamping mechanism is essential to prevent NaNs or infinite gradients [S2, unofficial].
*   **Non-Zero Residuals:** The forward solver may not find the exact zero-crossing, leaving $f(\mathbf{p}^*) \neq 0$. The surrogate formula $t_{\text{det}} - f/\text{dot}$ corrects this residual in the gradient computation, effectively linearizing the error [S2, unofficial].
*   **Solver Discontinuity:** The forward solver (e.g., bisection) may jump between local minima or branches. The IFT gradient is only valid if the surface is smooth and the solver converges to the correct branch [S1][S2, unofficial].
*   **Implementation Gap:** The core IFT logic is often implemented in a separate module (e.g., `isoray/torch/render.py`) and not always shown in the main rendering loop examples, requiring careful integration [S2, unofficial].

## Subtopics

### [Depth from Ray Distance](5665-depth-from-ray-distance.md)
Computes camera-space depth $z_c$ from the accumulated ray distance $d$. It treats ray distance as a single geometric scalar, differing from [Expected Depth Rendering](1332-differentiable-depth-rendering.md) by providing a direct projection of the ray endpoint.

### [Approximated Gradient Back-Propagation](5666-approximated-gradient-back-propagation.md)
Ignores high-order gradient terms (dependence of point positions on latent codes) to reduce memory. It uses only the direct SDF network gradient, suitable for optimizing latent codes in implicit shapes like DeepSDF.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, dominated by the implicit function evaluation and gradient computation. The backward pass is as expensive as the forward pass if gradients are computed via autograd, but avoids storing the entire trace history [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing surface points and gradients. Significantly lower than Reverse-Mode AD which requires $O(kn)$ intermediate states for $k$ samples [S5][S2, unofficial].
*   **Accuracy:** Provides exact gradients with respect to $\boldsymbol{\theta}$ assuming the implicit function is differentiable. The surrogate ensures the gradient matches the IFT derivative, correcting for solver residuals [S2, unofficial].
*   **Strengths:** Robust forward tracing (can use any solver); low memory; exact gradients for implicit parameters.
*   **Limitations:** Requires the implicit function to be differentiable; sensitive to grazing rays (requires clamping); forward solver must be stable [S2, unofficial].
*   **When to Use:** Optimizing neural implicit representations (SDFs) where memory is a constraint and exact gradients are needed for weights/latent codes [S1][S2, unofficial].
*   **When Not to Use:** When the surface is not defined by a smooth implicit function, or when the forward solver is inherently non-differentiable and cannot be decoupled [S2, unofficial].

## Background
The method originates from the need to optimize neural implicit shapes (e.g., DeepSDF) using differentiable rendering. Standard backpropagation through sphere tracing or bisection is memory-intensive and unstable [S1]. The Implicit Function Theorem provides an analytic way to compute $\frac{dt}{d\boldsymbol{\theta}}$ without differentiating the solver [S2, unofficial]. It was implemented in libraries like `isoray` to provide robust, differentiable depth rendering for implicit surfaces [S2, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
