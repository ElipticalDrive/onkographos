# Batch Stratified Sampling

Also called: Stratified Sampling, Uniform Stratified Sampling

## Overview
Batch Stratified Sampling generates $N_c$ evenly-spaced depth bins in the interval $[near, far]$ and perturbs sample positions within each bin to reduce aliasing artifacts during volume rendering [S1, unofficial]. It extends Uniform Ray Sampling by stratifying the depth interval, ensuring samples are distributed more evenly across the ray's depth range [S1, unofficial].

## How it works
**Inputs:**
- `ray_o`: Ray origins, shape $(N, 3)$.
- `ray_d`: Ray directions, shape $(N, 3)$.
- `near`: Minimum depth, scalar or shape $(N, 1)$.
- `far`: Maximum depth, scalar or shape $(N, 1)$.
- `num_samples` ($N_c$): Number of depth bins/samples per ray.
- `perturb` (bool): Whether to add random jitter within bins. Default `True` [S1, unofficial].

**Outputs:**
- `t_vals`: Sample depths along the ray, shape `(*, N_c)`.
- `pts`: 3D sample points, shape `(*, N_c, 3)`.
- `dirs`: Ray directions repeated for each sample, shape `(*, N_c, 3)`.

**Steps:**
1.  **Create Base Grid:** Compute evenly spaced depth values $t_{vals}$ in $[near, far]$ using linear spacing: $t_{vals} = \text{linspace}(near, far, N_c)$ [S1, unofficial].
2.  **Compute Midpoints:** Calculate the midpoints between adjacent $t_{vals}$ to define bin boundaries: $mids = 0.5 \times (t_{vals}[1:] + t_{vals}[:-1])$ [S1, unofficial].
3.  **Normalize and Expand:** Normalize `near` and `far` tensors to 2D shape `[num_rays, 1]` if they are 1D [S3, unofficial]. Expand $t_{vals}$ to shape `(*, N_c)` to match ray batch dimensions [S1, unofficial].
4.  **Construct Bin Boundaries (Normalized):** Create normalized $t_{vals}$ in $[0, 1]$ and expand to `[num_rays, N_c]` [S3, unofficial]. Calculate bin midpoints in normalized space: $mid = (t_{vals}[...,:-1] + t_{vals}[...,1:]) / 2$ [S1, unofficial]. Construct lower interval boundaries by concatenating the first $t$ value with midpoints: $lower = \text{concat}(t_{vals}[...,:1], mid)$ [S1, unofficial]. Construct upper interval boundaries by concatenating midpoints with the last $t$ value: $upper = \text{concat}(mid, t_{vals}[...,-1:])$ [S1, unofficial].
5.  **Interpolate to World Depth:** Map the normalized bin boundaries back to the $[near, far]$ interval for each ray: $bins = near \times (1.0 - t_{norm}) + far \times t_{norm}$ [S3, unofficial].
6.  **Sample Within Bins:** If `perturb` is `True`, perturb $t_{vals}$ by adding a random value scaled by bin width: $t_{vals} = lower + (upper - lower) \times \text{rand}()$ [S1, unofficial]. If `perturb` is `False`, use the bin midpoints directly [S1, unofficial].
7.  **Compute 3D Points:** Compute sample positions along the ray: $pts = ray\_o.\text{unsqueeze}(-2) + ray\_d.\text{unsqueeze}(-2) \times t_{vals}.\text{unsqueeze}(-1)$ [S1, unofficial].
8.  **Compute Directions:** Compute ray directions for MLP input: $dirs = ray\_d.\text{unsqueeze}(-2).\text{expand\_as}(pts)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def batch_stratified_sampling(ray_o, ray_d, near, far, num_samples, perturb=True):
    """
    ray_o: (N, 3)
    ray_d: (N, 3)
    near: scalar or (N, 1)
    far: scalar or (N, 1)
    num_samples: int
    perturb: bool
    """
    # 1. Create evenly spaced depth values
    t_vals = torch.linspace(near, far, num_samples, device=ray_o.device)
    
    # 2. Compute midpoints between adjacent t_vals
    mids = 0.5 * (t_vals[1:] + t_vals[:-1])
    
    # 3. Normalize near/far to 2D shape [num_rays, 1] if they are 1D
    N = ray_o.shape[0]
    if near.ndim == 0:
        near = near.expand(N, 1)
    if far.ndim == 0:
        far = far.expand(N, 1)
        
    # 4. Create normalized t_vals in [0, 1] and expand to [num_rays, num_samples]
    t_norm = torch.linspace(0, 1, num_samples, device=ray_o.device)
    t_norm = t_norm.expand(N, num_samples)
    
    # 5. Calculate bin midpoints in normalized space
    mid_norm = (t_norm[:, :-1] + t_norm[:, 1:]) / 2
    
    # 6. Construct lower and upper interval boundaries
    lower_norm = torch.cat((t_norm[:, :1], mid_norm), dim=-1)
    upper_norm = torch.cat((mid_norm, t_norm[:, -1:]), dim=-1)
    
    # 7. Interpolate bin boundaries for each ray to world depth
    lower_depth = near * (1.0 - lower_norm) + far * lower_norm
    upper_depth = near * (1.0 - upper_norm) + far * upper_norm
    
    # 8. Sample within bins
    if perturb:
        t_vals = lower_depth + (upper_depth - lower_depth) * torch.rand_like(lower_depth)
    else:
        t_vals = 0.5 * (lower_depth + upper_depth)
        
    # 9. Compute 3D points
    pts = ray_o.unsqueeze(1) + ray_d.unsqueeze(1) * t_vals.unsqueeze(2)
    
    # 10. Compute ray directions for MLP input
    dirs = ray_d.unsqueeze(1).expand_as(pts)
    
    return t_vals, pts, dirs
```

**Edge Cases and Pitfalls:**
-   **Precision:** Using float32 for `t_vals` is standard, but if `near` and `far` differ by orders of magnitude, precision loss may occur in the interpolation step [G]. Ensure `near` and `far` are normalized or scaled appropriately if extreme values are expected [G].
-   **Boundaries:** The first sample bin starts at `near` and the last ends at `far`. The midpoints ensure that the bins cover the entire interval without gaps or overlaps [S1, unofficial].
-   **Degenerate Input:** If `near == far`, the bin width is zero [G]. The implementation should handle this by returning `near` for all samples or raising an error [G].

## Subtopics

### Jittered Sampling
Generates a regular grid of samples per pixel with independent uniform random offsets within each grid cell to reduce aliasing while maintaining stratification [S4, unofficial]. It loops over grid indices $ax, ay$ from $0$ to $A_{depth}-1$, computes offsets $xOffset = ax/A_{depth}$, $yOffset = ay/A_{depth}$, and adds random jitter $xOffset_{random} = rand() \times xOffset$, $yOffset_{random} = rand() \times yOffset$ to the base sample position [S4, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N \cdot N_c)$, where $N$ is the number of rays and $N_c$ is the number of samples per ray [G].
-   **Memory Complexity:** $O(N \cdot N_c)$ for storing `t_vals`, `pts`, and `dirs` [G].
-   **Strengths:** Simple to implement, deterministic structure with random perturbation, effective for reducing high-frequency noise [G].
-   **Limitations:** Not optimal for importance sampling where density varies significantly [G].
-   **When to Use:** Standard direct volume rendering, NeRF training, when uniform distribution is desired but jitter is needed to break up regularity [G].
-   **When Not to Use:** When importance sampling based on density is required (use [Stratified Monte Carlo Radiance Estimate](1373-stratified-monte-carlo-radiance-estimate.md) or similar) [G].

## Background
Batch Stratified Sampling is a fundamental technique in computer graphics, particularly in the context of Neural Radiance Fields (NeRF) and volume rendering [S1, unofficial]. It builds on the concept of stratified sampling, which divides the domain into bins and samples uniformly within each bin to reduce variance [S1, unofficial]. In the context of NeRF, it is used to sample points along rays for querying the neural network [S1, unofficial].

## Sources
[S1] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S4] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
