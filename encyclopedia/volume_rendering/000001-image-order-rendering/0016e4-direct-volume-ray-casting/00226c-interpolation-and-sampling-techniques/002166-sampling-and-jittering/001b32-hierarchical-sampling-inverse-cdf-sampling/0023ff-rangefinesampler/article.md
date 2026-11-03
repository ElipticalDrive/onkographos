# RangeFineSampler


## Overview
RangeFineSampler is a hierarchical sampling technique for direct volume ray casting that refines coarse samples by generating a dense uniform grid around the coarse sample with the highest weight, supplemented by random samples across the full ray extent [S1, unofficial]. It addresses missing high-frequency details between coarse samples by concentrating computational effort near the most significant signal while maintaining global coverage through random jittering [S1, unofficial]. Unlike [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) which derives samples from opacity gradients, or [Proposal Network Sampling](1589-proposal-network-sampling.md) which uses learned neural networks to predict sample locations, RangeFineSampler relies on a deterministic geometric expansion around the peak weight index combined with stochastic uniform sampling [S1, unofficial].

## How it works
The method takes coarse sample positions and weights as input and outputs a refined set of sample positions along the ray [S1, unofficial]. It operates on batches of rays, where each ray has a set of coarse z-values (depths) and corresponding weights [S1, unofficial].

**Inputs:**
- `coarse_z_vals`: Tensor of shape `(num_rays, num_coarse_samples)` containing the depth positions of coarse samples [S1, unofficial].
- `coarse_weights`: Tensor of shape `(num_rays, num_coarse_samples)` containing the importance weights for each coarse sample [S1, unofficial].
- `near`: Float, the near plane distance of the ray [S1, unofficial].
- `far`: Float, the far plane distance of the ray [S1, unofficial].
- `max_samples`: Integer, the maximum total number of samples allowed per ray [S1, unofficial].
- `N_samples`: Integer, the number of fine samples to generate around the peak [S1, unofficial].
- `range`: Float, the total width of the interval around the max weight z-value where uniform samples are placed. Default is 0.5 [S1, unofficial].
- `deterministic`: Boolean, whether to use stratified sampling or pure random jittering [S1, unofficial].

**Outputs:**
- `z_vals_ret`: Tensor of shape `(num_rays, num_total_samples)` containing the sorted, refined sample positions [S1, unofficial].

**Data Structures:**
The implementation uses batched tensor indexing and sorting operations [S1, unofficial].

**Steps:**

1.  **Determine Effective Sample Count:**
    Calculate the number of fine samples to generate around the peak [S1, unofficial]. This is bounded by `max_samples` and `N_samples` [S1, unofficial].
    $$ N_{fine} = \min(\text{max\_samples}, N_{samples}) $$ [S1, unofficial]

2.  **Find Peak Weight Index:**
    Identify the index of the coarse sample with the maximum weight for each ray [S1, unofficial].
    $$ \text{max\_indices} = \text{argmax}_{dim=-1}(\text{coarse\_weights}) $$ [S1, unofficial]

3.  **Retrieve Peak Z-Value:**
    Get the depth value corresponding to the peak weight for each ray [S1, unofficial].
    $$ \text{max\_z\_vals} = \text{coarse\_z\_vals}[\text{arange}(\text{num\_rays}), \text{max\_indices}] $$ [S1, unofficial]

4.  **Generate Uniform Samples Around Peak:**
    Create a uniform grid of $N_{fine}$ samples centered around `max_z_vals` [S1, unofficial]. The source defines `range` as the total width, but the code formula uses `2 * range`, effectively making `range` a half-width in implementation [S1, unofficial]. The formula distributes points evenly across this interval [S1, unofficial].
    $$ z_{fine} = \text{max\_z\_vals}[:, \text{None}] - \text{range} + \frac{2 \cdot \text{range}}{N_{fine} - 1} \cdot \text{arange}(N_{fine}) $$ [S1, unofficial]

5.  **Apply Stratified Sampling (Optional):**
    If `deterministic` is False, apply stratified jittering to the uniform samples to reduce aliasing [S1, unofficial].
    - Compute midpoints between adjacent uniform samples:
      $$ \text{mids} = 0.5 \cdot (z_{fine}[:, 1:] + z_{fine}[:, :-1]) $$ [S1, unofficial]
    - Define intervals:
      $$ \text{lower} = \text{concat}([z_{fine}[:, :1], \text{mids}], \text{dim}=-1) $$ [S1, unofficial]
      $$ \text{upper} = \text{concat}([\text{mids}, z_{fine}[:, -1:]], \text{dim}=-1) $$ [S1, unofficial]
    - Generate random jitter:
      $$ t_{rand} = \text{rand}(z_{fine}.shape) $$ [S1, unofficial]
    - Jitter the samples:
      $$ z_{fine} = \text{lower} + (\text{upper} - \text{lower}) \cdot t_{rand} $$ [S1, unofficial]
    If `deterministic` is True, $z_{fine}$ remains the uniform grid from Step 4 [S1, unofficial].

6.  **Generate Random Samples Across Full Range:**
    Generate $N_{fine}$ random samples uniformly distributed between `near` and `far` for each ray [S1, unofficial].
    $$ z_{add} = \text{rand}((\text{num\_rays}, N_{fine})) \cdot (\text{far} - \text{near}) + \text{near} $$ [S1, unofficial]

7.  **Combine and Sort (Global):**
    Concatenate the original coarse samples with the random samples $z_{add}$ and sort them [S1, unofficial].
    $$ z_{base} = \text{sort}(\text{concat}([\text{coarse\_z\_vals}, z_{add}], \text{dim}=-1), \text{dim}=-1) $$ [S1, unofficial]

8.  **Inject Fine Samples for Non-Edge Rays:**
    For rays where the peak index is not at the very beginning (index > 0), concatenate the structured fine samples $z_{fine}$ with the coarse samples and sort [S1, unofficial].
    $$ \text{mask} = \text{max\_indices} > 0 $$ [S1, unofficial]
    $$ z_{ret}[\text{mask}] = \text{sort}(\text{concat}([\text{coarse\_z\_vals}[\text{mask}], z_{fine}[\text{mask}]], \text{dim}=-1), \text{dim}=-1) $$ [S1, unofficial]
    For rays where `max_indices == 0`, the sorted combination of coarse and random samples ($z_{base}$) is used [S1, unofficial].

9.  **Return Result:**
    The final tensor $z_{ret}$ contains the sorted sample positions [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def range_fine_sampler(coarse_z_vals, coarse_weights, near, far, max_samples, N_samples, range_val, deterministic=True):
    """
    RangeFineSampler implementation.
    
    Args:
        coarse_z_vals: (num_rays, num_coarse) Tensor of coarse sample depths.
        coarse_weights: (num_rays, num_coarse) Tensor of coarse sample weights.
        near: Float, near plane.
        far: Float, far plane.
        max_samples: Int, max total samples.
        N_samples: Int, number of fine samples to generate around peak.
        range_val: Float, width of interval around peak.
        deterministic: Bool, use stratified sampling.
        
    Returns:
        z_vals_ret: (num_rays, num_total) Tensor of sorted sample depths.
    """
    num_rays, num_coarse = coarse_z_vals.shape
    
    # Step 1: Determine effective sample count
    N_fine = min(max_samples, N_samples)
    
    # Step 2: Find peak weight index
    max_indices = torch.argmax(coarse_weights, dim=-1)
    
    # Step 3: Retrieve peak z-values
    arange_rays = torch.arange(num_rays).to(coarse_z_vals.device)
    max_z_vals = coarse_z_vals[arange_rays, max_indices]
    
    # Step 4: Generate uniform samples around peak
    # Handle N_fine = 1 case to avoid division by zero [G]
    if N_fine == 1:
        z_fine = max_z_vals.unsqueeze(1)
    else:
        arange_fine = torch.arange(N_fine).to(coarse_z_vals.device)
        z_fine = max_z_vals.unsqueeze(1) - range_val + (2 * range_val / (N_fine - 1)) * arange_fine.unsqueeze(0)
        
    # Step 5: Stratified sampling
    if not deterministic:
        mids = 0.5 * (z_fine[:, 1:] + z_fine[:, :-1])
        lower = torch.cat([z_fine[:, :1], mids], dim=-1)
        upper = torch.cat([mids, z_fine[:, -1:]], dim=-1)
        t_rand = torch.rand(z_fine.shape, device=coarse_z_vals.device)
        z_fine = lower + (upper - lower) * t_rand
        
    # Step 6: Generate random samples across full range
    z_add = torch.rand((num_rays, N_fine), device=coarse_z_vals.device) * (far - near) + near
    
    # Step 7: Combine coarse and random, sort
    z_vals_ret = torch.sort(torch.cat([coarse_z_vals, z_add], dim=-1), dim=-1)[0]
    
    # Step 8: Inject fine samples for rays where max_indices > 0
    mask = max_indices > 0
    if mask.any():
        z_combined = torch.cat([coarse_z_vals[mask], z_fine[mask]], dim=-1)
        z_vals_ret[mask] = torch.sort(z_combined, dim=-1)[0]
        
    return z_vals_ret
```

**Parameters:**
- `range`: Float, default 0.5. The total width of the interval around the max weight z-value [S1, unofficial].
- `N_samples`: Integer. Number of fine samples to generate [S1, unofficial].
- `max_samples`: Integer. Global cap on total samples [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `N_fine` is 1, the uniform spacing formula divides by zero. The implementation must handle this by returning the single peak value [G].
- **Boundary Conditions:** The random samples `z_add` are strictly within `[near, far]` [S1, unofficial]. The uniform samples `z_fine` are centered on `max_z_vals` with width `range` [S1, unofficial]. If `max_z_vals` is close to `near` or `far`, `z_fine` may extend outside the valid ray bounds [G].
- **Index 0 Peak:** When `max_indices == 0`, the fine samples are not injected [S1, unofficial]. This may lead to lower resolution near the start of the ray if the peak is at the beginning [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ per ray due to sorting, where $N$ is the total number of samples (coarse + fine + random) [G].
- **Memory Complexity:** $O(N)$ per ray to store the sample positions [G].
- **Performance:** Not given in the sources.
- **Strengths:** Simple to implement; concentrates samples where they are most needed (high weight); reduces aliasing via stratified jittering [S1, unofficial].
- **Limitations:** The fixed `range` parameter may not adapt well to varying scene densities; the lack of clamping for `z_fine` can lead to out-of-bounds samples; the logic for `max_indices == 0` may result in suboptimal sampling at the ray start [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
