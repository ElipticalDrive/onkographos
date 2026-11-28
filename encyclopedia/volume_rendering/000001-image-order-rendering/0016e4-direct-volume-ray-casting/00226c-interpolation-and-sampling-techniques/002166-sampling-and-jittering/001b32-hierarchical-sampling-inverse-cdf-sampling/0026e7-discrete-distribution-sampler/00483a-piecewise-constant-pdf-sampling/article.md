# Piecewise-Constant PDF Sampling

Also called: invert_cdf sampling

## Overview
Piecewise-Constant PDF Sampling generates sample coordinates from a discrete probability distribution defined by bin endpoints and logits by inverting the cumulative distribution function (CDF) [S1, S2, unofficial]. It differs from sibling methods by explicitly constructing a step-wise CDF from softmax-normalized weights and using inverse interpolation to map uniform random variables to bin centers or intervals, ensuring samples respect the learned probability mass of each bin [S1, S2, unofficial].

## How it works
**Inputs:**
*   `t`: Tensor of shape `[..., num_bins + 1]` containing sorted bin endpoint coordinates [S1, S2, unofficial].
*   `w_logits`: Tensor of shape `[..., num_bins]` containing raw logits for bin weights [S1, S2, unofficial].
*   `num_samples`: Integer specifying the number of samples to draw [S1, S2, unofficial].
*   `rng`: Optional random number generator. If `None`, deterministic sampling is used [S1, unofficial].
*   `deterministic_center`: Boolean. If `True`, deterministic samples are centered within bins; otherwise, they span the full range [S1, S2, unofficial].
*   `single_jitter`: Boolean. If `True`, all samples along a ray share the same jitter offset; if `False`, each sample gets independent jitter [S1, S2, unofficial].
*   `eps`: Machine epsilon for the dtype of `t`, used to prevent boundary issues (e.g., `torch.finfo(t.dtype).eps`) [S2, unofficial].

**Outputs:**
*   `t_samples`: Tensor of shape `[..., num_samples]` containing the sampled coordinates [S1, S2, unofficial].

**Data Structures:**
*   **CDF (`cw`)**: Cumulative sum of softmax weights, excluding the last bin, clamped to a maximum of 1. Shape `[..., num_bins]` [S2, unofficial].
*   **Full CDF (`cw0`)**: Concatenation of `0`, `cw`, and `1` along the last axis. Shape `[..., num_bins + 2]` [S2, unofficial].

**Steps:**

1.  **Compute Probabilities**: Calculate the probability density function (PDF) weights `w` by applying softmax to `w_logits` along the last axis:
    $$ w = \text{softmax}(w_{\text{logits}}, \text{dim}=-1) $$
    [S2, unofficial]

2.  **Construct CDF**: Compute the cumulative distribution function `cw` by taking the cumulative sum of the weights excluding the last bin, and clamp the maximum value to 1:
    $$ cw = \text{clamp}_{\max}(1, \text{cumsum}(w[..., :-1], \text{dim}=-1)) $$
    [S2, unofficial]

3.  **Generate Uniform Samples (`u`)**:
    *   **Deterministic (`rng` is `None`)**:
        *   If `deterministic_center` is `True`: Generate `u` using `linspace` over `[pad, 1 - pad - eps]`, where `pad = 1 / (2 * num_samples)`. This centers samples within the PDF intervals [S1, S2, unofficial].
        *   Else: Generate `u` using `linspace` over `[0, 1 - eps]` [S1, S2, unofficial].
    *   **Stochastic (`rng` is not `None`)**:
        *   Calculate `u_max = eps + (1 - eps) / num_samples` [S2, unofficial].
        *   Calculate `max_jitter = (1 - u_max) / (num_samples - 1) - eps` [S2, unofficial].
        *   Determine jitter dimension `d`: `1` if `single_jitter` is `True`, else `num_samples` [S2, unofficial].
        *   Generate base linspace `u_base = \text{linspace}(0, 1 - u_{\text{max}}, \text{num\_samples})` [S2, unofficial].
        *   Generate random jitter `j = \text{rand}(t.\text{shape}[:-1] + (d,))` [S2, unofficial].
        *   Compute final `u`:
            $$ u = u_{\text{base}} + j \times \text{max\_jitter} $$
            [S2, unofficial]
        *   *Note*: If `rng` is provided but `single_jitter` is False, the jitter shape must broadcast correctly with `t`'s batch dimensions [S2, unofficial].

4.  **Interpolate Samples**: Map the uniform samples `u` to the coordinate space `t` using the inverse CDF. This is implemented via sorted interpolation between the full CDF `cw0` and the bin endpoints `t`:
    $$ t_{\text{samples}} = \text{sorted\_interp}(u, cw0, t) $$
    [S2, unofficial]
    *   The full CDF `cw0` is constructed as:
        $$ cw0 = \text{concat}([0, cw, 1], \text{dim}=-1) $$
        [S2, unofficial]
    *   `sorted_interp` performs linear interpolation based on the sorted values of `cw0` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def piecewise_constant_pdf_sampling(t, w_logits, num_samples, rng=None, 
                                    deterministic_center=True, single_jitter=False, 
                                    eps=None, use_gpu_resampling=False):
    """
    Samples points from a piecewise-constant PDF defined by bin endpoints t and logits w_logits.
    
    Args:
        t: [..., num_bins + 1], sorted bin endpoint coordinates.
        w_logits: [..., num_bins], logits for bin weights.
        num_samples: int, number of samples to draw.
        rng: Optional random number generator. If None, uses deterministic linspace.
        deterministic_center: bool, if True, centers samples in bins for deterministic case.
        single_jitter: bool, if True, uses same jitter for all samples along a ray.
        eps: float, machine epsilon. If None, inferred from t.dtype.
        use_gpu_resampling: bool, if True, uses gather instruction (not implemented in pure Python pseudocode).
        
    Returns:
        t_samples: [..., num_samples], sampled coordinates.
    """
    if eps is None:
        eps = torch.finfo(t.dtype).eps
        
    # Step 1: Compute PDF weights
    w = torch.softmax(w_logits, dim=-1)
    
    # Step 2: Construct CDF (excluding last bin, clamped to 1)
    cw = torch.cumsum(w[..., :-1], dim=-1)
    cw = torch.clamp(cw, max=1.0)
    
    # Step 3: Construct Full CDF
    # cw0 = [0, cw, 1]
    zeros = torch.zeros_like(cw[..., :1])
    ones = torch.ones_like(cw[..., :1])
    cw0 = torch.cat([zeros, cw, ones], dim=-1)
    
    # Step 4: Generate Uniform Samples u
    if rng is None:
        # Deterministic sampling
        if deterministic_center:
            pad = 1.0 / (2.0 * num_samples)
            u = torch.linspace(pad, 1.0 - pad - eps, num_samples)
        else:
            u = torch.linspace(0.0, 1.0 - eps, num_samples)
        # Broadcast u to match t shape's batch dimensions if necessary
        # Assuming t has batch dims [...], u needs to be [..., num_samples]
        # For simplicity, assuming t is [..., num_bins+1] and we want output [..., num_samples]
        # u needs to be expanded to [..., num_samples]
        u = u.expand(*t.shape[:-1], num_samples)
    else:
        # Stochastic sampling
        u_max = eps + (1.0 - eps) / num_samples
        max_jitter = (1.0 - u_max) / (num_samples - 1) - eps
        d = 1 if single_jitter else num_samples
        
        u_base = torch.linspace(0.0, 1.0 - u_max, num_samples)
        # Generate random jitter
        # Shape: t.shape[:-1] + (d,)
        jitter_shape = list(t.shape[:-1]) + [d]
        j = torch.rand(jitter_shape, device=t.device)
        
        u = u_base.expand(*t.shape[:-1], num_samples) + j * max_jitter
        
    # Step 5: Interpolate using Inverse CDF
    # math.sorted_interp(u, cw0, t)
    # Since cw0 is sorted (cumulative sum of positive weights), we can use standard interpolation
    # However, sorted_interp implies handling potential unsorted inputs or specific GPU optimizations.
    # Here we assume standard linear interpolation between bin edges.
    
    # Find indices where u falls within cw0
    # cw0 shape: [..., num_bins + 2]
    # u shape: [..., num_samples]
    
    # Reshape for broadcasting: cw0 [..., num_bins+2], u [..., num_samples, 1]
    # We need to find the bin index for each sample.
    
    # Using searchsorted is more robust than manual loop
    # cw0 is sorted.
    # indices = torch.searchsorted(cw0, u, side='right') - 1
    # Clamp indices to valid range [0, num_bins]
    # indices = torch.clamp(indices, min=0, max=t.shape[-1] - 2) # num_bins
    
    # Linear interpolation
    # t_left = t[..., indices]
    # t_right = t[..., indices + 1]
    # cw_left = cw0[..., indices]
    # cw_right = cw0[..., indices + 1]
    
    # denom = cw_right - cw_left
    # denom = torch.where(denom == 0, 1.0, denom) # Avoid division by zero
    # t_samples = t_left + (u - cw_left) / denom * (t_right - t_left)
    
    # Note: The source uses math.sorted_interp which likely handles this efficiently.
    # For pure Python/PyTorch implementation:
    
    # Expand u for searchsorted
    u_expanded = u.unsqueeze(-1) # [..., num_samples, 1]
    cw0_expanded = cw0.unsqueeze(-2) # [..., 1, num_bins+2]
    
    # Find indices
    indices = torch.searchsorted(cw0_expanded, u_expanded, side='right') - 1
    indices = torch.clamp(indices, min=0, max=t.shape[-1] - 2)
    
    # Get bin edges
    t_left = torch.gather(t, -1, indices)
    t_right = torch.gather(t, -1, indices + 1)
    cw_left = torch.gather(cw0, -1, indices)
    cw_right = torch.gather(cw0, -1, indices + 1)
    
    # Interpolate
    denom = cw_right - cw_left
    denom = torch.where(denom == 0, 1.0, denom)
    t_samples = t_left + (u - cw_left) / denom * (t_right - t_left)
    
    return t_samples
```

**Edge Cases and Pitfalls:**
*   **Zero Probability Bins**: If a bin has zero weight, its contribution to the CDF is zero. The interpolation handles this by potentially returning the left edge if the CDF does not increase [S2, unofficial].
*   **Boundary Precision**: Using `eps` prevents division by zero and ensures samples stay within `[0, 1]` for the uniform distribution, avoiding out-of-bounds interpolation [S1, S2, unofficial].
*   **Jittering**: If `single_jitter` is `False`, each sample gets independent jitter, which may increase variance but reduces aliasing [S1, S2, unofficial]. If `single_jitter` is `True`, jitter is shared, which is faster but may introduce correlation artifacts [S1, S2, unofficial].
*   **GPU vs TPU**: `use_gpu_resampling` flag suggests that on TPUs, the gather instruction might be slow, so alternative implementations might be preferred [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot B)$, where $N$ is the number of samples and $B$ is the number of bins. The softmax, cumsum, and interpolation are all linear in the number of bins and samples [S1, S2, unofficial].
*   **Memory Complexity**: $O(N \cdot B)$ for storing the CDF and intermediate tensors [S1, S2, unofficial].
*   **Performance**: On GPUs, using gather instructions can be fast, but may be slow on TPUs [S1, unofficial]. Deterministic sampling is faster as it avoids random number generation [S1, S2, unofficial].
*   **Strengths**: Provides exact sampling from the learned discrete distribution; supports both deterministic and stochastic modes; handles jittering for anti-aliasing [S1, S2, unofficial].
*   **Limitations**: Requires storing the full CDF; interpolation can be computationally expensive for large numbers of bins [S1, S2, unofficial].
*   **When to Use**: When sampling from a learned discrete distribution in volume rendering or neural radiance fields (NeRF) [S1, S2, unofficial].
*   **When Not to Use**: When memory is constrained and the number of bins is very large; when deterministic sampling is not required and simpler uniform sampling suffices [S1, S2, unofficial].

## Background
Piecewise-Constant PDF Sampling is used in the context of neural volume rendering and Gaussian Splatting, specifically in implementations like DOFGS (Differentiable Oriented Gaussian Splatting) and density-bounded ray weights [S1, S2, unofficial]. It allows for efficient sampling of points along rays based on learned probability distributions, improving rendering quality and training stability [S1, S2, unofficial].

## Sources
[S1] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S2] rongduo/DOFGS_implementation, https://github.com/rongduo/DOFGS_implementation, fetched 2026-10-27 (unofficial)
