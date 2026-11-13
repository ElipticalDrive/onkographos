# Relaxed Sphere Tracing

Also called: Keinert et al. sphere tracing

## Overview
Relaxed sphere tracing is an acceleration technique for distance-field ray marching that increases the step size taken along a ray to reduce the number of Signed Distance Function (SDF) evaluations per pixel [S1, S2, unofficial]. It modifies the standard sphere tracing algorithm by scaling the conservative step size by a relaxation factor, allowing the ray to traverse empty space faster while maintaining safety guarantees [S2, unofficial]. It is distinguished from its sibling [Enhanced Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md) by its simpler geometric assumption (uniform scaling vs. tangential sphere approximation) and its tendency to fall back to standard steps more frequently, resulting in lower performance on complex scenes [S1, unofficial].

## How it works
Relaxed sphere tracing operates on the principle that the SDF value $f(\mathbf{p})$ provides a lower bound on the distance to the nearest surface. While standard sphere tracing steps exactly by $f(\mathbf{p})$, relaxed sphere tracing assumes that in many cases, the actual distance to the next surface is larger than the local SDF value, allowing for a larger, "relaxed" step [S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $\mathbf{p} \in \mathbb{R}^3$.
    *   Ray direction $\mathbf{v} \in \mathbb{R}^3$ with $|\mathbf{v}| = 1$.
    *   SDF estimate function $f: \mathbb{R}^3 \to \mathbb{R}$.
    *   Relaxation parameter $\omega \ge 0$.
    *   Maximum number of iterations $i_{max}$.
    *   Convergence threshold $\epsilon$ (implied by "not too small").
*   **Outputs**:
    *   Total distance traveled $t \in [0, \infty)$.
    *   Hit status (implicit in termination condition).

### Data Structures
No complex data structures are required. The algorithm maintains scalar variables for the current ray position $t$, the previous SDF value $r_i$, and the current SDF value $r_{i+1}$.

### Algorithm Steps
1.  **Initialization**: Set total distance $t = 0$, iteration counter $i = 0$. Evaluate the SDF at the ray origin to get the initial radius $r_0 = f(\mathbf{p})$. Set $r_{i+1} = \infty$ (or a sufficiently large value) to ensure the first iteration proceeds with the relaxed step [S1, unofficial].
2.  **Iteration Loop**: While $i < i_{max}$ and $r_{i+1}$ is not "too small" (i.e., $r_{i+1} \ge \epsilon$):
    a.  **Calculate Relaxed Step**: Compute the candidate step size $d_i$ by scaling the previous SDF value $r_i$ by the relaxation factor:
        $$ d_i = r_i + \omega \cdot r_i = r_i(1 + \omega) $$
        [S1, S2, unofficial]
    b.  **Advance Ray**: Move the ray forward by $d_i$. The new point is $\mathbf{p}_{new} = \mathbf{p} + (t + d_i)\mathbf{v}$.
    c.  **Evaluate SDF**: Compute the new SDF value $r_{i+1} = f(\mathbf{p}_{new})$.
    d.  **Safety Check (Disjoint Sphere Test)**: Verify if the unbounding spheres of the previous step and the current step are disjoint. The condition for safety is that the sum of the radii must be at least the distance traveled ($d_i$). If $d_i > r_i + r_{i+1}$, the spheres are disjoint, meaning the ray might have skipped an intersection.
        *   **Fallback**: If disjoint, discard the relaxed step. Revert to the standard sphere tracing step size:
            $$ d_i \leftarrow r_i $$
            Re-evaluate the SDF at the new position using this standard step:
            $$ r_{i+1} \leftarrow f(\mathbf{p} + (t + r_i)\mathbf{v}) $$
            [S1, S2, unofficial]
    e.  **Update State**: Add the confirmed step size to the total distance: $t \leftarrow t + d_i$. Update the previous radius for the next iteration: $r_i \leftarrow r_{i+1}$. Increment $i$.
3.  **Termination**: If $r_{i+1} < \epsilon$, the surface has been hit. Return $t$. If $i_{max}$ is reached, return $t$ as the maximum distance or indicate miss.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def relaxed_sphere_tracing(ray_origin, ray_direction, sdf, max_steps, epsilon, omega):
    """
    Relaxed sphere tracing algorithm.
    
    Args:
        ray_origin: vec3, starting point of the ray.
        ray_direction: vec3, normalized direction vector.
        sdf: callable, returns distance to surface.
        max_steps: int, maximum number of iterations.
        epsilon: float, convergence threshold for surface hit.
        omega: float, relaxation parameter (e.g., 0.6).
        
    Returns:
        float: distance traveled along the ray.
    """
    t = 0.0
    i = 0
    
    # Initial evaluation
    # r_prev corresponds to r_i in the source notation
    # We initialize r_next to infinity to force the first step to use the relaxed formula
    # based on r_prev. However, standard practice is to evaluate r_prev first.
    # Source [S1, unofficial] initializes r_{i+1} = C-1 (likely infinity/large constant) and r_i = 0?
    # Actually, source says: r_i = 0, r_{i+1} = C-1. This implies the first step 
    # uses r_i=0? No, that would yield d=0. 
    # Let's look at Algorithm 2 in [S1, unofficial]:
    # i=0, r_i=0, r_{i+1}=C-1 (large), t=0.
    # Loop: d_i = r_i + omega * r_i = 0. This seems wrong for the first step.
    # Correction: In standard sphere tracing, the first step is f(p).
    # The source notation might imply r_i is the radius of the *previous* unbounding sphere.
    # For the first step, there is no previous sphere. 
    # Let's assume standard initialization: evaluate f(ray_origin) first.
    
    r_prev = sdf(ray_origin)
    
    for _ in range(max_steps):
        if r_prev < epsilon:
            return t
        
        # Calculate relaxed step size
        d_relaxed = r_prev * (1.0 + omega)
        
        # Advance ray
        t_new = t + d_relaxed
        p_new = ray_origin + t_new * ray_direction
        
        # Evaluate SDF at new position
        r_next = sdf(p_new)
        
        # Safety check: disjoint spheres
        # Condition: d_i > r_i + r_{i+1} implies disjoint
        if d_relaxed > r_prev + r_next:
            # Fallback to standard step
            d_step = r_prev
            t_new = t + d_step
            p_new = ray_origin + t_new * ray_direction
            r_next = sdf(p_new)
        else:
            d_step = d_relaxed
            
        t = t_new
        r_prev = r_next
        
    return t
```

### Parameters
*   **$\omega$ (Relaxation Parameter)**: A scalar that determines the magnitude of the step increase. A value of $\omega = 0.6$ is cited as an example, resulting in a step size multiplier of $1.6$ [S2, unofficial]. The optimal value depends on the smoothness of the SDF; higher values increase speed but increase the frequency of fallbacks [S1, unofficial].
*   **$\epsilon$**: Convergence threshold. Not given in the sources, but typically $0.001$ or similar small value [G].

### Edge Cases and Pitfalls
*   **Disjoint Spheres**: The primary pitfall is skipping intersections. The fallback mechanism ensures correctness but can degrade performance if the SDF changes rapidly or if $\omega$ is too large, causing frequent re-evaluations [S1, unofficial].
*   **Initial Step**: The source pseudocode initialization ($r_i=0$) is ambiguous for the first iteration. Implementers should evaluate the SDF at the ray origin before entering the loop to establish $r_{prev}$ [G].
*   **Precision**: Floating-point errors in the disjoint sphere check ($d_i > r_i + r_{i+1}$) may cause unnecessary fallbacks or, rarely, missed hits. A small tolerance may be added to the comparison [G].

## Subtopics

### Enhanced Sphere Tracing
A variant that approximates planar surfaces optimally by ensuring consecutive unbounding spheres are tangential rather than just intersecting. It is decisively better in performance metrics (error-decrease velocity) than relaxed sphere tracing across all iteration counts. [Enhanced Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md)

### Basic Sphere Tracing
The foundational algorithm that uses the SDF value directly as the step size. Relaxed sphere tracing is a direct optimization of this method [S1, unofficial]. [Sphere Tracing and Step Rules](9643-sphere-tracing-and-step-rules.md)

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of SDF evaluations. $N$ is reduced compared to basic sphere tracing by a factor of approximately $1/(1+\omega)$ in empty space, but increases due to fallbacks near complex geometry.
*   **Memory Complexity**: $O(1)$. Only scalar variables are stored.
*   **Performance**: The algorithm provides a performance increase over basic sphere tracing [S2, unofficial]. However, [Enhanced Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md) outperforms relaxed sphere tracing on every iteration count because it approximates smooth surfaces optimally and rarely branches [S1, unofficial].
*   **Strengths**: Simple to implement; significant speedup in large empty volumes; maintains safety guarantees.
*   **Limitations**: Performance degrades near complex surfaces due to frequent fallbacks; less efficient than enhanced sphere tracing for smooth surfaces [S1, unofficial].
*   **When to use**: When scene complexity is low or empty space is dominant, and implementation simplicity is prioritized over maximum convergence speed.
*   **When not to use**: For scenes with high-frequency details or smooth surfaces where enhanced sphere tracing is superior [S1, unofficial].

## Background
Relaxed sphere tracing was introduced by Keinert et al. in 2014 [S1, S2, unofficial]. It was later popularized in the "Claybook" project for real-time simulation-based computer games [S2, unofficial]. The method was analyzed and compared against enhanced sphere tracing by Csaba Bálint and Gábor Valasek in 2018, who demonstrated that while relaxed sphere tracing is faster than basic sphere tracing, it is inferior to their proposed enhanced variant [S1, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Csaba Bálint 1. Introduction 2. Sphere Tracing 3. Relaxed ..., https://people.inf.elte.hu/csabix/publications/articles/WSPS_2019_Poster.pdf, fetched 2026-10-29 (unofficial)
