# Dynamic Epsilon for Self-Intersection Prevention

Also called: Dynamic offset, Dynamic ε for self-intersection prevention

## Overview
Dynamic epsilon is a method for preventing self-intersection artifacts in secondary rays (e.g., refraction) during sphere tracing by computing a local offset $\varepsilon_{\text{dynamic}}$ at each intersection point, rather than using a global constant [S1][S2, unofficial]. It solves the problem where global epsilon values are inappropriate for sphere tracing with screen-space termination metrics because intersection distances from the surface exhibit large variations [S1][S2, unofficial]. Unlike global epsilon, dynamic epsilon adapts to the local surface distance, making it easier and more robust to tune than a single global value [S1][S2, unofficial].

## How it works
**Inputs:**
- Intersection point $p$ obtained from sphere tracing.
- Signed distance function value at $p$, denoted $f(p)$.
- Global minimal offset constant $\varepsilon_{\text{min}}$.
- Surface normal at $p$, denoted $\mathbf{n}$ (or ray direction $\mathbf{d}$ if normal is unavailable, though normal is preferred for refraction [S1]).

**Outputs:**
- New ray origin $p'$ for the secondary ray.

**Data Structures:**
- No complex data structures required. The method operates on scalar values and vectors available at the intersection point.

**Parameters:**
- $\varepsilon_{\text{min}}$: Global minimal offset value. Must be chosen carefully [S1][S2, unofficial].
  - Too small: Causes numerical problems ($f(p) \approx 0$) and poor performance (slow acceleration away from surface) [S1][S2, unofficial].
  - Too large: May cause unnecessary displacement, potentially missing thin features or causing other artifacts (not explicitly detailed, but implied by the need for careful choice) [S1][S2, unofficial].
  - Typical values: Not given in the sources.

**Steps:**
1. Evaluate the signed distance function at the intersection point $p$ to obtain $f(p)$ [S1][S2, unofficial].
2. Compute the absolute distance to the surface: $d_{\text{surf}} = |f(p)|$ [S1][S2, unofficial].
3. Calculate the dynamic epsilon $\varepsilon_{\text{dynamic}}$ using the formula:
   $$ \varepsilon_{\text{dynamic}} = 2 \cdot \max(\varepsilon_{\text{min}}, |f(p)|) $$
   This ensures $\varepsilon_{\text{dynamic}} \ge 2 \cdot \varepsilon_{\text{min}}$ [S1][S2, unofficial].
4. Offset the intersection point $p$ along the surface normal $\mathbf{n}$ (or ray direction $\mathbf{d}$) by $\varepsilon_{\text{dynamic}}$ to compute the new ray origin $p'$:
   $$ p' = p + \varepsilon_{\text{dynamic}} \cdot \mathbf{n} $$
   [S1][S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dynamic_epsilon_offset(p, f_p, epsilon_min, normal):
    """
    Compute the new ray origin for a secondary ray to prevent self-intersection.
    
    Args:
        p: Intersection point (vec3).
        f_p: Signed distance function value at p (float).
        epsilon_min: Global minimal offset constant (float).
        normal: Surface normal at p (vec3, normalized).
        
    Returns:
        p_prime: New ray origin (vec3).
    """
    # Step 1 & 2: Get absolute distance to surface
    abs_f_p = abs(f_p)
    
    # Step 3: Calculate dynamic epsilon
    # Formula: epsilon_dynamic = 2 * max(epsilon_min, |f(p)|)
    epsilon_dynamic = 2.0 * max(epsilon_min, abs_f_p)
    
    # Step 4: Offset the point along the normal
    p_prime = p + epsilon_dynamic * normal
    
    return p_prime
```

**Edge Cases and Pitfalls:**
- **Numerical Stability:** If $f(p) \approx 0$ (point is very close to the surface), using a global epsilon might be too small or too large. The dynamic method ensures $\varepsilon_{\text{dynamic}} \ge 2 \cdot \varepsilon_{\text{min}}$, avoiding numerical problems [S1][S2, unofficial].
- **Performance:** If $\varepsilon_{\text{min}}$ is too small, the new ray origin may still be very close to the surface, causing sphere tracing to spend many iterations accelerating away from the surface, degrading performance [S1][S2, unofficial].
- **Screen-Space Termination:** Global epsilon is inappropriate when using screen-space termination metrics because intersection distances vary significantly [S1][S2, unofficial]. Dynamic epsilon adapts to these variations.
- **Choice of $\varepsilon_{\text{min}}$:** Manually choosing $\varepsilon_{\text{min}}$ is easier and more robust than choosing a global epsilon that works for all distances [S1][S2, unofficial].

## Operating characteristics
- **Time Complexity:** Negligible additional cost. The computation of $\varepsilon_{\text{dynamic}}$ requires only a few arithmetic operations and is performed once per secondary ray origin calculation [S1][S2, unofficial].
- **Memory Complexity:** $O(1)$ additional memory per ray, as no new data structures are allocated [S1][S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of secondary rays, as the cost is per-ray and constant [S1][S2, unofficial].
- **Strengths:**
  - Prevents self-intersection artifacts effectively in scenarios with large variations in surface distance [S1][S2, unofficial].
  - Adapts to local geometry, making it robust for screen-space termination metrics [S1][S2, unofficial].
  - Easier to tune than a global epsilon value [S1][S2, unofficial].
- **Limitations:**
  - Requires careful selection of $\varepsilon_{\text{min}}$ to balance numerical stability and performance [S1][S2, unofficial].
  - May still require many iterations if $\varepsilon_{\text{min}}$ is too small, leading to performance degradation [S1][S2, unofficial].
- **When to use:** Use when rendering secondary rays (e.g., refraction) in sphere tracing, especially when using screen-space termination metrics or when intersection distances from the surface vary significantly [S1][S2, unofficial].
- **When not to use:** Not necessary for primary rays or when using global epsilon is sufficient (e.g., simple scenes with uniform scale and no screen-space termination) [S1][S2, unofficial].

## Background
- **Origin:** Proposed in "Enhanced Sphere Tracing" by [S1][S2, unofficial].
- **Date:** 2014 (based on DOI and PDF metadata) [S1][S2, unofficial].
- **History and Development:** The method addresses a well-known problem in ray tracing: preventing self-intersections of secondary rays [S1][S2, unofficial]. While virtual offsetting with a global epsilon is common, it fails in sphere tracing due to large variations in intersection distances, particularly with screen-space termination metrics [S1][S2, unofficial]. The dynamic epsilon method was introduced to resolve this by adapting the offset to the local surface distance [S1][S2, unofficial].
- **Usage:** Used in non-real-time GPU path tracers and real-time sphere tracing applications to improve rendering quality and performance [S1][S2, unofficial].

## Sources
[S1] Enhanced Sphere Tracing, https://doi.org/10.2312/stag.20141233, fetched 2026-10-26
[S2] Enhanced Sphere Tracing, https://kev.town/raymarching-toolkit/media/enhanced_sphere_tracing.pdf, fetched 2026-10-27 (unofficial)
