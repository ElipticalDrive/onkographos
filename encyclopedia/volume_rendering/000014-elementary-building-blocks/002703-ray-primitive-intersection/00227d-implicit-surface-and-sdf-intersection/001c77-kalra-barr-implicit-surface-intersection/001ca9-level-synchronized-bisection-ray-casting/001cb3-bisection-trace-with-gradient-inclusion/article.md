# Bisection Trace with Gradient Inclusion

Also called: Level-Synchronized Interval Bisection

## Overview
Bisection Trace with Gradient Inclusion is a robust ray casting algorithm that renders the zero-level-set of an implicit function $f: \mathbb{R}^3 \to \mathbb{R}$ using level-synchronized interval bisection [S1, unofficial]. It guarantees no surface features are missed by using an inclusion function to bound $f$ over axis-aligned boxes and pruning intervals that cannot contain a zero crossing [S1, unofficial].

## How it works
**Inputs:** `origins` (array of $\mathbf{o} \in \mathbb{R}^3$), `directions` (array of $\mathbf{d} \in \mathbb{R}^3$, normalized), `t_min` and `t_max` (scalar bounds for the ray segment), `hit_mask` (boolean array indicating active rays), `inclusion` (`GradientInclusion` instance), `f_np` (implicit function evaluator), `max_depth` (integer, maximum bisection depth), `threshold` (float, bisection convergence threshold), `lipschitz` (float or 'auto', Lipschitz constant for the inclusion function), `newton_iter` (integer, optional Newton refinement steps), `batch_size` (integer, number of rays to process per batch; `None` processes all at once) [S1, unofficial].
**Outputs:** Final hit positions $\mathbf{x}$ and depths $t$ for all rays.
**Data Structures:** `Interval` $[f_-, f_+]$ representing the guaranteed range of $f$; ray state arrays `t_start` and `t_end` tracking the current search interval for each ray.
**Parameters:** `max_depth` (typical: 20–30), `threshold` (typical: $10^{-6}$ to $10^{-8}$ relative to scene scale), `lipschitz` (if 'auto', estimated by sampling gradient norms over the bounding box via `torch.autograd` [S1, unofficial]).
**Algorithm Steps:**
1. **Initialization:** Initialize `inclusion` with $f_{np}$ and $L$ [S1, unofficial]. For each active ray, set $[t_{start}, t_{end}] = [t_{min}, t_{max}]$.
2. **Level-Synchronized Bisection Loop:** For `step` from 1 to `max_depth`:
   - Compute midpoint $t_{mid} = (t_{start} + t_{end}) / 2$.
   - Evaluate inclusion function: $[f_{min}, f_{max}] = \text{inclusion.evaluate}(t_{start}, t_{end})$.
   - **Pruning:** If $0 \notin [f_{min}, f_{max}]$, deactivate the ray [S1, unofficial].
   - **Convergence:** If $|t_{end} - t_{start}| < \text{threshold}$, deactivate the ray.
   - **Refinement:** Evaluate $f(t_{start})$ and $f(t_{mid})$. If signs differ, set $t_{end} = t_{mid}$; otherwise set $t_{start} = t_{mid}$.
3. **Newton Refinement:** For converged rays, perform `newton_iter` steps: $t_{new} = t_{old} - f(\mathbf{x}) / (\nabla f(\mathbf{x}) \cdot \mathbf{d})$ [G].
4. **Output:** Return final $\mathbf{x}$ and $t$.
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bisection_trace_with_gradient_inclusion(origins, directions, t_min, t_max, hit_mask, inclusion, f_np, max_depth, threshold, lipschitz, newton_iter, batch_size):
    inclusion.lipschitz = lipschitz
    t_start = np.full_like(origins[..., 0], t_min)
    t_end = np.full_like(origins[..., 0], t_max)
    active_indices = np.where(hit_mask)[0]
    if batch_size is not None:
        for b_start in range(0, len(active_indices), batch_size):
            b_end = min(b_start + batch_size, len(active_indices))
            _trace_rays_batch(active_indices[b_start:b_end], origins, directions, t_start, t_end, hit_mask, inclusion, f_np, max_depth, threshold, newton_iter)
    else:
        _trace_rays_batch(active_indices, origins, directions, t_start, t_end, hit_mask, inclusion, f_np, max_depth, threshold, newton_iter)
    return t_start, origins + t_start[..., None] * directions

def _trace_rays_batch(indices, origins, directions, t_start, t_end, hit_mask, inclusion, f_np, max_depth, threshold, newton_iter):
    for step in range(max_depth):
        active_in_batch = hit_mask[indices]
        if not np.any(active_in_batch):
            break
        t_mid = 0.5 * (t_start[indices] + t_end[indices])
        f_min, f_max = inclusion.evaluate(t_start[indices], t_end[indices])
        no_zero = (f_min > 0) | (f_max < 0)
        hit_mask[indices[no_zero]] = False
        active_in_batch = hit_mask[indices]
        if not np.any(active_in_batch):
            break
        converged = (t_end[indices] - t_start[indices]) < threshold
        hit_mask[indices[converged]] = False
        remaining = active_in_batch & ~converged
        if np.any(remaining):
            rem_idx = indices[remaining]
            t_mid_rem = t_mid[remaining]
            pts_mid = origins[rem_idx] + t_mid_rem[..., None] * directions[rem_idx]
            f_mid = f_np(pts_mid)
            pts_start = origins[rem_idx] + t_start[rem_idx][..., None] * directions[rem_idx]
            f_start = f_np(pts_start)
            sign_change = f_start * f_mid < 0
            t_end[rem_idx[sign_change]] = t_mid_rem[sign_change]
            t_start[rem_idx[~sign_change]] = t_mid_rem[~sign_change]
```
**Edge Cases and Pitfalls:**
- **Lipschitz Constant:** If $L$ is underestimated, the inclusion bounds are not guaranteed, risking missed surfaces [G].
- **Degenerate Intervals:** The threshold check prevents division by zero during Newton refinement [G].
- **Multiple Crossings:** The method finds the first zero crossing within the active interval [G].

## Subtopics

### [Interval Arithmetic Inclusion Strategies](1410-interval-arithmetic-inclusion-strategies.md)
Provides the bounding functions used by this method. Choose `GradientInclusion` for a balance of tightness and generality, or `AffineInclusion` for symbolic functions requiring tighter bounds.

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of rays and $D$ is `max_depth` [G].
- **Memory Complexity:** $O(N)$ for storing ray states [G].
- **Performance:** Robust against aliasing and missed surfaces, but slower than simple bisection due to inclusion overhead [G].
- **Strengths:** Guaranteed no-miss property; handles complex implicit functions well [G].
- **Limitations:** Computationally expensive due to interval arithmetic; requires Lipschitz constant estimation [G].

## Background
The method is implemented in the `iso-surface-rendering` library by narnia-ai-mason [S1, unofficial]. It builds on level-synchronized bisection ray casting, enhancing it with gradient-based inclusion strategies for improved accuracy and efficiency.

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
