# Near-Middle-Far Ray Generation
## Overview
Near-Middle-Far Ray Generation divides a camera ray into two distinct segments: a linear interpolation from the near plane to a middle point, and a disparity (inverse-linear) interpolation from the middle point to the far plane [S1, unofficial]. It allocates a configurable fraction of samples to the near-middle segment and the remainder to the middle-far segment [S1, unofficial].
## How it works
**Inputs:**
- `near`: float or tensor, the distance to the near clipping plane.
- `far`: float or tensor, the distance to the far clipping plane.
- `middle`: float or tensor, the distance to the middle point where the sampling strategy switches.
- `point_count`: int, the total number of sample points to generate along the ray.
- `middle_split`: float, the fraction of `point_count` allocated to the near-middle segment (default 0.6) [S1, unofficial].
- `jitter`: float in $[0, 1)$, a fraction of the step length used to perturb sample positions [S1, unofficial].
- `campos`: tensor, the camera position.
- `raydir`: tensor, the ray direction.

**Outputs:**
- `raypos`: tensor of sample positions along the ray.
- `valid`: tensor of ones indicating all samples are valid [S1, unofficial].

**Data Structures:**
- `t0`, `t1`: tensors of linearly spaced values in $[0, 1]$ with sizes $N_0$ and $N_1$.
- `d0`, `d1`: tensors of computed distances for the near-middle and middle-far segments.
- `all_dists`: concatenated distance tensor of shape `(batch, height, width, N0 + N1)`.
- `segment_length`: tensor of truncated, jittered segment lengths of shape `(batch, height, width, point_count)`.
- `end_point_ts`: tensor of cumulative end-point distances of shape `(batch, height, width, point_count + 1)`.
- `middle_point_ts`: tensor of midpoint distances of shape `(batch, height, width, point_count)`.

**Steps:**
1. Calculate point counts: $N_0 = \lfloor \text{point\_count} \times \text{middle\_split} \rfloor + 1$ and $N_1 = \lfloor \text{point\_count} \times (1 - \text{middle\_split}) \rfloor + 2$ [S1, unofficial].
2. Generate linear parameter values: `t0 = linspace(0, 1, N0)` and `t1 = linspace(0, 1, N1)` [S1, unofficial].
3. Compute near-middle distances: `d0 = near * (1 - t0) + middle * t0` [S1, unofficial].
4. Compute middle-far distances: `d1 = 1.0 / (1.0/middle * (1 - t1) + 1.0/far * t1)` [S1, unofficial].
5. Concatenate distances along the sample dimension: `all_dists = cat([d0, d1], dim=-1)` [S1, unofficial].
6. Compute raw segment lengths: `diffs = all_dists[..., 1:] - all_dists[..., :-1]` [S1, unofficial].
7. Apply jitter: `jittered = diffs * (1.0 + jitter * (rand_like(diffs) - 0.5))` [S1, unofficial].
8. Truncate to `point_count` samples: `segment_length = jittered[..., :point_count]` [S1, unofficial].
9. Compute cumulative distances, prepend zeros, and add `near` offset: `end_point_ts = near + cat([zeros_like(segment_length[..., :1]), cumsum(segment_length)], dim=-1)` [S1, unofficial].
10. Compute middle-point distances: `middle_point_ts = (end_point_ts[..., :-1] + end_point_ts[..., 1:]) / 2.0` [S1, unofficial].
11. Compute ray positions: `raypos = campos + raydir * middle_point_ts` [S1, unofficial].
12. Set validity mask: `valid = ones_like(middle_point_ts)` [S1, unofficial].

**Formulas:**
- Linear interpolation: $d_0 = \text{near} \times (1 - t) + \text{middle} \times t$
- Disparity interpolation: $d_1 = \frac{1}{\frac{1}{\text{middle}} \times (1 - t) + \frac{1}{\text{far}} \times t}$
- Jittered segment length: $\Delta = \Delta_{\text{raw}} \times (1 + \text{jitter} \times (u - 0.5))$, where $u \sim \mathcal{U}(0, 1)$ [S1, unofficial].
- Midpoint distance: $t_{\text{mid}} = \frac{t_{\text{end}, i} + t_{\text{end}, i+1}}{2}$ [S1, unofficial].

**Parameters:**
- `middle_split`: float, default 0.6 [S1, unofficial].
- `jitter`: float in $[0, 1)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `middle` is not strictly between `near` and `far`, the disparity formula may yield invalid or inverted distances [G].
- If `point_count` exceeds $N_0 + N_1 - 1$, the truncation step discards samples; the source assumes $N_0 + N_1 - 1 \ge \text{point\_count}$ [S1, unofficial].
- Division by zero occurs if `middle` or `far` is zero in the disparity formula [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def near_middle_far_ray_generation(near, far, middle, point_count, middle_split=0.6, jitter=0.0, campos=None, raydir=None):
    # 1. Calculate point counts
    N0 = int(point_count * middle_split) + 1
    N1 = int(point_count * (1 - middle_split)) + 2

    # 2. Generate linear parameter values [0, 1]
    t0 = torch.linspace(0, 1, N0)
    t1 = torch.linspace(0, 1, N1)

    # 3. Compute near-middle distances (linear)
    d0 = near * (1.0 - t0) + middle * t0

    # 4. Compute middle-far distances (disparity)
    inv_middle = 1.0 / middle
    inv_far = 1.0 / far
    d1 = 1.0 / (inv_middle * (1.0 - t1) + inv_far * t1)

    # 5. Concatenate distances along the sample dimension
    all_dists = torch.cat([d0, d1], dim=-1)

    # 6. Compute raw segment lengths
    diffs = all_dists[..., 1:] - all_dists[..., :-1]

    # 7. Apply jitter
    random_vals = torch.rand_like(diffs)
    jittered = diffs * (1.0 + jitter * (random_vals - 0.5))

    # 8. Truncate to point_count samples
    segment_length = jittered[..., :point_count]

    # 9. Compute cumulative distances, prepend zeros, add near offset
    cumsum_lengths = torch.cumsum(segment_length, dim=-1)
    zeros = torch.zeros_like(segment_length[..., :1])
    end_point_ts = near + torch.cat([zeros, cumsum_lengths], dim=-1)

    # 10. Compute middle-point distances
    middle_point_ts = (end_point_ts[..., :-1] + end_point_ts[..., 1:]) / 2.0

    # 11. Compute ray positions
    if campos is not None and raydir is not None:
        raypos = campos + raydir * middle_point_ts.unsqueeze(-1)
    else:
        raypos = middle_point_ts

    # 12. Set validity mask
    valid = torch.ones_like(middle_point_ts)

    return raypos, valid
```
## Operating characteristics
- Time complexity: Not given in the sources.
- Memory complexity: Not given in the sources.
- Measured performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths and limitations: Not given in the sources.
- When to use it and when not: Not given in the sources.
## Background
Origin: Described in the Xharlie/pointnerf repository [S1, unofficial].
## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
