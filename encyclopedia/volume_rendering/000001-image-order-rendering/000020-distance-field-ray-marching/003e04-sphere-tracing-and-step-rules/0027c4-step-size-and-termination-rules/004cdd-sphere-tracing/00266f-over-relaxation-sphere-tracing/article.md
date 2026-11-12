# Over-relaxation Sphere Tracing

Also called: Over-relaxed sphere tracing, Sphere tracing with over-relaxation, Over-relaxation-based sphere tracing, Over-relaxation sphere tracing with screen-space selection, Over-relaxation Sphere Tracing with Screen-Space Intersection Selection

## Overview
Over-relaxation sphere tracing accelerates the convergence of standard sphere tracing by stepping along the ray by a factor $\omega > 1$ times the current signed distance estimate (SDE), reducing the total number of distance function evaluations [S1][S3, unofficial][S4, abstract only, unofficial]. It differs from standard sphere tracing, which steps by exactly the SDE, by trading safety for speed: if the larger step overshoots the surface (detected by checking if the new unbounding sphere is disjoint from the previous one), the algorithm falls back to standard sphere tracing ($\omega=1$) from a corrected position [S1][S3, unofficial]. Unlike [Enhanced Sphere Tracing](3811-enhanced-sphere-tracing.md), which uses historical step data to approximate surface curvature, over-relaxation uses a constant multiplier and a simple overlap test [S4, abstract only, unofficial]. It also integrates a screen-space error metric to select the best intersection candidate when the maximum iteration count is exceeded, avoiding artifacts common when simply using the last evaluated point [S1].

## How it works
**Inputs:**
*   Ray origin $o$ and normalized direction $d$.
*   Signed distance function $f(p)$, returning the signed distance bound at point $p$.
*   $t_{min}$: Minimum ray distance.
*   $t_{max}$: Maximum ray distance.
*   $\omega$: Relaxation parameter, typically in $[1, 2)$ [S1][S2, unofficial].
*   $MAX\_ITERATIONS$: Maximum number of steps (e.g., 80) [S1][S2, unofficial].
*   $pixelRadius$: Radius of a pixel at $t=1$, used as a sub-pixel accuracy threshold [S1][S2, unofficial].
*   $forceHit$: Boolean flag; if true, returns the best candidate even if error exceeds $pixelRadius$ or $t_{max}$ is reached [S1].

**Outputs:**
*   $t_{hit}$: The distance to the intersection, or $INFINITY$ if no valid hit is found.

**Data Structures:**
*   `candidate_t`: Stores the best intersection distance found so far.
*   `candidate_error`: Stores the screen-space error of `candidate_t`.
*   `previousRadius`: Stores the absolute SDE of the previous step.
*   `stepLength`: The distance to advance in the current step.

**Algorithm Steps:**
1.  **Initialization:**
    *   Set $t = t_{min}$.
    *   Set `candidate_error` = $INFINITY$.
    *   Set `candidate_t` = $t_{min}$.
    *   Set `previousRadius` = 0.
    *   Set `stepLength` = 0.
    *   Determine `functionSign`: If $f(o) < 0$, `functionSign` = -1; else +1. This handles the sign of the distance bound relative to the ray origin [S1][S2, unofficial].

2.  **Iteration Loop:** For $i$ from 0 to $MAX\_ITERATIONS - 1$:
    *   **Evaluate Distance:** Compute current position $p = o + t \cdot d$. Calculate `signedRadius` = `functionSign` $\cdot f(p)$ [S1][S2, unofficial].
    *   **Absolute Radius:** Compute `radius` = $| \text{signedRadius} |$ [S1][S2, unofficial].
    *   **Overshoot Detection (SOR Fail):** Check if over-relaxation has caused the step to skip the surface. This occurs if $\omega > 1$ AND the sum of the absolute radii of the current and previous steps is less than the length of the previous step:
        $$ \text{radius} + \text{previousRadius} < \text{stepLength} $$
        If this condition is true, `sorFail` = true; otherwise, `sorFail` = false [S1][S2, S3, unofficial].
    *   **Handle Failure or Advance:**
        *   **If `sorFail` is true:**
            *   The previous step overshot. Retreat by backing up the step length: `stepLength` = `stepLength` - $\omega \cdot \text{stepLength}$ [S1][S2, unofficial].
            *   Disable over-relaxation for subsequent steps: $\omega = 1$ [S1][S2, unofficial].
            *   *Note:* Some implementations suggest falling back to a specific position $p_{fallback} = p_i + d \cdot \delta_{i-1} \cdot (1-\omega)$, but the provided pseudocode effectively handles this by reducing the step size and resetting $\omega$ for the next iteration, which naturally performs a standard sphere tracing step from the current position $p$ [S1][S2, unofficial].
        *   **If `sorFail` is false:**
            *   Calculate the new step length: `stepLength` = `signedRadius` $\cdot \omega$ [S1][S2, unofficial].
            *   Update `previousRadius` = `radius` [S1][S2, unofficial].
            *   **Screen-Space Error:** Calculate error metric: `error` = `radius` / $t$ [S1][S2, unofficial].
            *   **Update Best Candidate:** If `error` < `candidate_error`, update `candidate_t` = $t$ and `candidate_error` = `error` [S1][S2, unofficial].
            *   **Early Termination:** If `error` < `pixelRadius`, break the loop [S1][S2, unofficial].
            *   **Advance Ray:** $t = t + \text{stepLength}$ [S1][S2, unofficial].
    *   **Post-Step Check:** If $t > t_{max}$, break the loop [S1][S2, unofficial].

3.  **Final Result:**
    *   If `candidate_error` > `pixelRadius` AND $t > t_{max}$ AND `forceHit` is false, return $INFINITY$ [S1][S2, unofficial].
    *   Otherwise, return `candidate_t` [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def over_relaxation_sphere_tracing(o, d, f, t_min, t_max, omega, max_iterations, pixel_radius, force_hit):
    t = t_min
    candidate_error = float('inf')
    candidate_t = t_min
    previous_radius = 0.0
    step_length = 0.0
    
    # Determine sign of the distance bound at origin
    # f(o) < 0 implies inside, f(o) >= 0 implies outside
    function_sign = -1.0 if f(o) < 0 else 1.0
    
    for i in range(max_iterations):
        # Evaluate signed distance at current position
        # Note: d is normalized, so d*t + o is the world position
        p = o + d * t
        signed_radius = function_sign * f(p)
        radius = abs(signed_radius)
        
        # Check for over-relaxation failure (overshoot)
        # Condition: omega > 1 AND (current_radius + previous_radius) < previous_step_length
        sor_fail = (omega > 1) and ((radius + previous_radius) < step_length)
        
        if sor_fail:
            # Fallback: reduce step length and disable over-relaxation
            # Effectively backing up to perform a standard step next iteration
            step_length -= omega * step_length
            omega = 1.0
        else:
            # Calculate over-relaxed step size
            step_length = signed_radius * omega
            previous_radius = radius
            
            # Screen-space error metric
            # Avoid division by zero if t is very small
            if t > 0:
                error = radius / t
            else:
                error = float('inf')
            
            # Track best candidate
            if error < candidate_error:
                candidate_t = t
                candidate_error = error
            
            # Early termination if sub-pixel accuracy reached
            if error < pixel_radius:
                break
            
            # Advance ray
            t += step_length
            
            # Check max distance
            if t > t_max:
                break
    
    # Final decision
    if (t > t_max or candidate_error > pixel_radius) and not force_hit:
        return float('inf')
    else:
        return candidate_t
```

**Parameters:**
*   $\omega$: Relaxation parameter. Range $[1, 2)$ [S1][S2, unofficial]. Typical value 1.2 provides a good balance between performance and stability; values closer to 2 increase the risk of overshooting and premature fallback [S1][S2, unofficial].
*   $MAX\_ITERATIONS$: Typical value 80 [S1][S2, unofficial].
*   $pixelRadius$: Half the pixel size at $t=1$ [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Overshooting:** If $\omega$ is too large, the algorithm may step over the surface. The overlap test $|f(p_{i-1})| + |f(p_i)| < \delta_{i-1}$ detects this [S1][S2, unofficial]. If not handled, rendering artifacts such as holes appear [S3, unofficial].
*   **Non-Lipschitz SDFs:** Numeric or non-Lipschitz distance estimators (e.g., some user-defined DEs) can make the overlap test unreliable. In such cases, fall back to $\omega=1$ or make $\omega$ tunable [S3, unofficial].
*   **Grazing Rays:** When a ray grazes an object, it may terminate in mid-air if the last point is used. The screen-space error metric selects the best candidate, reducing artifacts [S1].
*   **Branching Divergence:** Checking if the fallback sphere overlaps with the current sphere to guarantee no surface was passed requires additional state and branching, which can diminish GPU performance due to thread divergence [S1][S2, unofficial]. The provided implementation avoids this by simply resetting $\omega$ and backing up the step size.

## Subtopics

### [Binary Sphere Tracing](10174-binary-sphere-tracing.md)
An extreme variant of over-relaxation sphere tracing designed for conformally transformed Kernel-Reflection Sequences (KRSs). It uses a binary search approach (advancing by full diameter or reducing radius) instead of distance functions, requiring only binary intersection tests with spheres. It is chosen when geometries are defined by implicit kernel sequences rather than explicit SDFs.

## Operating characteristics
*   **Performance:** Reduces the number of distance function evaluations (steps) by 20–40% in 3D renders compared to standard sphere tracing [S3, unofficial][S4, abstract only, unofficial].
*   **Measured Performance:**
    *   City scene (1280x720, NVIDIA GTX Titan): 16.829 ms (59.4 Hz) with over-relaxation vs 20.409 ms (49.0 Hz) without [S1].
    *   Jewel scene: 2.978 ms (335.7 Hz) with over-relaxation vs 3.472 ms (288.0 Hz) without [S1].
    *   Path tracing integration: 101.9 ms vs 118.3 ms for one sample per pixel [S1].
*   **Scaling:** Performance gains are consistent across hardware types (CPU/GPU) as the reduction is in function evaluations [S3, unofficial].
*   **Strengths:** Significant speedup for complex scenes; improves visual quality by allowing deeper tracing before hitting iteration limits; reduces artifacts from grazing rays.
*   **Limitations:** Requires careful tuning of $\omega$; larger $\omega$ leads to more frequent fallbacks; non-Lipschitz SDFs may require disabling over-relaxation; additional branching for robust fallback detection can hurt GPU performance [S1][S3, unofficial].
*   **When to use:** Real-time rendering of SDF-based geometry where performance is critical and SDFs are well-behaved (Lipschitz continuous).
*   **When not to use:** When using non-Lipschitz or user-defined distance estimators that violate the overlap test assumptions, or when strict correctness without any fallback risk is required without the overhead of additional checks.

## Background
The method was introduced by Keinert et al. in 2014 in the paper "Enhanced Sphere Tracing" [S1][S3, unofficial][S4, abstract only, unofficial]. It is part of a suite of enhancements to classical sphere tracing, including screen-space intersection selection and dynamic epsilon for self-intersection prevention. The technique is widely used in real-time ray marching for fractals and implicit surfaces [S3, unofficial][S4, abstract only, unofficial].

## Sources
[S1] Enhanced Sphere Tracing, https://doi.org/10.2312/stag.20141233, fetched 2026-10-26
[S2] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
[S3] Over-relaxed sphere tracing for the DE raymarchers (3D: ~20–40% fewer steps), https://github.com/AloneButsober/FracturingFog/issues/1335, fetched 2026-10-27 (unofficial)
[S4] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
