# Adaptive Lipschitz Constant (ALC)

Also called: BVH-based Adaptive Lipschitz Constant, Nested BVH Step Size Optimization

## Overview
Adaptive Lipschitz Constant (ALC) is a sphere tracing enhancement that uses a nested Bounding Volume Hierarchy (BVH) to compute tighter, adaptive Lipschitz constants for step size calculation, rather than using the global sum Lipschitz constant [S1]. It solves the problem of overly conservative step sizes in large scenes where the naive global Lipschitz constant $\lambda_{F_n}$ becomes very large, limiting efficiency [S1].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) [G].
- Current position $t_0$ along the ray [G].
- A nested BVH structure representing the implicit function $F_n = \sum_{i=1}^n f_i$ [S1].
- For each BVH level $i \in [1, n]$: the value of the partial sum at $t_0$, denoted $\text{BVH}_i(t_0)$ [S1]; the Lipschitz constant for the partial sum, $\lambda_{\text{BVH}_i} = \lambda_{F_i}$ [S1].

**Outputs:**
- The next safe step size $\Delta t$ to advance the ray [G].

**Data Structures:**
- **BVH Nodes:** Store bounding volumes, Lipschitz constants, and partial sum values at the current ray position [S1].
- **Sum Elements:** The functions $f_i$ are pre-sorted from higher to lower range of values to achieve close-to-maximal steps with low overhead [S1].

**Method:**
1. **Identify Minimum Level:** Determine $i_{min}$, the first BVH level where the ray is outside the bounding volume. For all levels $i \ge i_{min}$, the ray is outside the isosurface of $BVH_i$, making the intersection with $BVH_i$'s boundary a safe step target [S1].
2. **Construct Inclusion Functions:** For each valid BVH level $i$, construct a forward inclusion function: $[\text{BVH}_{\pm i}](t) = \text{BVH}_{\pm i}(t_0) \pm \lambda_{\text{BVH}_i}(t - t_0)$, where $\lambda_{\text{BVH}_i} = \lambda_{F_i}$ [S1].
3. **Combine Bounds:** Combine the inclusion functions from all valid BVH levels by taking their intersection to narrow the range of possible values for $F_n(t)$: $F_n(t) \in \bigcap_{i=1}^n [\text{BVH}_{i\pm}](t)$ [S1].
4. **Compute Step Candidates:** Calculate candidate step sizes for each BVH level $i$ using the lower bound of the combined inclusion function: $F_{n-1}(t) = \max_{i \in [1, n]} (\text{BVH}_{i-1}(t_0) - \lambda_{\text{BVH}_i}(t - t_0))$ [S1]. Solving $F_{n-1}(t) = 0$ yields the candidate step $\Delta t_i = \text{BVH}_{i-1}(t_0) / \lambda_{\text{BVH}_i}$ [S1].
5. **Select Final Step:** Select the largest (best) candidate as the final step size, which corresponds to the minimum of the individual upper-bound roots to satisfy the intersection constraint [S1].
6. **Advance Ray:** Move the ray origin to $t_0 + \Delta t$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_alc_step(current_t, bvh_levels):
    """
    Compute the next safe step size using Adaptive Lipschitz Constant.
    
    Args:
        current_t: float, current distance along the ray.
        bvh_levels: list of dicts, each containing:
            - 'value': float, BVH_i(t_0)
            - 'lambda': float, Lipschitz constant lambda_BVH_i
            
    Returns:
        float, the next safe step size.
    """
    best_step = float('inf')
    
    for level in bvh_levels:
        val = level['value']
        lam = level['lambda']
        
        if lam <= 0:
            continue
            
        candidate_step = val / lam
        
        # The source states to "select the largest (best) candidate".
        # Mathematically, intersecting inclusion functions requires taking 
        # the minimum of the individual roots to guarantee safety [S1].
        if candidate_step > 0 and candidate_step < best_step:
            best_step = candidate_step
            
    return best_step if best_step != float('inf') else 0.0
```

**Parameters:**
- **Pre-sorting:** Sum elements are pre-sorted from higher to lower range of values to achieve close-to-maximal steps with low overhead [S1].
- **Dynamic Re-ordering:** Not used. It is not GPU-friendly because it requires storing each element in an $O(n)$ array and sorting dynamically at every step [S1].

**Edge Cases and Pitfalls:**
- **Precision:** If $\lambda_{\text{BVH}_i}$ is very small, division by zero or numerical instability may occur. Guard against $\lambda \le \epsilon$ [G].
- **Degenerate Input:** If the ray is inside a BVH level, the inclusion function logic for "outside" steps may not apply directly. The method assumes the ray is outside the target isosurface for the step calculation [G].
- **GPU Performance:** Dynamic re-ordering of sum elements is avoided due to high memory and sorting overhead on GPUs [S1].

## Subtopics

### [Sub-stepping](1937-sub-stepping.md)
Sub-stepping is an optimization for adaptive ray marching that guarantees non-decreasing step sizes by performing intermediate sub-steps along the ray as intermediate field sums are computed. It solves the problem of "bad candidates" in standard Adaptive Lipschitz Constant (ALC) methods, where computing additional field components $f_i$ can unpredictably reduce the step size, leading to wasted computation. Unlike standard ALC, which calculates the next step based on the current position and all available bounds, Sub-stepping physically advances the ray to validate bounds, ensuring that each additional level of detail contributes positively or neutrally to the step size.

### Per-ray and per-primitive Lipschitz constant computation
Computes a tighter Lipschitz constant for sphere-tracing by evaluating it per primitive and per ray, mitigating performance loss from varying primitive radii [S2]. For each ray, find the primitive point with the smallest radius whose kernel support intersects the ray [S2]. Solve the polynomial equation for intersection roots to determine the smallest radius along the ray [S2]. The kernel support intersection condition is:
$$ \|\mathbf{p} - (\mathbf{p}^T \mathbf{d}) \mathbf{d}\|^2 - \sigma^2(\tau_0 + s\Delta\tau)^2 \leq 0 $$
where $\mathbf{p} = \mathbf{m} - \mathbf{s}\mathbf{u}$, $\mathbf{d}$ is the ray direction, $\sigma$ is the kernel scale, $\tau_0$ is the base radius, $\Delta\tau$ is the radius variation step, and $s$ is the ray parameter [S2]. The parameter $\sigma$ is the kernel scale factor used in the intersection polynomial [S2].

## Operating characteristics
- **Time Complexity:** $O(n)$ per step to evaluate all BVH levels and compute inclusion functions, where $n$ is the number of levels in the BVH [S1].
- **Memory Complexity:** $O(n)$ to store BVH levels and their Lipschitz constants, with no dynamic arrays required, making it GPU-friendly [S1].
- **Measured Performance:** Improves efficiency over naive global Lipschitz constant methods, especially when $n$ is large and $\lambda_{F_n}$ would otherwise be very big [S1].
- **Strengths:** Tighter bounds lead to larger steps. GPU-friendly due to lack of dynamic sorting [S1].
- **Limitations:** Requires a nested BVH structure. Pre-sorting may not always yield optimal steps, though it is close-to-optimal for functions like FBM [S1].
- **When to Use:** When rendering procedural volumetric noise or large detailed scenes where the global Lipschitz constant is too conservative [S1].
- **When Not to Use:** When the BVH structure is not available or too expensive to build [S1].

## Background
Originating from the work on fast sphere tracing of procedural volumetric noise, ALC leverages the bounded nature of functions through a BVH to make bigger and cheaper steps [S1]. It builds on the intuition of Worley and Hart regarding dynamic re-ordering but adapts it for GPU-friendly static pre-sorting [S1]. The method is described as an enhancement to sphere tracing using nested BVH levels [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
