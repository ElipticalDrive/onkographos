# Bisection Trace with Gradient Inclusion

Also called: Level-Synchronized Interval Bisection

## Overview
Bisection Trace with Gradient Inclusion is a robust ray casting method that renders the zero-level-set of an implicit function $f: \mathbb{R}^3 \to \mathbb{R}$ by combining level-synchronized bisection with gradient-based inclusion functions [S1, unofficial]. It solves the problem of missing surface features during ray traversal by using interval arithmetic to guarantee that no zero-crossings are pruned incorrectly. It differs from its siblings by leveraging gradient information to compute tighter bounds on the function's range over axis-aligned boxes, enabling more efficient pruning than Lipschitz-based methods while maintaining robustness [S1, unofficial].

## How it works
**Inputs:**
- Ray origins $\mathbf{o} \in \mathbb{R}^3$ and directions $\mathbf{d} \in \mathbb{R}^3$ (normalized).
- Ray segment bounds $t_{min}, t_{max} \in \mathbb{R}$.
- Implicit function $f(\mathbf{x})$ and its gradient $\nabla f(\mathbf{x})$.
- Parameters: `max_depth` (integer), `threshold` (float), `lipschitz` (float or 'auto'), `batch_size` (integer or None).

**Outputs:**
- Hit mask: Boolean array indicating which rays intersected the surface.
- Hit distances: Float array of $t$ values for intersected rays.

**Data Structures:**
- **Affine Form (AA):** Represents the range of $f$ over an interval. Defined as a tuple $(x_0, \xi, \delta)$ where:
  - $x_0$: Center value.
  - $\xi$: Vector of coefficients for independent variables (rays).
  - $\delta$: Scalar bound for dependent error (function evaluation uncertainty).
  - The range is $[x_0 - \|\xi\|_1 - \delta, x_0 + \|\xi\|_1 + \delta]$.
- **Inclusion Function:** An object implementing the `evaluate(lo, hi)` method, returning conservative bounds $[f_{min}, f_{max}]$ for $f$ over the box defined by $[\mathbf{o} + t_{lo}\mathbf{d}, \mathbf{o} + t_{hi}\mathbf{d}]$.

**Steps:**
1. **Initialize Inclusion Function:** Create a `GradientInclusion` instance using $f$, $\nabla f$, and the Lipschitz constant $L$. If `lipschitz` is 'auto', estimate $L$ by sampling $\|\nabla f\|$ over the bounding box using automatic differentiation [S1, unofficial].
2. **Batch Rays:** Divide rays into batches of size `batch_size` (or all rays if None) to manage memory.
3. **Level-Synchronized Bisection Loop:** For each batch:
   a. Initialize active rays with $t_{curr} = t_{min}$ and depth $d=0$.
   b. While active rays exist and $d < \text{max\_depth}$:
      i. **Evaluate Inclusion:** For each active ray, compute the box $[\mathbf{o} + t_{curr}\mathbf{d}, \mathbf{o} + (t_{curr} + \Delta t)\mathbf{d}]$ where $\Delta t$ is the current step size (initially $t_{max}-t_{min}$). Call `inclusion.evaluate(lo, hi)` to get $[f_{min}, f_{max}]$.
      ii. **Prune:** If $f_{max} < 0$ or $f_{min} > 0$, the interval cannot contain a zero crossing. Mark the ray as missed and remove from active set.
      iii. **Check Convergence:** If $f_{max} - f_{min} < \text{threshold}$, the interval is sufficiently small. Record $t_{curr}$ as the hit and remove from active set.
      iv. **Bisect:** For remaining active rays, bisect the interval: $t_{new} = t_{curr} + \Delta t / 2$. Update $t_{curr} = t_{new}$ and $\Delta t = \Delta t / 2$. Increment $d$.
4. **Newton Refinement (Optional):** For rays that hit, apply `newton_iter` steps of Newton's method to refine the hit location using $f$ and $\nabla f$ [S1, unofficial].

**Formulas:**
- **Affine Arithmetic Addition:** $a + b = (x_0^a + x_0^b, \xi^a + \xi^b, \delta^a + \delta^b)$ [S1, unofficial].
- **Affine Arithmetic Multiplication by Scalar $\alpha$:** $\alpha \cdot a = (\alpha x_0, \alpha \xi, |\alpha| \delta)$ [S1, unofficial].
- **Inclusion Bound:** $f(\mathbf{x}) \in [x_0 - \|\xi\|_1 - \delta, x_0 + \|\xi\|_1 + \delta]$ [S1, unofficial].

**Parameters:**
- `max_depth`: Typical value 10-20. Limits bisection iterations to prevent infinite loops.
- `threshold`: Typical value $10^{-4}$ to $10^{-6}$. Convergence criterion for interval width.
- `lipschitz`: If 'auto', computed via gradient sampling. Otherwise, a user-provided constant.
- `batch_size`: Controls memory usage. Larger batches improve throughput but require more GPU memory.

**Edge Cases:**
- **Degenerate Rays:** Rays with zero direction or $t_{min} \ge t_{max}$ should be handled before tracing.
- **Precision:** Affine arithmetic can suffer from dependency problem. Use higher-order forms or interval arithmetic fallback if bounds are too loose [S1, unofficial].
- **Lipschitz Estimation:** If 'auto', ensure sufficient samples to capture max gradient norm.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$ where $N$ is the number of rays and $D$ is the average bisection depth. Each bisection step involves an inclusion function evaluation, which is $O(1)$ per ray if precomputed, or $O(M)$ if $M$ is the number of variables in the affine form.
- **Memory Complexity:** $O(N \cdot M)$ for storing affine forms for all active rays.
- **Performance:** Faster than pure interval arithmetic due to tighter bounds from gradient inclusion. Slower than simple bisection due to overhead of affine arithmetic.
- **Strengths:** Robustness against missing features; efficient pruning compared to coarse interval methods.
- **Limitations:** Higher computational cost per step; dependency problem in affine arithmetic can lead to overestimation.
- **When to Use:** When robustness is critical and the implicit function is differentiable.
- **When Not to Use:** For non-differentiable functions or when performance is paramount and some feature loss is acceptable.

## Background
The method is based on level-synchronized bisection ray casting, which ensures all rays are processed in lockstep to maximize parallel efficiency on GPUs [S1, unofficial]. Gradient inclusion strategies were introduced to improve the tightness of bounds in interval arithmetic, reducing the number of bisection steps required [S1, unofficial]. The implementation is part of the `iso-surface-rendering` library, which provides a modular framework for robust volume rendering [S1, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
