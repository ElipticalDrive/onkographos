# Bisection Trace with Gradient Inclusion

Also called: Level-Synchronized Interval Bisection

## Overview
Bisection Trace with Gradient Inclusion is a robust ray casting method that renders the zero-level-set of an implicit function $f: \mathbb{R}^3 \to \mathbb{R}$ by combining level-synchronized bisection with gradient-based inclusion functions [S1, unofficial]. It solves the problem of missing surface features during ray traversal by using interval arithmetic to guarantee that no zero-crossings are pruned incorrectly [S1, unofficial].

## How it works
**Inputs:** Ray origins $\mathbf{o} \in \mathbb{R}^3$, directions $\mathbf{d} \in \mathbb{R}^3$, segment bounds $t_{\min}, t_{\max}$, implicit function $f(\mathbf{x})$, and parameters `max_depth`, `threshold`, `lipschitz`, `batch_size` [S1, unofficial].
**Outputs:** `hit_mask` (boolean array) and `hit_distances` (float array) [S1, unofficial].
**Data Structures:** 
- `AffineForm(x0, xi, delta)`: Represents the range of $f$ over an interval, where $x_0$ is the center, $\xi$ is the vector of noise coefficients, and $\delta$ is the scalar error bound [S1, unofficial].
- `InclusionFunction`: Implements `evaluate(lo, hi)` returning conservative bounds $[f_{\min}, f_{\max}]$ for $f$ over the axis-aligned box defined by $[\mathbf{o} + t_{lo}\mathbf{d}, \mathbf{o} + t_{hi}\mathbf{d}]$ [S1, unofficial].

**Steps:**
1. **Initialize Inclusion Function:** Create a `GradientInclusion` instance using $f$, $\nabla f$, and the Lipschitz constant $L$ [S1, unofficial].
2. **Batch Rays:** Divide rays into batches of size `batch_size` (or all rays if `None`) to manage memory [S1, unofficial].
3. **Level-Synchronized Bisection Loop:** For each batch, initialize active rays with $t_{\text{curr}} = t_{\min}$ and depth $d=0$. While active rays exist and $d < \text{max\_depth}$: evaluate inclusion bounds for the current interval, prune rays where $f_{\max} < 0$ or $f_{\min} > 0$, record hits where $f_{\max} - f_{\min} < \text{threshold}$, and bisect remaining intervals [S1, unofficial].
4. **Newton Refinement:** Apply `newton_iter` steps of Newton's method to refine hit locations using $f$ and $\nabla f$ [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
- `max_depth`: Maximum bisection depth [S1, unofficial].
- `threshold`: Bisection convergence threshold [S1, unofficial].
- `lipschitz`: Lipschitz constant for the inclusion function. If `'auto'`, estimated by sampling gradient norms over the bounding box via automatic differentiation [S1, unofficial].
- `batch_size`: Number of rays to process per batch. `None` processes all at once [S1, unofficial].

**Edge Cases:**
- **Degenerate Rays:** Rays with zero direction or $t_{\min} \ge t_{\max}$ should be handled before tracing [G].
- **Precision:** Affine arithmetic can suffer from the dependency problem, leading to overestimation [G].
- **Lipschitz Estimation:** If `'auto'`, ensure sufficient samples to capture the maximum gradient norm [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_rays(origins, directions, t_min, t_max, hit_mask, inclusion, f_np, max_depth, threshold, newton_iter, batch_size):
    import numpy as np
    
    hit_mask[:] = False
    hit_distances = np.full_like(origins[:, 0], np.inf)
    
    batch_size = batch_size if batch_size is not None else len(origins)
    for start in range(0, len(origins), batch_size):
        end = min(start + batch_size, len(origins))
        b_origins = origins[start:end]
        b_dirs = directions[start:end]
        b_t_min = t_min[start:end]
        b_t_max = t_max[start:end]
        
        t_curr = b_t_min.copy()
        t_step = (b_t_max - b_t_min).copy()
        active = np.ones(len(b_origins), dtype=bool)
        depth = 0
        
        while np.any(active) and depth < max_depth:
            lo = b_origins + t_curr[:, None] * b_dirs
            hi = b_origins + (t_curr + t_step)[:, None] * b_dirs
            f_min, f_max = inclusion.evaluate(lo, hi)
            
            missed = (f_max < 0) | (f_min > 0)
            active[missed] = False
            
            converged = (f_max - f_min) < threshold
            hit_mask[start:end][active & converged] = True
            hit_distances[start:end][active & converged] = t_curr[active & converged]
            active[converged] = False
            
            t_curr[active] += t_step[active] / 2.0
            t_step[active] /= 2.0
            depth += 1
            
        # [G] Newton refinement: apply newton_iter steps to refine hit_distances using f_np and its gradient
```

## Subtopics

### [Interval Arithmetic Inclusion Strategies](1410-interval-arithmetic-inclusion-strategies.md)
A family of methods used within level-synchronized bisection ray casting to compute valid bounds for an implicit function over axis-aligned boxes. They are distinguished by their specific approach to bounding function variation, using either black-box gradient sampling or symbolic analytic formulas.

## Operating characteristics
Not given in the sources.

## Background
The method builds on level-synchronized bisection ray casting, which processes all active rays in lockstep by depth level to maximize parallel efficiency [S1, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
