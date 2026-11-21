# Dynamic Epsilon for Self-Intersection Prevention

Also called: Dynamic offset, Dynamic ε for self-intersection prevention

## Overview
Dynamic epsilon is a method for preventing self-intersection artifacts in secondary rays (such as refraction or reflection) within sphere tracing by computing a local offset dynamically at each intersection point, rather than using a global constant [S1][S3, unofficial]. It solves the problem where global epsilon values are inappropriate for sphere tracing because intersection distances from the surface exhibit large variations, particularly when screen-space termination metrics are used [S1][S3, unofficial]. Unlike siblings that may use fixed offsets or affine arithmetic, this method adapts the offset magnitude to the local signed distance bound, ensuring robustness against precision issues near the surface while maintaining performance [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** The intersection point $p$ obtained from sphere tracing, the signed distance function value $f(p)$ at that point, and a global minimal offset constant $\epsilon_{min}$.
- **Output:** A dynamic offset value $\epsilon_{dynamic}$ and the new ray origin for the secondary ray.

**Data Structures:**
No specific complex data structures are required beyond standard floating-point variables for the point $p$, the scalar distance $f(p)$, and the constants.

**Parameters:**
- $\epsilon_{min}$: A global minimal offset value. It must be chosen carefully; if too small, it causes numerical problems (when $f(p) \approx 0$) and poor performance (slow acceleration away from the surface). If too large, it may introduce visible artifacts. The sources do not provide a specific numerical value, but note that manually choosing $\epsilon_{min}$ is easier and more robust than choosing a global epsilon that works for all distances [S1][S3, unofficial].

**Formulas:**
The dynamic epsilon is calculated as:
$$ \epsilon_{dynamic} = 2 \cdot \max(\epsilon_{min}, |f(p)|) $$
This ensures that $\epsilon_{dynamic} \ge 2 \cdot \epsilon_{min}$, which obviates numerical problems associated with $f(p) \approx 0$ [S1][S3, unofficial].

**Steps:**
1. At each intersection point $p$ obtained from sphere tracing, compute the signed distance bound $f(p)$ [S1].
2. Calculate $\epsilon_{dynamic}$ using the formula $\epsilon_{dynamic} = 2 \cdot \max(\epsilon_{min}, |f(p)|)$ [S1][S3, unofficial].
   - If $|f(p)| \ge \epsilon_{min}$, then $\epsilon_{dynamic} = 2 \cdot |f(p)|$ [S1][S3, unofficial].
   - If $|f(p)| \le \epsilon_{min}$, then $\epsilon_{dynamic} = 2 \cdot \epsilon_{min}$ [S1][S3, unofficial].
3. Offset the new ray origin for the secondary ray by $\epsilon_{dynamic}$ along the surface normal at $p$ (or along the ray direction, though the normal is preferred for refraction to avoid grazing issues) [S1][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dynamic_epsilon_offset(p, f_p, epsilon_min, normal):
    """
    Computes the new ray origin for a secondary ray to prevent self-intersection.
    
    Args:
        p: The intersection point from sphere tracing.
        f_p: The signed distance function value at p.
        epsilon_min: The global minimal offset constant.
        normal: The surface normal at p.
        
    Returns:
        new_origin: The offset ray origin for the secondary ray.
    """
    # Calculate dynamic epsilon based on the absolute signed distance
    # Formula: epsilon_dynamic = 2 * max(epsilon_min, abs(f_p))
    abs_f_p = abs(f_p)
    epsilon_dynamic = 2.0 * max(epsilon_min, abs_f_p)
    
    # Offset the ray origin along the normal
    # Note: The sources specify offsetting along the normal. 
    # In some contexts, offsetting along the ray direction is mentioned, 
    # but normal offset is standard for refraction to prevent grazing self-intersection.
    new_origin = p + epsilon_dynamic * normal
    
    return new_origin
```

**Edge Cases and Pitfalls:**
- **Precision Issues:** If $f(p)$ is very close to zero, a global epsilon might be too small, causing self-intersection. The dynamic method ensures $\epsilon_{dynamic} \ge 2 \cdot \epsilon_{min}$, avoiding this [S1][S3, unofficial].
- **Performance Degradation:** If $\epsilon_{min}$ is too small, the sphere tracing for the secondary ray must spend many iterations accelerating away from the surface, degrading performance [S1][S3, unofficial].
- **Screen-Space Termination:** Standard origin offsetting is insufficient for sphere tracing because the intersection point can be far from the surface, especially when using epsilon-based termination. Dynamic epsilon accommodates these large variations [S2, S3, unofficial].

## Subtopics

### [Affine Arithmetic Early Skip Test](4313-affine-arithmetic-early-skip-test.md)
Uses affine arithmetic to skip steps where the ray cannot intersect the object. Choose when performance is critical and the scene has simple primitives.

### [Convex Object Optimization](4367-convex-object-optimization.md)
Accelerates intersection tests for convex objects enclosed in convex bounding volumes. Choose when rendering scenes with many convex primitives.

### [Discontinuity Reduction via Fixed-Point Iteration](4368-discontinuity-reduction-via-fixed-point-iteration.md)
Reduces discontinuity artifacts in sphere tracing using fixed-point iteration. Choose when procedural texturing causes visual artifacts at intersection boundaries.

### [Ray Intersection with Grazing Point Detection](4463-ray-intersection-with-grazing-point-detection.md)
Detects grazing points to improve intersection accuracy. Choose when rendering scenes with thin geometry or grazing angles.

### [Gradient Magnitude Windowing](8299-gradient-magnitude-windowing.md)
Uses gradient magnitude to adjust step sizes. Choose when the signed distance function has varying smoothness.

## Operating characteristics
- **Time Complexity:** The computation of $\epsilon_{dynamic}$ is negligible because $f(p)$ is already known after tracing to the surface [S1][S3, unofficial]. The offset operation is a simple vector addition and scalar multiplication.
- **Memory Complexity:** Constant space, requiring only a few additional floating-point variables.
- **Measured Performance:** The sources do not provide specific benchmark numbers. However, it is noted that manually choosing $\epsilon_{min}$ is more robust than a global epsilon, and the cost of computing $\epsilon_{dynamic}$ is negligible [S1][S3, unofficial].
- **Scaling:** The method scales well as it does not depend on the number of iterations or scene complexity, only on the local properties at the intersection point.
- **Strengths:** Adapts to local surface distance, preventing self-intersection artifacts even with large variations in intersection distances. Robust against precision issues near the surface.
- **Limitations:** Requires careful tuning of $\epsilon_{min}$. If $\epsilon_{min}$ is too small, performance may degrade due to slow acceleration away from the surface.
- **When to Use:** Use when rendering secondary rays (refraction/reflection) in sphere tracing, especially with screen-space termination metrics.
- **When Not to Use:** Not necessary for primary rays or when using a different intersection method that does not suffer from self-intersection issues.

## Background
Dynamic epsilon was introduced in the paper "Enhanced Sphere Tracing" by [S1][S3, unofficial]. The paper presents several performance and quality enhancements to classical sphere tracing, including this method for dynamically preventing self-intersections upon converting signed distance bounds. The method addresses the limitations of global epsilon values in the context of sphere tracing, where intersection distances can vary significantly. The work is part of a broader effort to improve the robustness and performance of sphere tracing in real-time and non-real-time rendering contexts.

## Sources
[S1] Enhanced Sphere Tracing, https://doi.org/10.2312/stag.20141233, fetched 2026-10-26
[S2] 要約EnhancedSphereTracing～より良いレイマーチングに向けて～, https://qiita.com/Hirai0827/items/eddcb73a1976c3088b88, fetched 2026-10-26 (unofficial)
[S3] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
