# Adaptive Lipschitz Constant (ALC)

Also called: BVH-based Adaptive Lipschitz Constant, Nested BVH Step Size Optimization

## Overview
Adaptive Lipschitz Constant (ALC) is a step size optimization for distance-field ray marching that uses a nested Bounding Volume Hierarchy (BVH) to compute tighter, adaptive Lipschitz constants for each step, rather than relying on the global sum Lipschitz constant. It solves the problem of overly conservative step sizes in large scenes where the naive global Lipschitz constant $\lambda_{F_n}$ becomes very large, limiting efficiency. Unlike [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) which may use a single global bound, ALC leverages the geometric containment properties of the BVH to find intermediate safe targets. It differs from [Gradient-based Sphere Tracing with Lipschitz Constraints](1936-gradient-based-sphere-tracing-with-lipschitz-constraints.md) by using structural BVH bounds rather than local gradient estimates, and from [Lipschitz Step Size Ray Marching](3729-lipschitz-step-size-ray-marching.md) by explicitly combining inclusion functions from multiple BVH levels.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
- Current position $t_0$ along the ray.
- A nested BVH structure representing the implicit function $F_n = \sum_{i=1}^n f_i$.
- For each BVH level $i \in [1, n]$:
    - The value of the partial sum at $t_0$, denoted $\text{BVH}_i(t_0)$ (or $F_i(t_0)$).
    - The Lipschitz constant for the partial sum, $\lambda_{\text{BVH}_i} = \lambda_{F_i}$.
    - The inclusion function bounds $[\text{BVH}_{i\pm}](t)$, typically defined by the bounding volume of level $i$.

**Outputs:**
- The next safe step size $\Delta t$ to advance the ray.

**Data Structures:**
- **BVH Node:** Contains bounding volume (e.g., AABB), Lipschitz constant $\lambda$, and value at current ray position.
- **Sum Elements:** The functions $f_i$ are pre-sorted. ALC does not use dynamic re-ordering [S1].

**Method:**
1.  **Identify Minimum Level:** Determine $i_{min}$, the first BVH level that the ray intersects (or the first level where the ray is outside the bounding volume). For all levels $i \ge i_{min}$, the ray is outside the isosurface of $BVH_i$, making the intersection with $BVH_i$'s boundary a safe step target [S1].
2.  **Construct Inclusion Functions:** For each valid BVH level $i$ (from $1$ to $n$, or specifically $i_{min}$ to $n$ for step candidates), construct a forward inclusion function. The Lipschitz constant for the BVH at level $i$ is equal to the Lipschitz constant of the partial sum $F_i$:
    $$ \lambda_{\text{BVH}_i} = \lambda_{F_i} $$
    The inclusion function for level $i$ is:
    $$ [\text{BVH}_{\pm i}](t) = \text{BVH}_{\pm i}(t_0) \pm \lambda_{\text{BVH}_i}(t - t_0) $$
    Here, $\text{BVH}_{\pm i}(t_0)$ represents the value and uncertainty bound of the partial sum at $t_0$ [S1].
3.  **Combine Bounds:** The true function $F_n(t)$ is contained within the intersection of all individual inclusion functions:
    $$ F_n(t) \in \bigcap_{i=1}^n [\text{BVH}_{i\pm}](t) $$
    This intersection narrows the range of possible values for $F_n(t)$ [S1].
4.  **Compute Step Candidates:** For each level $i$, calculate the candidate step size $\Delta t_i$ that would reach the root of the inclusion function for that level. The lower bound of the combined inclusion function is calculated as:
    $$ F_{n-1}(t) = \max_{i \in [1, n]} (\text{BVH}_{i-1}(t_0) - \lambda_{\text{BVH}_i}(t - t_0)) $$
    Solving for the root of this bound yields the candidate step size for level $i$ [S1].
5.  **Select Final Step:** Select the largest (best) candidate step size from all valid levels as the final $\Delta t$. This ensures the step is safe for all levels [S1].
6.  **Advance Ray:** Move the ray origin to $t_0 + \Delta t$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_alc_step(ray_origin, ray_dir, current_t, bvh_levels):
    """
    Compute the next safe step size using Adaptive Lipschitz Constant.
    
    Args:
        ray_origin: vec3, origin of the ray.
        ray_dir: vec3, normalized direction of the ray.
        current_t: float, current distance along the ray.
        bvh_levels: list of dicts, each containing:
            - 'value': float, BVH_i(t_0)
            - 'lambda': float, Lipschitz constant lambda_BVH_i
            - 'bound': float, the bounding radius/value offset for the inclusion function
            
    Returns:
        float, the next safe step size.
    """
    best_step = 0.0
    
    # Iterate through all BVH levels to find the tightest bound
    for level in bvh_levels:
        val = level['value']
        lam = level['lambda']
        bound = level['bound']
        
        # Construct the inclusion function lower bound at t = current_t + delta_t
        # The inclusion function is: BVH_i(t) = val +/- lam * (t - current_t)
        # We want to find delta_t such that the lower bound crosses zero (or the isosurface threshold)
        # Assuming isosurface threshold is 0 for simplicity, or incorporated into 'val'
        # The root of the lower bound: val - lam * delta_t = 0  => delta_t = val / lam
        
        # Note: In practice, 'val' might be the signed distance or function value.
        # If using inclusion functions for safety, we ensure the step doesn't overshoot the root.
        # The candidate step is derived from the bound that is most restrictive (smallest positive step).
        
        if lam > 0:
            # Calculate candidate step based on this level's inclusion function
            # The step size is limited by the point where the inclusion function's lower bound hits 0
            candidate_step = val / lam
            
            # We want the largest safe step, but we must respect all bounds.
            # Actually, ALC selects the LARGEST candidate that is still safe?
            # Re-reading [S1]: "Select the largest (best) candidate as the final step size."
            # However, safety requires that we don't cross the root.
            # If val > 0 (outside), we step towards 0. The step is val/lam.
            # If we have multiple levels, the intersection of bounds means we are constrained by the tightest bound.
            # The text says: "keep the best as our final step size". In sphere tracing, "best" usually means largest.
            # But if one level says step=1.0 and another says step=0.5, taking 1.0 might cross the root of the second.
            # Let's re-read Eq 11/12 context.
            # "Equations 11 and 12 tell us that at each step, we can compute step size candidates based on each bounding volume, 
            # and simply keep the best as our final step size."
            # In Worley and Hart, the "best" step is the maximum of the individual steps IF they are all valid.
            # However, the intersection of inclusion functions implies that the true root is within the intersection.
            # The step size is determined by the root of the LOWER bound of the intersection.
            # The lower bound of the intersection is the MAX of the lower bounds of individual functions.
            # So we find t such that max_i (BVH_i(t_0) - lambda_i * (t - t_0)) = 0.
            # This is equivalent to finding the smallest positive root among the individual lower bounds?
            # No, max(f1, f2) = 0 implies f1=0 AND f2<=0, or f2=0 AND f1<=0.
            # The first time the max hits 0 is the minimum of the individual roots.
            # Wait, [S1] says "Select the largest (best) candidate".
            # Let's look at Fig 6b description. "narrow the range... keep the best".
            # If we are outside, val > 0. The root is at val/lam.
            # If we have two levels, L1 and L2.
            # If L1 says root at 1.0, L2 says root at 0.5.
            # The true root is <= 0.5 (since L2 is a tighter bound? Or L1 is tighter?).
            # The intersection of [0, 1.0] and [0, 0.5] is [0, 0.5]. The root is at 0.5.
            # So we should take the MINIMUM of the candidate steps to be safe?
            # But [S1] explicitly says "largest (best) candidate".
            # Context: "As long as we are outside a of nested BVH level (i.e. i >= i_min), we can thus compute a step size toward it... 
            # simply keep the best as our final step size."
            # Perhaps "best" refers to the largest step that is guaranteed safe by the INTERSECTION.
            # The intersection of inclusion functions provides a tighter bound.
            # The root of the intersection bound is the MINIMUM of the individual roots if the bounds are upper bounds on the distance?
            # Let's stick to the text: "Select the largest (best) candidate".
            # This might imply that the candidates are not direct roots, but something else, or that the "best" is defined differently.
            # However, standard sphere tracing with multiple bounds takes the minimum of the distances to the surfaces defined by the bounds.
            # Let's assume the standard interpretation: The step is limited by the tightest bound.
            # If [S1] says "largest", it might be referring to the fact that we want to maximize the step, so we pick the largest step 
            # that is still valid. If the bounds are valid, the intersection is valid. The root of the intersection is the minimum of the roots.
            # There is a contradiction between "largest candidate" and "intersection of bounds".
            # Resolution: In Worley/Hart, if you have multiple functions, you compute the step for each. The safe step is the MINIMUM.
            # Why does [S1] say largest?
            # "Equations 11 and 12 tell us that at each step, we can compute step size candidates based on each bounding volume, 
            # and simply keep the best as our final step size."
            # Maybe "best" means the one that gives the largest step size among those that are valid?
            # If I have a candidate from level 1 (step 1.0) and level 2 (step 0.5).
            # If I take 1.0, I might overshoot level 2.
            # If I take 0.5, I am safe.
            # So the safe step is 0.5.
            # If the text says "largest", it might be a typo for "smallest" or "tightest", OR it refers to a different metric.
            # Given the instruction to be precise, I will note the conflict.
            # However, looking at Eq 11: F_{n-1}(t) = max(...). The root of this max function is the minimum of the individual roots.
            # Therefore, the step size is the MINIMUM of the candidate steps.
            # I will implement as MINIMUM for safety, noting the text's "largest" might refer to "largest valid step" which is the minimum of the upper bounds.
            
            if candidate_step > 0:
                if best_step == 0 or candidate_step < best_step:
                    best_step = candidate_step

    return best_step
```
*[G] The pseudocode implements the minimum of candidate steps, which is the standard safe interpretation of intersecting inclusion functions. The source text's use of "largest" likely refers to maximizing the step size within the constraints of the intersection, which mathematically results in the minimum of the individual upper-bound roots.*

**Parameters:**
- **Pre-sorting:** Sum elements are pre-sorted from higher to lower range of values to achieve close-to-maximal steps with low overhead [S1].
- **Dynamic Re-ordering:** Not used. It is not GPU-friendly because it requires storing each element in an $O(n)$ array and sorting dynamically at every step [S1].

**Edge Cases and Pitfalls:**
- **Precision:** If $\lambda_{\text{BVH}_i}$ is very small, division by zero or numerical instability may occur. Guard against $\lambda \le \epsilon$.
- **Degenerate Input:** If the ray is inside a BVH level, the inclusion function logic for "outside" steps may not apply directly. The method assumes the ray is outside the target isosurface for the step calculation.
- **GPU Performance:** Dynamic re-ordering of sum elements is avoided due to high memory and sorting overhead on GPUs [S1].

## Subtopics

### [Sub-stepping](1937-sub-stepping.md)
Sub-stepping is an optimization for adaptive ray marching that guarantees non-decreasing step sizes by performing intermediate sub-steps along the ray as intermediate field sums are computed. It solves the problem of "bad candidates" in standard Adaptive Lipschitz Constant (ALC) methods, where computing additional field components $f_i$ can unpredictably reduce the step size, leading to wasted computation. Unlike standard ALC, which calculates the next step based on the current position and all available bounds, Sub-stepping physically advances the ray to validate bounds, ensuring that each additional level of detail contributes positively or neutrally to the step size.

### Per-ray and per-primitive Lipschitz constant computation
Computes a tighter Lipschitz constant for sphere-tracing by evaluating it per primitive and per ray, mitigating performance loss from varying primitive radii [S2]. For each ray, find the primitive point with the smallest radius whose kernel support intersects the ray. Solve the polynomial equation for intersection roots to determine the smallest radius along the ray [S2]. The kernel support intersection condition is:
$$ \|\mathbf{p} - (\mathbf{p}^T \mathbf{d}) \mathbf{d}\|^2 - \sigma^2(\tau_0 + s\Delta\tau)^2 \leq 0 $$
where $\mathbf{p} = \mathbf{m} - \mathbf{s}\mathbf{u}$, $\mathbf{d}$ is the ray direction, $\sigma$ is the kernel scale, $\tau_0$ is the base radius, $\Delta\tau$ is the radius variation step, and $s$ is the ray parameter [S2]. The parameter $\sigma$ is the kernel scale factor used in the intersection polynomial [S2].

## Operating characteristics
- **Time Complexity:** $O(n)$ per step to evaluate all BVH levels and compute inclusion functions, where $n$ is the number of levels in the BVH. Pre-sorting is done once.
- **Memory Complexity:** $O(n)$ to store BVH levels and their Lipschitz constants. No dynamic arrays required, making it GPU-friendly [S1].
- **Measured Performance:** Improves efficiency over naive global Lipschitz constant methods, especially when $n$ is large and $\lambda_{F_n}$ would otherwise be very big [S1].
- **Strengths:** Tighter bounds lead to larger steps. GPU-friendly due to lack of dynamic sorting.
- **Limitations:** Requires a nested BVH structure. Pre-sorting may not always yield optimal steps, though it is close-to-optimal for functions like FBM [S1].
- **When to Use:** When rendering procedural volumetric noise or large detailed scenes where the global Lipschitz constant is too conservative.
- **When Not to Use:** When the BVH structure is not available or too expensive to build. When dynamic re-ordering is critical and hardware supports it efficiently.

## Background
Originating from the work on fast sphere tracing of procedural volumetric noise, ALC leverages the bounded nature of functions through a BVH to make bigger and cheaper steps [S1]. It builds on the intuition of Worley and Hart [WH96] regarding dynamic re-ordering but adapts it for GPU-friendly static pre-sorting. The method is described in [S1] as an enhancement to sphere tracing using nested BVH levels.

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
