# Depth-Prior Guided Stratified Sampling

Also called: Confidence-bounded sampling, Priors-guided sampling

## Overview
Depth-Prior Guided Stratified Sampling replaces uniform depth sampling in volume rendering by using predicted depth priors and associated confidence bounds to focus sampling on likely object surfaces [S1, unofficial].

## How it works
**Inputs:**
- `depth_priors`: Tensor of shape $[N_{rays}]$, containing the predicted depth for each ray [S1, unofficial].
- `depth_confidences`: Tensor of shape $[N_{rays}]$, containing the uncertainty/confidence width for each ray [S1, unofficial].
- `near_bound`: Scalar float, the minimum allowed depth offset multiplier [S1, unofficial].
- `far_bound`: Scalar float, the maximum allowed depth offset multiplier [S1, unofficial].
- `lindisp`: Boolean flag. If `True`, samples linearly in inverse depth; otherwise, linearly in depth [S1, unofficial].
- `perturb`: Float. If non-zero, stratified random sampling is applied [S1, unofficial].
- `N_samples`: Integer, number of samples per ray [S1, unofficial].

**Outputs:**
- `z_vals`: Tensor of shape $[N_{rays}, N_{samples}]$, containing the sampled depths along each ray [S1, unofficial].

**Data Structures:**
- Not given in the sources.

**Steps:**
1. **Compute Bounds:** Calculate the near and far depth limits for each ray using the depth prior and confidence, clamped to the global bounds [S1, unofficial].
   $$ near = depth\_priors \times (1 - \text{clamp}(depth\_confidences, min=near\_bound, max=far\_bound)) $$
   $$ far = depth\_priors \times (1 + \text{clamp}(depth\_confidences, min=near\_bound, max=far\_bound)) $$
   Here, $\text{clamp}(v, min, max)$ restricts $v$ to the range $[min, max]$ [S1, unofficial].
2. **Generate Linear Spacing:** Create a 1D tensor `t_vals` of shape $[N_{samples}]$ linearly spaced from 0 to 1 [S1, unofficial].
3. **Interpolate Depths:**
   - If `lindisp` is `False`:
     $$ z\_vals = near \times (1 - t\_vals) + far \times t\_vals $$
   - If `lindisp` is `True`:
     $$ z\_vals = \frac{1}{\frac{1}{near}(1 - t\_vals) + \frac{1}{far}t\_vals} $$
   This results in `z_vals` of shape $[N_{rays}, N_{samples}]$ [S1, unofficial].
4. **Stratified Sampling (if `perturb > 0`):**
   - Compute midpoints between adjacent samples:
     $$ mids = 0.5 \times (z\_vals[:, 1:] + z\_vals[:, :-1]) $$
   - Construct upper and lower bounds for each interval:
     $$ upper = \text{concatenate}(mids, z\_vals[:, -1:]) $$
     $$ lower = \text{concatenate}(z\_vals[:, :1], mids) $$
   - Generate random values:
     $$ t\_rand = \text{random\_uniform}(shape=z\_vals.shape) $$
   - Sample new depths:
     $$ z\_vals = lower + (upper - lower) \times t\_rand $$
   All steps follow the source implementation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def depth_prior_guided_stratified_sampling(
    depth_priors: torch.Tensor,
    depth_confidences: torch.Tensor,
    near_bound: float,
    far_bound: float,
    lindisp: bool,
    perturb: float,
    N_samples: int
) -> torch.Tensor:
    # Step 1: Compute bounds
    clamped_conf = torch.clamp(depth_confidences, min=near_bound, max=far_bound)
    near = depth_priors * (1.0 - clamped_conf)
    far = depth_priors * (1.0 + clamped_conf)

    # Step 2: Generate t_vals
    t_vals = torch.linspace(0, 1, N_samples)

    # Step 3: Interpolate
    t_vals_expanded = t_vals.unsqueeze(0)
    near_expanded = near.unsqueeze(1)
    far_expanded = far.unsqueeze(1)

    if lindisp:
        inv_near = 1.0 / near_expanded
        inv_far = 1.0 / far_expanded
        denom = inv_near * (1.0 - t_vals_expanded) + inv_far * t_vals_expanded
        z_vals = 1.0 / denom
    else:
        z_vals = near_expanded * (1.0 - t_vals_expanded) + far_expanded * t_vals_expanded

    # Step 4: Stratified Sampling if perturb > 0
    if perturb > 0:
        mids = 0.5 * (z_vals[:, 1:] + z_vals[:, :-1])
        upper = torch.cat([mids, z_vals[:, -1:]], dim=1)
        lower = torch.cat([z_vals[:, :1], mids], dim=1)
        t_rand = torch.rand(z_vals.shape)
        z_vals = lower + (upper - lower) * t_rand

    return z_vals
```

**Parameters:**
- `lindisp`: `bool`. If `True`, sampling occurs in inverse depth space [S1, unofficial].
- `perturb`: `float`. If non-zero, stratified random sampling is used [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** In `lindisp=True` mode, if `near` or `far` is zero, `inv_near` or `inv_far` becomes infinite. The source does not specify a guard for this, but numerically, $1/0$ should be handled (e.g., clamping near/far to a small epsilon) [G].
- **Clamping Logic:** The confidence is clamped to `near_bound` and `far_bound` [S1, unofficial].
- **Shape Broadcasting:** Care must be taken to broadcast `t_vals` correctly against `near` and `far` tensors of shape `[N_rays]` to produce `[N_rays, N_samples]` [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Focuses sampling on high-probability regions defined by the depth prior [S1, unofficial].
- **Limitations:** Relies on the accuracy of `depth_priors` and `depth_confidences` [S1, unofficial].
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
