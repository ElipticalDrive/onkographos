# Over-relaxation Sphere Tracing

Also called: Over-relaxation Sphere Tracing with Screen-Space Intersection Selection, Over-relaxation sphere tracing with screen-space selection, Over-relaxed sphere tracing, Sphere tracing with over-relaxation

## Overview
Over-relaxation Sphere Tracing accelerates the convergence of sphere tracing by stepping further than the signed distance bound allows, controlled by a relaxation parameter $\omega$ [S1, S2, unofficial]. It solves the problem of excessive iteration counts in complex scenes by reducing the number of signed distance function (SDF) evaluations required to reach an intersection or the maximum depth [S1, unofficial]. Unlike standard sphere tracing, which uses the exact distance bound, this method selects the final intersection point based on a screen-space error metric to reduce aliasing artifacts when the maximum iteration count is exceeded or when rays graze object edges [S1, S2, unofficial].

## How it works
The method combines an accelerated stepping strategy with a robust fallback mechanism and screen-space-aware termination. It requires a signed distance bound function $f(p)$ and a ray defined by origin $o$ and normalized direction $d$ [S1, S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $o$ and direction $d$ (normalized) [S1, S2, unofficial].
    *   Signed distance bound function $f(p)$ [S1, S2, unofficial].
    *   $t_{min}$: Minimum ray distance [S1, S2, unofficial].
    *   $t_{max}$: Maximum ray distance [S1, S2, unofficial].
    *   $pixelRadius$: The radius of a pixel at $t=1$, used as a sub-pixel accuracy threshold [S1, S2, unofficial].
    *   $MAX\_ITERATIONS$: Maximum number of iterations (typical value: 80) [S1, S2, unofficial].
    *   $omega$ ($\omega$): Relaxation parameter, range $[1, 2)$ (typical value: 1.2) [S1, S2, unofficial].
    *   $forceHit$: Boolean flag; if true, returns the best candidate even if error exceeds threshold [S1, S2, unofficial].
*   **Output**:
    *   Intersection distance $t$, or $INFINITY$ if no valid intersection is found within constraints [S1, S2, unofficial].

### Data Structures
The algorithm maintains the following state variables within the iteration loop:
*   $t$: Current distance along the ray.
*   $candidate\_t$: The best intersection distance found so far.
*   $candidate\_error$: The screen-space error associated with $candidate\_t$.
*   $previousRadius$: The absolute signed distance from the previous iteration.
*   $stepLength$: The distance to advance in the current step.
*   $functionSign$: $+1$ if $f(o) \ge 0$, $-1$ if $f(o) < 0$ [S1, S2, unofficial].

### Steps
1.  **Initialization**:
    *   Set $t = t_{min}$ [S1, S2, unofficial].
    *   Set $candidate\_error = INFINITY$ [S1, S2, unofficial].
    *   Set $candidate\_t = t_{min}$ [S1, S2, unofficial].
    *   Set $previousRadius = 0$ [S1, S2, unofficial].
    *   Set $stepLength = 0$ [S1, S2, unofficial].
    *   Determine $functionSign$: if $f(o) < 0$ then $-1$, else $+1$ [S1, S2, unofficial].

2.  **Iteration Loop** (for $i$ from 0 to $MAX\_ITERATIONS - 1$):
    *   **Evaluate SDF**:
        *   Calculate current position $p = o + t \cdot d$ [S1, S2, unofficial].
        *   Calculate $signedRadius = functionSign \cdot f(p)$ [S1, S2, unofficial].
        *   Calculate $radius = |signedRadius|$ [S1, S2, unofficial].
    *   **Check Over-Relaxation Failure (SOR Fail)**:
        *   A failure occurs if $\omega > 1$ AND $(radius + previousRadius) < stepLength$ [S1, S2, unofficial].
        *   This condition indicates that the unbounding spheres of the current and previous steps overlap, meaning the previous step may have overshot a surface [S1, S2, unofficial].
    *   **Handle Failure or Advance**:
        *   **If SOR Fail**:
            *   Reduce step length: $stepLength = stepLength - \omega \cdot stepLength$ [S1, S2, unofficial].
            *   Disable over-relaxation: Set $\omega = 1$ [S1, S2, unofficial].
            *   *Note*: The sources describe a fallback position $p_{fallback} = p_i + d \cdot \delta_{i-1} \cdot (1-\omega)$ for conventional tracing, but the provided pseudocode implements a step-length reduction and continues from the current position $p_i$ after resetting $\omega$ [S1, S2, unofficial]. The sources note that checking if the fallback sphere overlaps with the current sphere to guarantee no surface was passed is possible but omitted for performance due to branching divergence [S1, S2, unofficial].
        *   **If No SOR Fail**:
            *   Calculate step length: $stepLength = signedRadius \cdot \omega$ [S1, S2, unofficial].
            *   Update previous radius: $previousRadius = radius$ [S1, S2, unofficial].
            *   Calculate screen-space error: $error = radius / t$ [S1, S2, unofficial].
            *   **Update Best Candidate**:
                *   If $error < candidate\_error$:
                    *   Set $candidate\_t = t$ [S1, S2, unofficial].
                    *   Set $candidate\_error = error$ [S1, S2, unofficial].
            *   **Check Early Termination**:
                *   If ($error < pixelRadius$) OR ($t > t_{max}$):
                    *   Break loop [S1, S2, unofficial].
            *   **Advance Ray**:
                *   $t = t + stepLength$ [S1, S2, unofficial].

3.  **Post-Loop Result**:
    *   If ($t > t_{max}$ OR $candidate\_error > pixelRadius$) AND $forceHit$ is false:
        *   Return $INFINITY$ [S1, S2, unofficial].
    *   Else:
        *   Return $candidate\_t$ [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def over_relaxation_sphere_tracing(o, d, f, t_min, t_max, pixel_radius, max_iterations, omega=1.2, force_hit=False):
    t = t_min
    candidate_error = float('inf')
    candidate_t = t_min
    previous_radius = 0
    step_length = 0
    
    # Determine sign based on initial position
    if f(o) < 0:
        function_sign = -1
    else:
        function_sign = 1
        
    for i in range(max_iterations):
        # Evaluate signed distance
        p = o + t * d
        signed_radius = function_sign * f(p)
        radius = abs(signed_radius)
        
        # Check for over-relaxation failure (overshoot)
        # Condition: omega > 1 AND sum of radii < previous step length
        sor_fail = (omega > 1) and ((radius + previous_radius) < step_length)
        
        if sor_fail:
            # Reduce step length and disable over-relaxation
            step_length = step_length - omega * step_length
            omega = 1
        else:
            # Apply over-relaxation
            step_length = signed_radius * omega
            previous_radius = radius
            
            # Calculate screen-space error
            if t > 0:
                error = radius / t
            else:
                error = float('inf')
                
            # Update best candidate
            if error < candidate_error:
                candidate_t = t
                candidate_error = error
                
            # Check termination conditions
            if (not sor_fail and error < pixel_radius) or (t > t_max):
                break
                
            # Advance ray
            t = t + step_length
            
    # Final result logic
    if (t > t_max or candidate_error > pixel_radius) and not force_hit:
        return float('inf')
    else:
        return candidate_t
```

### Parameters
*   **$\omega$ (omega)**: The relaxation parameter. Range $[1, 2)$. Typical value is 1.2 for the best balance of performance and quality [S1, S2, unofficial]. Larger values cause earlier premature defaulting to conventional sphere tracing [S1, S2, unofficial].
*   **$pixelRadius$**: The radius of a pixel at $t=1$. Used as the threshold for sub-pixel accuracy [S1, S2, unofficial].
*   **$MAX\_ITERATIONS$**: Maximum iterations. Typical value 80 [S1, S2, unofficial].
*   **$forceHit$**: Boolean. If true, forces the return of $candidate\_t$ even if the error is large or $t_{max}$ is exceeded [S1, S2, unofficial].

### Edge Cases and Pitfalls
*   **Overshooting**: Over-relaxation can cause the ray to step into or over objects, missing intersections [S1, S2, unofficial]. This is detected by the overlap of unbounding spheres [S1, S2, unofficial].
*   **Grazing Rays**: When a ray grazes an object's edge, it may terminate in mid-air if the last point is used. The screen-space error metric mitigates this by selecting the point with the smallest unbounding sphere radius [S1, S2, unofficial].
*   **Branching Divergence**: An optimization to check if the fallback position's unbounding sphere overlaps with the current position's sphere (guaranteeing no surface was passed) is omitted in the standard implementation because it introduces branching and state handling that diminishes GPU performance [S1, S2, unofficial].
*   **Self-Intersections**: Secondary rays may suffer from self-intersection artifacts. The sources mention dynamic $\epsilon$ for prevention, but this is a separate concern from the core over-relaxation algorithm [S1, S2, unofficial].

## Operating characteristics
*   **Performance**: The method reduces the number of required SDF evaluations and iterations, allowing rays to trace deeper into the scene [S1, S2, unofficial].
    *   **City Scene** (1280x720, NVIDIA GTX Titan): 16.829 ms (59.421 Hz) with over-relaxation vs 20.409 ms (48.998 Hz) without [S1, S2, unofficial].
    *   **Jewel Scene** (1280x720, NVIDIA GTX Titan): 2.978 ms (335.744 Hz) with over-relaxation vs 3.472 ms (288.018 Hz) without [S1, S2, unofficial].
    *   **Path Tracing** (Jewel Scene, 10 bounces): 101.934 ms (9.8 Hz) with over-relaxation ($\omega=1.4$) vs 118.302 ms (8.453 Hz) without [S1, S2, unofficial].
*   **Strengths**: Significant performance improvement in complex scenes; reduces aliasing artifacts from grazing rays and iteration limits [S1, S2, unofficial].
*   **Limitations**: Tuning $\omega$ is challenging; too high a value causes premature fallback to slower conventional tracing [S1, S2, unofficial]. The fallback mechanism may still miss intersections if the overlap check is not rigorous enough (as per the simplified implementation) [S1, S2, unofficial].

## Background
The method was introduced in the paper "Enhanced Sphere Tracing" by the authors of the sources [S1, S2, unofficial]. It applies the principle of over-relaxation (common in numerical analysis) to sphere tracing [S1, S2, unofficial]. The technique was implemented in DirectX/DirectCompute and demonstrated real-time performance [S1, S2, unofficial].

## Sources
[S1] Enhanced Sphere Tracing, https://doi.org/10.2312/stag.20141233, fetched 2026-10-26 (unofficial)
[S2] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
