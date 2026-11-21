# Bisection Ray Tracing with Interval Inclusion

Also called: Fast and robust ray intersection with arbitrary implicits, Interval/Affine Arithmetic Ray Tracing, IsoRay bisection tracer, Robust Ray Intersection with Arbitrary Implicits, Self-validated arithmetic ray tracing

## Overview
Bisection Ray Tracing with Interval Inclusion is a robust ray-surface intersection algorithm that guarantees finding the nearest intersection of an implicit surface with a ray without missing fine features [S1, unofficial]. It solves the problem of precision loss and feature skipping inherent in fixed-step ray marching by using interval arithmetic to prune intervals that cannot contain a root, followed by bisection to narrow down the interval containing the sign change of the implicit function $f$ [S1, unofficial]. Unlike standard bisection which processes rays independently, this method processes rays in batches and uses front-to-back pruning to discard intervals that are behind already found hits, improving efficiency for scenes with multiple surfaces or occlusion [S1, unofficial]. It builds on [Kalra & Barr Implicit Surface Intersection](1278-kalra-barr-implicit-surface-intersection.md) by providing a specific batched, interval-arithmetic-based implementation strategy [S1, unofficial].

## How it works
**Inputs:**
- `origins`: Array of shape $(R, 3)$, ray origins [S1, unofficial].
- `directions`: Array of shape $(R, 3)$, ray directions (normalized) [S1, unofficial].
- `t_min`: Array of shape $(R)$, near plane distances [S1, unofficial].
- `t_max`: Array of shape $(R)$, far plane distances [S1, unofficial].
- `hit_mask`: Boolean array of shape $(R)$, rays that intersect the bounding volume [S1, unofficial].
- `inclusion`: An inclusion function object (e.g., `SamplingInclusion`) capable of evaluating bounds on the implicit field $f$ over an interval $[t_{lo}, t_{hi}]$ [S1, unofficial].
- `f`: The implicit surface function [S1, unofficial].
- `max_depth`: Maximum number of bisection iterations allowed per ray. Default value is 40 [S1, unofficial].
- `threshold`: The convergence width threshold for the bisection interval. Default value is $10^{-5}$ [S1, unofficial].

**Outputs:**
- `result`: A `TraceResult` dataclass containing:
  - `hit`: Boolean array $(R)$, true if a surface was found [S1, unofficial].
  - `t`: Float64 array $(R)$, distance to the closest hit [S1, unofficial].
  - `position`: Float64 array $(R, 3)$, hit position ($origin + t \cdot direction$) [S1, unofficial].
  - `normal`: Float64 array $(R, 3)$, surface normal (zeros if not hit) [S1, unofficial].
  - `depth_count`: Int32 array $(R)$, number of bisection steps taken [S1, unofficial].
  - `t_lo`: Float64 array $(R)$, lower bound of the final interval [S1, unofficial].
  - `t_hi`: Float64 array $(R)$, upper bound of the final interval [S1, unofficial].

**Data Structures:**
The algorithm maintains active intervals for rays [S1, unofficial]. Key arrays include:
- `ray_idx`: Indices of active rays [S1, unofficial].
- `iv_t_lo`: Lower bounds of intervals for active rays [S1, unofficial].
- `iv_t_hi`: Upper bounds of intervals for active rays [S1, unofficial].
- `result_t`: Array storing the closest hit distance found so far for each ray (initialized to `t_max` or a large value) [S1, unofficial].
- `result_hit`: Boolean array tracking if a hit has been recorded [S1, unofficial].

**Steps:**

1. **Initialization:**
   - Initialize `result_t` with `t_max` (or a large value) for all rays [S1, unofficial].
   - Initialize `result_hit` to `False` [S1, unofficial].
   - Initialize `result_depth` to 0 [S1, unofficial].
   - Identify valid rays: `valid_ray = hit_mask` [S1, unofficial].
   - Set initial intervals: `iv_t_lo = t_min`, `iv_t_hi = t_max` for valid rays [S1, unofficial].
   - `ray_idx` contains indices of valid rays [S1, unofficial].

2. **Bisection Loop:**
   - While `len(ray_idx) > 0` and current `depth < max_depth`:
     a. **Front-to-Back Pruning:**
        - Remove intervals that are entirely beyond the closest known hit [S1, unofficial].
        - `keep = iv_t_lo[ray_idx] < result_t[ray_idx]` [S1, unofficial].
        - Update `ray_idx`, `iv_t_lo`, `iv_t_hi` using the `keep` mask [S1, unofficial].
        - If `len(ray_idx) == 0`, break loop [S1, unofficial].
     
     b. **Clip Far End:**
        - Ensure intervals do not extend past the closest known hit [S1, unofficial].
        - `iv_t_hi[ray_idx] = min(iv_t_hi[ray_idx], result_t[ray_idx])` [S1, unofficial].
     
     c. **Interval Evaluation & Sign Change Detection:**
        - For each active interval $[t_{lo}, t_{hi}]$, use the inclusion function to compute bounds on $f(t)$ [S1, unofficial].
        - Identify intervals where the sign change occurs (i.e., the interval bounds contain 0, or $f(t_{lo})$ and $f(t_{hi})$ have opposite signs, depending on the specific inclusion function implementation) [S1, unofficial].
        - `sign_change = inclusion.contains_zero(iv_t_lo[ray_idx], iv_t_hi[ray_idx])` [S1, unofficial].
        - `valid_ray = ray_idx[sign_change]` [S1, unofficial].
        - `valid_t_lo = iv_t_lo[sign_change]` [S1, unofficial].
        - `valid_t_hi = iv_t_hi[sign_change]` [S1, unofficial].
        - If no sign changes, break loop (no more hits to find) [S1, unofficial].
     
     d. **Convergence Check:**
        - Check if interval width is below threshold: `converged = (valid_t_hi - valid_t_lo) < threshold` [S1, unofficial].
        - If converged, record hit:
          - `result_hit[valid_ray] = True` [S1, unofficial].
          - `result_t[valid_ray] = (valid_t_lo + valid_t_hi) * 0.5` [S1, unofficial].
          - `result_t_lo[valid_ray] = valid_t_lo` [S1, unofficial].
          - `result_t_hi[valid_ray] = valid_t_hi` [S1, unofficial].
          - `result_depth[valid_ray] = depth + 1` [S1, unofficial].
        - Mark these rays as converged to remove them from further bisection [S1, unofficial].
     
     e. **Bisect Step:**
        - For non-converged valid rays:
          - `t_mid = (valid_t_lo + valid_t_hi) * 0.5` [S1, unofficial].
          - Create new intervals:
            - `new_ray_idx = [valid_ray, valid_ray]` [S1, unofficial].
            - `new_t_lo = [valid_t_lo, t_mid]` [S1, unofficial].
            - `new_t_hi = [t_mid, valid_t_hi]` [S1, unofficial].
          - Update `iv_t_lo`, `iv_t_hi`, `ray_idx` with these new intervals [S1, unofficial].
        - Increment `depth` [S1, unofficial].

3. **Post-Processing (Optional Newton Refinement):**
   - For rays marked as hit, optional Newton refinement can be applied to converge the hit point to machine precision [S1, unofficial]. This involves evaluating $f$ and its gradient $\nabla f$ at the hit position and iterating $t_{new} = t_{old} - f(t_{old}) / (\nabla f(t_{old}) \cdot direction)$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class TraceResult:
    hit: np.ndarray
    t: np.ndarray
    position: np.ndarray
    normal: np.ndarray
    depth_count: np.ndarray
    t_lo: np.ndarray
    t_hi: np.ndarray

def bisect_trace(origins, directions, t_min, t_max, hit_mask, inclusion, f, 
                 max_depth=40, threshold=1e-5):
    R = len(origins)
    result_t = np.full(R, np.inf)
    result_hit = np.zeros(R, dtype=bool)
    result_t_lo = np.zeros(R)
    result_t_hi = np.zeros(R)
    result_depth = np.zeros(R, dtype=np.int32)
    
    # Active intervals
    ray_idx = np.where(hit_mask)[0]
    iv_t_lo = t_min[ray_idx]
    iv_t_hi = t_max[ray_idx]
    
    for depth in range(max_depth):
        if len(ray_idx) == 0:
            break
            
        # Front-to-back pruning: remove intervals beyond closest hit
        keep = iv_t_lo < result_t[ray_idx]
        ray_idx = ray_idx[keep]
        iv_t_lo = iv_t_lo[keep]
        iv_t_hi = iv_t_hi[keep]
        
        if len(ray_idx) == 0:
            break
            
        # Clip far end to closest hit
        iv_t_hi = np.minimum(iv_t_hi, result_t[ray_idx])
        
        # Evaluate inclusion function for sign change
        sign_change = inclusion.contains_zero(iv_t_lo, iv_t_hi)
        valid_ray = ray_idx[sign_change]
        valid_t_lo = iv_t_lo[sign_change]
        valid_t_hi = iv_t_hi[sign_change]
        
        if len(valid_ray) == 0:
            break
            
        # Check convergence
        converged = (valid_t_hi - valid_t_lo) < threshold
        
        # Record hits for converged intervals
        hit_indices = valid_ray[converged]
        if len(hit_indices) > 0:
            mid_t = (valid_t_lo[converged] + valid_t_hi[converged]) * 0.5
            result_hit[hit_indices] = True
            result_t[hit_indices] = mid_t
            result_t_lo[hit_indices] = valid_t_lo[converged]
            result_t_hi[hit_indices] = valid_t_hi[converged]
            result_depth[hit_indices] = depth + 1
            
        # Prepare next iteration for non-converged intervals
        not_converged = ~converged
        if np.any(not_converged):
            valid_ray_nc = valid_ray[not_converged]
            valid_t_lo_nc = valid_t_lo[not_converged]
            valid_t_hi_nc = valid_t_hi[not_converged]
            
            t_mid = (valid_t_lo_nc + valid_t_hi_nc) * 0.5
            
            # Concatenate new intervals
            new_ray_idx = np.concatenate([valid_ray_nc, valid_ray_nc])
            new_t_lo = np.concatenate([valid_t_lo_nc, t_mid])
            new_t_hi = np.concatenate([t_mid, valid_t_hi_nc])
            
            # Update active intervals
            ray_idx = new_ray_idx
            iv_t_lo = new_t_lo
            iv_t_hi = new_t_hi
        else:
            ray_idx = np.array([], dtype=int)
            
    # Compute positions
    positions = origins + result_t[:, np.newaxis] * directions
    
    # Compute normals using centered finite differences [G]
    normals = np.zeros_like(directions)
    hit_positions = positions[result_hit]
    if len(hit_positions) > 0:
        eps = 1e-6
        grad = np.zeros_like(hit_positions)
        for i in range(3):
            offset = np.zeros_like(hit_positions)
            offset[:, i] = eps
            f_pos, _ = f(hit_positions + offset)
            f_neg, _ = f(hit_positions - offset)
            grad[:, i] = (f_pos - f_neg) / (2 * eps)
        norms = np.linalg.norm(grad, axis=1, keepdims=True)
        norms = np.maximum(norms, 1e-12)
        normals[result_hit] = grad / norms
        
    return TraceResult(
        hit=result_hit,
        t=result_t,
        position=positions,
        normal=normals,
        depth_count=result_depth,
        t_lo=result_t_lo,
        t_hi=result_t_hi
    )
```

**Parameters:**
- `max_depth`: Maximum number of bisection iterations allowed per ray. Default value is 40 [S1, unofficial].
- `threshold`: The convergence width threshold for the bisection interval. The loop terminates when $(t_{hi} - t_{lo}) < threshold$. Default value is $10^{-5}$ [S1, unofficial].
- `batch_size`: Number of rays to process per batch. `None` processes all rays at once. Positive integer splits rays into batches to limit peak memory usage [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision/Convergence:** The algorithm uses a threshold (default $10^{-5}$) to determine convergence. If the interval width is below this threshold, the ray is considered converged and removed from further processing [S1, unofficial].
- **Smooth Blends:** Reduced efficiency in the presence of shapes with many smooth blends, as interval evaluations may remain wide for many iterations [S2].
- **Self-Intersection:** The algorithm handles self-intersecting surfaces (e.g., Whitney Umbrella) by finding the nearest intersection [S1, unofficial].

## Subtopics

### Batched Bisection Ray Tracing
Splits ray tracing into batches to limit peak memory usage. Processes each batch independently and merges results. If `batch_size` is `None` or $\ge R$, process all rays in a single call to `bisect_trace`. Otherwise, loop over `start` in `range(0, R, batch_size)`, `end = min(start + batch_size, R)`. Slice `origins`, `directions`, `t_min`, `t_max`, `hit_mask`. Call `bisect_trace` on the slice. Apply Newton refinement if `newton_iter > 0`. Merge results into global result arrays [S1, unofficial]. Parameter: `batch_size`: Number of rays to process per batch. `None` processes all rays at once. Positive integer splits rays into batches [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(R \cdot D)$, where $R$ is the number of rays and $D$ is the average number of bisection steps. Each step involves interval evaluation and bisection [S1, unofficial].
- **Memory Complexity:** $O(R)$ for storing active intervals and results. Batched processing reduces peak memory usage to $O(B)$, where $B$ is the batch size [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of rays. Batched processing allows handling larger scenes with limited memory [S1, unofficial].
- **Strengths:** Guarantees no surface features are missed, unlike fixed-step ray marching which can skip thin features or high-frequency detail [S1, unofficial]. Robust for arbitrary function-based field definitions [S2].
- **Limitations:** Reduced efficiency in the presence of shapes with many smooth blends [S2]. Interval arithmetic can be computationally expensive compared to direct evaluation [S3, unofficial].
- **When to Use:** When robustness and feature preservation are critical, and performance is secondary. Suitable for high-frequency surfaces, thin shells, and complex CAD meshes [S1, unofficial].
- **When Not to Use:** When performance is critical and the surface is simple or smooth. When memory is extremely constrained and batched processing is not feasible [S3, unofficial].

## Background
Originated from the need for robust ray tracing of implicit surfaces. The method relies on iterative interval refinement and interval/affine arithmetics to compute bounds on field variation [S2]. It was implemented in the `isoray` library by narnia-ai-mason [S1, unofficial]. The approach is discussed in the context of fast ray tracing of scale-invariant integral surfaces, where it is noted as a robust alternative to sphere tracing for arbitrary function-based field definitions [S2].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
