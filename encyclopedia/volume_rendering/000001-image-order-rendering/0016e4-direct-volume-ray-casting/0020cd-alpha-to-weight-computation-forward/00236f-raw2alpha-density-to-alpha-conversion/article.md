# Raw2Alpha Density to Alpha Conversion

Also called: Raw2Alpha, density to alpha

## Overview
Raw2Alpha converts raw density values (often called "raw" or "sigma") into per-sample alpha values for direct volume ray casting. It solves the problem of mapping unbounded scalar fields to bounded opacity values suitable for front-to-back compositing. It is distinguished from its parent [Alpha-to-Weight Computation (Forward)](2373-alpha-to-weight-computation-forward.md) by performing the non-linear activation *before* weight/transmittance accumulation, whereas the parent handles the integration step.

## How it works
**Inputs:**
- `density`: Tensor of raw density values $\sigma$ (shape: $N$).
- `shift`: Scalar bias term $b$.
- `interval`: Scalar segment length $\delta$ (step size along the ray).

**Outputs:**
- `alpha`: Tensor of alpha values $\alpha$ (shape: $N$).

**Data Structures:**
- `density`: Floating point tensor.
- `alpha`: Floating point tensor.

**Formulas:**
The conversion uses a shifted softplus-like activation followed by an exponential decay based on segment length:
$$ \alpha = 1 - (1 + \exp(\sigma + b))^{-\delta} $$

The derivative required for backpropagation is:
$$ \frac{\partial \alpha}{\partial \sigma} = \delta \cdot (1 + \exp(\sigma + b))^{-(\delta + 1)} \cdot \exp(\sigma + b) $$

**Parameters:**
- `shift` ($b$): Bias. Typical values are not explicitly fixed in the source but are learned or tuned hyperparameters [S1, unofficial].
- `interval` ($\delta$): Distance between samples. Must be positive [S1, unofficial][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def raw2alpha_forward(density: torch.Tensor, shift: float, interval: float) -> torch.Tensor:
    """
    Computes alpha from raw density.
    :param density: Raw density values (sigma).
    :param shift: Bias term.
    :param interval: Segment length (delta).
    :return: Alpha values.
    """
    # Compute shifted softplus: log(1 + exp(sigma + shift))
    # Note: The formula is 1 - (1 + exp(sigma + shift))^-delta
    # We compute the base term first
    base = 1.0 + torch.exp(density + shift)
    alpha = 1.0 - torch.pow(base, -interval)
    return alpha

def raw2alpha_backward(grad_back: torch.Tensor, density: torch.Tensor, shift: float, interval: float) -> torch.Tensor:
    """
    Computes gradient of alpha w.r.t density.
    :param grad_back: Gradient from subsequent layers (dL/d_alpha).
    :param density: Raw density values (sigma).
    :param shift: Bias term.
    :param interval: Segment length (delta).
    :return: Gradient dL/d_sigma.
    """
    # Derivative: delta * (1 + exp(sigma + shift))^(-delta - 1) * exp(sigma + shift)
    exp_term = torch.exp(density + shift)
    base = 1.0 + exp_term
    
    # Compute d_alpha/d_sigma
    # Note: pow(base, -interval - 1) is equivalent to base^(-interval) / base
    d_alpha_d_sigma = interval * torch.pow(base, -interval - 1.0) * exp_term
    
    # Chain rule: dL/d_sigma = dL/d_alpha * d_alpha/d_sigma
    grad_density = grad_back * d_alpha_d_sigma
    return grad_density
```

**Edge Cases:**
- If `interval` is 0, alpha is 0 [S1, unofficial].
- Numerical stability: `exp(density + shift)` can overflow if the argument is large. In practice, `shift` is often negative to keep values in a safe range [S1, unofficial].

## Subtopics

### Post-activation Trilinear Interpolation
A variant where trilinear interpolation of raw values is performed *before* the Raw2Alpha activation, rather than after. This is used in voxel-based methods to maintain smoothness in the raw field before opacity calculation. The formula is identical to the standard Raw2Alpha but applied to the interpolated raw value $V_{interp}$: $\alpha = 1 - (1 + \exp(V_{interp} + b))^{-\delta}$ [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ for forward and backward passes, where $N$ is the number of samples. The operations are element-wise and highly parallelizable on GPU [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store intermediate `exp` and `base` terms if gradients are needed [S1, unofficial].
- **Strengths:** Differentiable, smooth transition from density to alpha, avoids hard thresholds [S1, unofficial].
- **Limitations:** Sensitive to the choice of `shift` and `interval`. Poorly chosen parameters can lead to vanishing or exploding gradients [S1, unofficial].

## Background
The method is central to Direct Voxel Grid Optimization (DVGO) and its variants like Ref-DVGO. It allows for efficient training of neural radiance fields by operating on voxel grids rather than implicit MLPs, leveraging the speed of trilinear interpolation and the differentiability of the activation function [S1, unofficial][S2].

## Sources
[S1] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S2] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
