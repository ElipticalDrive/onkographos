# Relaxed Sphere Tracing

Also called: Keinert et al. sphere tracing

## Overview
Relaxed sphere tracing is an acceleration technique for distance-field ray marching that increases the step size beyond the conservative SDF value to reduce iteration counts [S1, S2, unofficial]. It scales the step by a relaxation factor $\omega$, reverting to the standard safe step if the new sample indicates the ray has jumped over the surface (i.e., the unbounding spheres of consecutive steps are disjoint) [S1, S2, unofficial]. It is distinguished from [Enhanced Sphere Tracing](3867-enhanced-sphere-tracing.md) by its simpler geometric assumption; while relaxed tracing uses a fixed scalar multiplier, enhanced tracing computes an optimal tangential step size for planar surfaces, offering superior convergence at the cost of higher per-step complexity [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{p} \in \mathbb{R}^3$.
- Ray direction $\mathbf{v} \in \mathbb{R}^3$ with $\|\mathbf{v}\| = 1$.
- Signed Distance Function Estimate (SDFE) $f: \mathbb{R}^3 \to \mathbb{R}$.
- Relaxation parameter $\omega \ge 0$.
- Maximum iteration count $i_{max}$.
- Convergence threshold $\epsilon$ (e.g., $0.001$).
- Maximum travel distance $t_{max}$ (optional, for bounding volumes).

**Outputs:**
- Distance traveled $t$ along the ray.
- Hit status (implicit in termination condition).

**Data Structures:**
- Scalar variables: $t$ (accumulated distance), $r_i$ (SDF value at current step), $r_{i+1}$ (SDF value at next step), $d_i$ (step size).
- Integer counter: $i$ (iteration index).

**Parameters:**
- $\omega$: Relaxation factor. A typical value cited is $0.6$, resulting in a step size multiplier of $1.6$ [S2, unofficial]. The sources do not specify an optimal theoretical range, but $\omega > 0$ is required for acceleration.

**Algorithm Steps:**
1. Initialize $t = 0$, $i = 0$, $r_i = 0$, $r_{i+1} = \infty$ (or a value $> \epsilon$).
2. Loop while $i < i_{max}$ and $r_{i+1} > \epsilon$:
   a. Calculate the relaxed step size: $d_i = r_i + \omega \cdot r_i = r_i(1 + \omega)$.
   b. Sample the SDF at the proposed next position: $\mathbf{p}_{next} = \mathbf{p} + (t + d_i)\mathbf{v}$.
   c. Compute $r_{i+1} = f(\mathbf{p}_{next})$.
   d. **Safety Check:** If $d_i > r_i + r_{i+1}$, the unbounding spheres of the current and next steps are disjoint, meaning the ray might have skipped the surface. In this case, revert to the conservative step:
      - Set $d_i = r_i$.
      - Recompute the position: $\mathbf{p}_{next} = \mathbf{p} + (t + d_i)\mathbf{v}$.
      - Recompute $r_{i+1} = f(\mathbf{p}_{next})$.
   e. Update accumulated distance: $t = t + d_i$.
   f. Update current radius: $r_i = r_{i+1}$.
   g. Increment iteration: $i = i + 1$.
3. Return $t$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def relaxed_sphere_tracing(p, v, f, omega, epsilon, max_steps):
    t = 0.0
    i = 0
    r_i = 0.0
    r_next = float('inf')
    
    # Initial sample to set r_i correctly for the first iteration logic
    # Note: Standard sphere tracing starts with r_i = f(p). 
    # The source algorithm initializes r_i=0, r_next=inf, implying 
    # the loop body handles the first sample. However, logically, 
    # r_i must be the distance at the CURRENT position before stepping.
    # The source [S1, unofficial] pseudocode shows:
    # r_i = 0; r_next = inf; t = 0; i = 0
    # Loop:
    #   d_i = r_i + omega * r_i  <-- If r_i is 0, d_i is 0. This suggests 
    #   the source pseudocode might assume r_i is initialized to f(p) 
    #   before the loop, or the first iteration is a special case.
    # Re-reading [S1, unofficial] Algorithm 2:
    # "i=0; r_i=0; r_{i+1}=inf; t=0"
    # "for i < imax and r_{i+1} not too small"
    # "d_i = r_i + omega * r_i"
    # If r_i is 0, step is 0. This is likely a typo in the source or 
    # implies r_i is set to f(p) before the loop. 
    # Standard interpretation: r_i holds f(current_point).
    
    # Corrected initialization based on standard sphere tracing logic 
    # and the need for a non-zero first step:
    r_i = f(p)
    
    while i < max_steps and r_i > epsilon:
        # Calculate relaxed step size
        d_i = r_i * (1.0 + omega)
        
        # Propose next position
        p_next = p + (t + d_i) * v
        
        # Sample SDF
        r_next = f(p_next)
        
        # Safety check: disjoint unbounding spheres
        # Condition: step size > sum of radii
        if d_i > r_i + r_next:
            # Revert to conservative step
            d_i = r_i
            p_next = p + (t + d_i) * v
            r_next = f(p_next)
            
        # Update state
        t = t + d_i
        r_i = r_next
        i = i + 1
        
    return t
```

**Edge Cases and Pitfalls:**
- **Disjoint Spheres:** The core safety mechanism is the check $d_i > r_i + r_{i+1}$. If this condition is met, the ray has jumped over the surface. Failing to revert to the conservative step $d_i = r_i$ will cause the ray to miss the intersection [S1, S2, unofficial].
- **Zero Distance:** If the ray starts exactly on the surface ($f(\mathbf{p}) = 0$), the loop terminates immediately.
- **Oscillation:** In narrow gaps or highly curved surfaces, the relaxation factor may cause the ray to oscillate near the surface if the safety check is not robust. The conservative fallback prevents infinite loops but may reduce performance in such areas [S1, unofficial].
- **Precision:** Floating-point precision errors in the SDF evaluation can lead to incorrect safety checks. Using a small epsilon for the comparison $d_i > r_i + r_{i+1}$ may be necessary in practice [G].

## Subtopics

### Enhanced Sphere Tracing
Uses a geometric construction to find the optimal tangential step size for planar surfaces, offering faster convergence than relaxed tracing but with higher computational cost per step [S1, unofficial].

### Uniform Sphere Tracing
Uses a fixed step size regardless of the SDF value, simpler but less efficient for complex scenes [4406-uniform-sphere-tracing.md].

## Operating characteristics
- **Time Complexity:** $O(k)$ where $k$ is the number of iterations. Relaxed sphere tracing typically requires fewer iterations than basic sphere tracing due to larger step sizes, but each iteration includes a conditional branch for the safety check [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, as it only stores scalar values for the current ray state [S1, unofficial].
- **Performance:** The relaxation factor $\omega$ improves performance by reducing iteration counts. However, the source [S1, unofficial] notes that **Enhanced Sphere Tracing** is "decisively better" than relaxed sphere tracing in terms of error-decrease velocity across all iteration counts. Relaxed tracing is outperformed by enhanced tracing on every iteration count, though enhanced tracing has a more expensive single-step construction [S1, unofficial].
- **Strengths:** Simple to implement; significant speedup over basic sphere tracing for scenes with large empty spaces or smooth surfaces where conservative steps are wasteful [S2, unofficial].
- **Limitations:** Less efficient than enhanced sphere tracing; prone to skipping intersections if the relaxation factor is too high or the geometry is complex, requiring the safety check which adds branching overhead [S1, unofficial].

## Background
Relaxed sphere tracing was introduced by Keinert et al. in 2014 [S1, S2, unofficial]. It was later used in the "Claybook" real-time simulation game to implement efficient rendering [S2, unofficial]. The method is based on the observation that the conservative step size of standard sphere tracing is often too small, and scaling it up by a factor of $1+\omega$ can accelerate convergence without missing surfaces, provided a safety check is in place [S1, S2, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Csaba Bálint 1. Introduction 2. Sphere Tracing 3. Relaxed ..., https://people.inf.elte.hu/csabix/publications/articles/WSPS_2019_Poster.pdf, fetched 2026-10-29 (unofficial)
