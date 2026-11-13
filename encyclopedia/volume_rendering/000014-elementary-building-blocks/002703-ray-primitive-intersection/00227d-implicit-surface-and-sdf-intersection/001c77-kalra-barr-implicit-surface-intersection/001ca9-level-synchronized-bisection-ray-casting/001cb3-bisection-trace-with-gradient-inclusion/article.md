# Bisection Trace with Gradient Inclusion

Also called: Level-Synchronized Interval Bisection

## Overview
Bisection Trace with Gradient Inclusion is a robust ray casting algorithm that renders the zero-level-set of an implicit function $f: \mathbb{R}^3 \to \mathbb{R}$ by combining level-synchronized bisection with affine arithmetic inclusion functions. It solves the problem of missing surface features in implicit rendering by using rigorous interval bounds to prune empty space and guarantee convergence. It differs from standard bisection by using gradient-based Lipschitz constants to tighten inclusion bounds, and from pure interval arithmetic by leveraging affine forms to reduce dependency errors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Ray origins $\mathbf{o}$, directions $\mathbf{d}$, near/far planes $t_{min}, t_{max}$, implicit function $f$, maximum bisection depth $D_{max}$, convergence threshold $\tau$, and Lipschitz constant $L$ (or 'auto').
- **Outputs:** Hit mask (boolean array), hit distances $t$, and surface normals $\nabla f(\mathbf{p})$.

**Data Structures:**
- **AffineForm:** Represents an affine arithmetic interval. Fields:
  - `x0`: Center value (scalar or vector).
  - `xi`: Noise symbols coefficients (vector of size $N$, where $N$ is the number of independent noise symbols).
  - `delta`: Remainder bound (scalar).
  - Layout: `AffineForm(x0: np.ndarray, xi: np.ndarray, delta: np.ndarray)`.
- **InclusionFunction:** Protocol defining `evaluate(lo, hi)` which returns `(lo_f, hi_f)` bounds.

**Parameters:**
- `max_depth`: Integer, typical value 10–20.
- `threshold`: Float, typical value $10^{-4}$ to $10^{-6}$.
- `lipschitz`: Float or 'auto'. If 'auto', estimated via sampling gradient norms.
- `batch_size`: Integer or None. Processes rays in chunks to manage memory.

**Steps:**
1. **Initialize Inclusion Function:** Create an instance of `GradientInclusion` using the implicit function $f$ and Lipschitz constant $L$. If $L$ is 'auto', estimate it by sampling $\|\nabla f\|$ over the bounding box using automatic differentiation [S1, unofficial].
2. **Batch Processing:** Divide rays into batches of size `batch_size`. If `batch_size` is None, process all rays.
3. **Ray Tracing Loop:** For each batch:
   a. **Interval Evaluation:** For each active ray, define the current interval $[t_{lo}, t_{hi}]$. Compute the inclusion function $[f]([t_{lo}, t_{hi}])$ using affine arithmetic. This involves mapping ray parameters to affine forms and evaluating $f$ symbolically [S1, unofficial].
   b. **Pruning:** If $0 \notin [f]([t_{lo}, t_{hi}])$, the interval is empty. Mark the ray as missed or inactive.
   c. **Bisection:** If $[f]$ contains zero and $t_{hi} - t_{lo} > \tau$:
      i. Compute midpoint $t_{mid} = (t_{lo} + t_{hi}) / 2$.
      ii. Evaluate $f(t_{mid})$ using standard floating-point arithmetic for the hit point.
      iii. Update interval: If $f(t_{mid})$ has the same sign as $f(t_{lo})$, set $t_{lo} = t_{mid}$; else $t_{hi} = t_{mid}$.
   d. **Convergence:** If $t_{hi} - t_{lo} \le \tau$, record $t_{hit} = (t_{lo} + t_{hi}) / 2$ and compute normal $\nabla f(\mathbf{p})$ via gradient [S1, unofficial].
4. **Repeat** until all rays are resolved or `max_depth` is reached.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def trace_rays(origins, directions, t_min, t_max, hit_mask, inclusion_func, f_func, max_depth, threshold, newton_iter, batch_size):
    """
    Trace rays using level-synchronized bisection with gradient inclusion.
    """
    n_rays = len(origins)
    t_lo = np.full(n_rays, t_min)
    t_hi = np.full(n_rays, t_max)
    active = np.ones(n_rays, dtype=bool)
    
    # Pre-estimate Lipschitz if needed (handled in initialization)
    
    for _ in range(max_depth):
        if not np.any(active):
            break
            
        # Process in batches to manage memory
        for start in range(0, n_rays, batch_size if batch_size else n_rays):
            end = min(start + (batch_size if batch_size else n_rays), n_rays)
            batch_active = active[start:end]
            
            if not np.any(batch_active):
                continue
                
            # Get current intervals for active rays
            lo_vals = t_lo[start:end][batch_active]
            hi_vals = t_hi[start:end][batch_active]
            
            # Evaluate inclusion function
            # Returns conservative bounds [f_lo, f_hi] for f over [lo, hi]
            f_lo, f_hi = inclusion_func.evaluate(lo_vals, hi_vals)
            
            # Prune intervals that do not contain zero
            # Zero is in [f_lo, f_hi] if f_lo <= 0 <= f_hi
            contains_zero = (f_lo <= 0) & (f_hi >= 0)
            
            # Update active status for this batch
            # Note: In a full implementation, we need to track which rays are still active
            # For simplicity, we assume 'active' is updated globally after checking all rays
            # Here we just mark which ones to keep bisecting
            keep_bisect = contains_zero & (hi_vals - lo_vals > threshold)
            
            # Update intervals for rays to bisect
            mid = (lo_vals + hi_vals) / 2.0
            f_mid = f_func(mid) # Standard float evaluation for hit point
            
            # Update t_lo and t_hi based on sign of f_mid
            # If f(mid) > 0, root is in [lo, mid], so hi = mid
            # If f(mid) < 0, root is in [mid, hi], so lo = mid
            # Handle f(mid) == 0 as hit
            
            # Apply updates only to rays that are still active and within threshold
            # This is a simplified step; real implementation tracks per-ray state
            
            # For the sake of the algorithm description, we assume the inclusion
            # function handles the pruning and the bisection updates the bounds.
            # The actual update logic depends on the specific sign check.
            
            # Update global active mask based on pruning
            # This requires careful indexing in a real implementation
            
            # Convergence check
            converged = (hi_vals - lo_vals) <= threshold
            # Mark as hit if converged and active
            
    return hit_mask, t_lo, t_hi

class GradientInclusion:
    def __init__(self, f_np, lipschitz='auto'):
        self.f_np = f_np
        if lipschitz == 'auto':
            # Estimate Lipschitz constant by sampling gradient norms
            # This is a placeholder for the actual estimation logic
            self.lipschitz = self._estimate_lipschitz()
        else:
            self.lipschitz = lipschitz
            
    def _estimate_lipschitz(self):
        # Sample gradient norms over bounding box
        # Return max norm
        return 1.0 # Placeholder
        
    def evaluate(self, lo, hi):
        # Use affine arithmetic to compute bounds
        # This is a simplified interface
        # In reality, this maps lo/hi to AffineForm and evaluates f
        return lo, hi # Placeholder
```

**Edge Cases and Pitfalls:**
- **Precision:** Affine arithmetic can suffer from dependency errors if not managed. Use interval arithmetic fallback if affine bounds are too loose [S1, unofficial].
- **Boundaries:** Ensure $t_{min}$ and $t_{max}$ are valid and do not cause overflow in interval evaluation.
- **Degenerate Input:** If $f$ is constant, the inclusion function should return a single value, and the algorithm should handle it gracefully (e.g., no hit or full hit) [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of rays and $D$ is the maximum bisection depth. Each step involves an inclusion function evaluation, which is more expensive than a single float evaluation but reduces the number of bisection steps needed.
- **Memory Complexity:** $O(N \cdot S)$, where $S$ is the number of noise symbols in the affine arithmetic. This can be significant for complex functions.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of rays. Batch processing helps manage memory.
- **Strengths:** Robustness against missing surface features; guaranteed convergence within bounds.
- **Limitations:** Higher computational cost per step due to affine arithmetic; memory overhead for noise symbols.
- **When to use:** When robustness is critical and performance is secondary; for complex implicit functions where standard bisection might miss details.
- **When not to use:** For real-time applications with strict performance constraints; for simple implicit functions where standard bisection is sufficient.

## Background
Originated in the context of robust iso-surface rendering, specifically in the `iso-surface-rendering` repository by narnia-ai-mason. It builds on the principles of level-synchronized bisection ray casting and incorporates affine arithmetic for tighter bounds [S1, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
