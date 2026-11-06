# Discontinuity Reduction via Fixed-Point Iteration

Also called: Discontinuity smoothing, Fixed-point iteration smoothing, Fixed-point smoothing

## Overview
Discontinuity reduction via fixed-point iteration is a post-processing step applied to intersection points retrieved by sphere tracing to smooth out discontinuities in the signed distance function (SDF) $f(p)$ and reduce artifacts in procedural texturing [S1][S2, unofficial]. It sets itself apart from its siblings by trading a slight increase in average geometric error for significantly reduced visual discontinuities across the screen-space domain, whereas [Affine Arithmetic Early Skip Test](4313-affine-arithmetic-early-skip-test.md) focuses on skipping empty space and [Dynamic Epsilon for Self-Intersection Prevention](4318-dynamic-epsilon-for-self-intersection-prevention.md) focuses on preventing self-intersection artifacts [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An initial intersection point $p_0$ obtained from sphere tracing, the ray direction $d$, the ray origin $o$, the SDF function $f$, and a screen-space error function $\text{err}(\cdot)$.
*   **Output:** A smoothed intersection point $p_{final}$ that satisfies the fixed-point condition.

**Data Structures:**
No complex data structures are required. The method operates on scalar and vector variables:
*   $p_i$: Current point estimate (vec3).
*   $d$: Ray direction (vec3, normalized).
*   $o$: Ray origin (vec3).
*   $f(p)$: SDF value at point $p$ (float).
*   $\text{err}(r)$: Screen-space error threshold based on distance from origin (float).

**Parameters:**
*   **Number of iterations ($N$):** Typically 3 iterations are sufficient to eliminate artifacts [S1][S2, unofficial]. 5 iterations are used in some visualizations [S1][S2, unofficial].
*   **Screen-space error function $\text{err}(r)$:** Defines the target error tolerance based on the distance from the ray origin. The exact implementation of $\text{err}$ depends on the screen-space criterion used (e.g., pixel radius), but it returns a scalar distance value [S1][S2, unofficial].

**Algorithm Steps:**
1.  Obtain the initial intersection point $p_0$ from the sphere tracing loop.
2.  For $i$ from 0 to $N-1$:
    a. Calculate the current distance from the ray origin: $r_i = \|p_i - o\|_2$.
    b. Evaluate the SDF at the current point: $val_i = f(p_i)$.
    c. Calculate the target error: $target_i = \text{err}(r_i)$.
    d. Update the point using the fixed-point iteration formula:
       $$p_{i+1} = p_i + d \cdot (val_i - target_i)$$
3.  Return $p_N$ as the smoothed intersection point.

**Formula:**
The core update rule is derived from seeking a point $p$ such that:
$$f(p) = \text{err}\left(\|p - o\|_2\right) \quad (2)$$
The iterative update moves the point along the ray direction $d$ by the difference between the current SDF value and the target screen-space error [S1][S2, unofficial]:
$$p_{i+1} = p_i + d \cdot (f(p_i) - \text{err}(\|p_i - o\|_2)) \quad (3)$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def discontinuity_reduction(p0, d, o, f, err_func, num_iterations=3):
    """
    Smooths an intersection point obtained from sphere tracing.
    
    Args:
        p0: Initial intersection point (vec3) from sphere tracing.
        d: Ray direction (vec3), assumed normalized.
        o: Ray origin (vec3).
        f: Signed Distance Function (callable taking vec3, returning float).
        err_func: Screen-space error function (callable taking distance float, returning float).
        num_iterations: Number of smoothing iterations (int).
        
    Returns:
        Smoothed intersection point (vec3).
    """
    p = p0
    
    for i in range(num_iterations):
        # Calculate distance from ray origin
        dist_from_origin = length(p - o)
        
        # Evaluate SDF at current point
        sdf_val = f(p)
        
        # Get target error from screen-space criterion
        target_error = err_func(dist_from_origin)
        
        # Update point along ray direction
        # p_{i+1} = p_i + d * (f(p_i) - err(||p_i - o||_2))
        p = p + d * (sdf_val - target_error)
        
    return p
```

**Edge Cases and Pitfalls:**
*   **Increased Average Error:** The average value of $f(p)$ may increase after applying this technique, resulting in greater overall geometric error [S1][S2, unofficial]. However, this is a deliberate trade-off to achieve significantly reduced discontinuities and better visual quality [S1][S2, unofficial].
*   **Convergence:** The method assumes the fixed-point iteration converges. With typical parameters (3-5 iterations), it effectively smooths discontinuities caused by limited sphere tracing iterations and constant/screen-space thresholds [S1][S2, unofficial].
*   **Ray Direction:** The update moves strictly along the ray direction $d$. If the true intersection is significantly off-axis, this method may not correct it fully, but it is designed to smooth discontinuities along the ray path [S1][S2, unofficial].

## Subtopics

### [Affine Arithmetic Early Skip Test](4313-affine-arithmetic-early-skip-test.md)
Uses affine arithmetic to skip empty space efficiently. Choose when performance in empty regions is critical.

### [Dynamic Epsilon for Self-Intersection Prevention](4318-dynamic-epsilon-for-self-intersection-prevention.md)
Prevents self-intersection artifacts in secondary rays. Choose when rendering reflections/refractions from SDF surfaces.

### [Convex Object Optimization](4367-convex-object-optimization.md)
Accelerates tracing through convex objects using bounding volumes. Choose when rendering scenes with many convex SDF objects.

### [Ray Intersection with Grazing Point Detection](4463-ray-intersection-with-grazing-point-detection.md)
Detects grazing intersections to improve normal accuracy. Choose when surface normals are critical for shading.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of smoothing iterations. Each iteration requires one SDF evaluation and one error function evaluation.
*   **Memory Complexity:** $O(1)$, as it only stores the current point and a few scalars.
*   **Measured Performance:** The method adds a low, fixed cost. In the source experiments, it was used to improve visual quality without significantly impacting frame rates, especially when combined with other optimizations like convex object optimization which provided a ~10% speedup in a path tracer [S1][S2, unofficial].
*   **Strengths:** Significantly reduces visual artifacts in procedural texturing caused by discontinuities in intersection distance and traveled distance [S1][S2, unofficial]. Allows for satisfying images even with reduced precision and a low number of sphere tracing iterations [S1][S2, unofficial].
*   **Limitations:** Increases the average geometric error of the intersection point [S1][S2, unofficial]. It is a post-processing step and does not improve the accuracy of the sphere tracing itself, only the visual smoothness of the result.
*   **When to use:** When rendering procedural textures or when visual smoothness is more important than strict geometric accuracy, especially with limited iteration counts.
*   **When not to use:** When strict geometric accuracy is required, or when the SDF is not continuous/smooth enough for the fixed-point iteration to converge effectively.

## Background
The method was introduced in the paper "Enhanced Sphere Tracing" by [S1][S2, unofficial]. It addresses the issue that limited sphere tracing iterations and constant or screen-space based termination criteria lead to discontinuities in the intersection points' distance from the surface and the distance traveled along the ray [S1][S2, unofficial]. These discontinuities cause unpleasant artifacts, particularly with procedural texturing [S1][S2, unofficial]. The technique employs a fixed-point iteration scheme to smooth out these discontinuities in the SDF $f(p)$ [S1][S2, unofficial].

## Sources
[S1] Enhanced Sphere Tracing, https://doi.org/10.2312/stag.20141233, fetched 2026-10-26
[S2] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
