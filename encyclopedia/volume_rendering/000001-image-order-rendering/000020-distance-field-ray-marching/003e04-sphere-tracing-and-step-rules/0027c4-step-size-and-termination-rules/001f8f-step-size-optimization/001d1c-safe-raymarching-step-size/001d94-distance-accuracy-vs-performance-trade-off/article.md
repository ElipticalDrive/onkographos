# Distance Accuracy vs Performance Trade-off

Also called: Distance Estimate Accuracy, Distance accuracy vs performance, Sphere Tracing Performance Analysis, Underestimate vs Exact SDF Performance

## Overview
This method analyzes the inverse relationship between the accuracy of a Signed Distance Function (SDF) evaluation and the computational cost of sphere tracing [S2]. It establishes that underestimating the distance to the surface forces the ray marcher to take smaller steps, significantly increasing the number of iterations required for convergence [S1]. The core insight is that a slower, exact SDF evaluation can outperform a faster, inaccurate distance bound if the inaccuracy forces a disproportionate increase in step count [S1].

## How it works
The method relies on the geometric constraint that the step size $s$ in sphere tracing must satisfy $s \le d(\mathbf{p})$, where $d(\mathbf{p})$ is the distance estimate at the current ray position $\mathbf{p}$ [G].

**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
- SDF function $d(\mathbf{p})$ which may be an approximation or exact [S1].
- Convergence threshold $\epsilon$ [G].
- Maximum traversal distance $\text{max\_dist}$ [G].
- Maximum iteration count $\text{max\_steps}$ [G].

**Outputs:**
- Boolean indicating surface intersection [G].
- Number of steps taken [G].

**Data Structures:**
- Current ray position $\mathbf{p}$ (vector) [G].
- Step counter $n$ (integer) [G].

**Numbered Steps:**
1. Initialize $\mathbf{p} = \mathbf{o}$, normalize $\mathbf{d}$, and set $n = 0$ [G].
2. Evaluate distance estimate $d_{est} = d(\mathbf{p})$ [G].
3. Clamp negative estimates to zero: $d_{est} \leftarrow \max(d_{est}, 0)$ [G].
4. If $d_{est} < \epsilon$, terminate successfully [G]. If $\|\mathbf{p} - \mathbf{o}\| > \text{max\_dist}$ or $n \ge \text{max\_steps}$, terminate unsuccessfully [G].
5. Advance ray: $\mathbf{p} \leftarrow \mathbf{p} + d_{est} \cdot \mathbf{d}$ [G].
6. Increment $n \leftarrow n + 1$ [G].
7. Repeat from step 2 [G].

**Formulas:**
The relationship between distance accuracy and step count is derived from the convergence rate. Experiments on a sphere show that halving the distance estimate doubles the number of steps [S1, S2]. Thus, the step count scales inversely with the estimate magnitude:
$$ N_{steps} \propto \frac{1}{d_{est}} $$
Where $N_{steps}$ is the number of iterations and $d_{est}$ is the average distance estimate magnitude [S1, S2].

**Parameters:**
- $\epsilon$: Convergence threshold [G].

**Edge Cases and Pitfalls:**
- **Severe Underestimation:** If $d_{est}$ is significantly smaller than the true distance $d_{true}$, the ray takes many small steps, wasting computation [S1]. For example, a bound that returns at worst 1% of the true distance may perform worse than an exact function in specific cases [S1, S2].
- **Exact vs. Approximate Cost:** The total cost is $Cost = N_{steps} \times Cost_{eval}$ [G]. A slower exact function may have higher $Cost_{eval}$ but much lower $N_{steps}$, resulting in lower total time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sphere_trace(ray_origin, ray_direction, sdf_function, epsilon, max_dist, max_steps):
    """
    Demonstrates the trade-off by comparing step counts for different SDF accuracies.
    """
    # Normalize direction to ensure step size equals distance estimate
    d_len = math.sqrt(sum(x**2 for x in ray_direction))
    d = [x / d_len for x in ray_direction]
    
    p = list(ray_origin)
    steps = 0
    
    while steps < max_steps:
        # Evaluate distance. The accuracy of this call determines performance.
        dist = sdf_function(p)
        
        # Clamp negative distances to zero to prevent stepping inside the surface
        dist = max(dist, 0.0)
        
        if dist < epsilon:
            return True, steps
            
        # Check maximum traversal distance
        ray_len = math.sqrt(sum((p[i] - ray_origin[i])**2 for i in range(3)))
        if ray_len > max_dist:
            return False, steps
            
        # Advance ray by the estimated distance
        p = [p[i] + dist * d[i] for i in range(3)]
        steps += 1
        
    return False, steps
```

## Subtopics

### [Adaptive Convergence Threshold](2727-adaptive-convergence-threshold.md)
Adaptive Convergence Threshold is a stopping condition for distance-field ray marching that determines when the ray has sufficiently converged to the surface to halt further computation. It solves the trade-off between rendering accuracy and computational cost by dynamically calculating a threshold $\epsilon$ based on camera parameters, ensuring that the ray front ends of neighboring pixels do not intersect. Unlike fixed thresholds, this method scales with the scene geometry and camera setup to prevent both dilation (loss of detail) and erosion (incomplete geometry).

## Operating characteristics
**Time Complexity:**
The number of steps $N$ is inversely proportional to the accuracy of the distance estimate [S1]. Halving the distance estimate doubles the number of steps [S1, S2].

**Measured Performance:**
- **Sphere Experiments:** Halving the distance estimate doubles the number of steps [S1, S2].
- **Noise Functions:** In experiments with fractal noise, the rendering time for $1/f^2$ noise was 7.3 times that of single noise, despite an expected 12x increase due to 6 invocations per evaluation [S2]. For $1/f$ noise, rendering time was 27.5 times single noise time (expected 36x) [S2]. This indicates that while evaluation cost scales linearly with complexity, the step count reduction from better distance estimates (or other optimizations) can mitigate total time [S2].
- **Ellipsoid Case:** For an ellipsoid with axes 100, 100, 1, a fast signed distance bound (returning at worst 1% of the true distance) performs worse than an exact SDF using Newton iterations, despite the exact function being slower per evaluation [S1, S2].

**Strengths and Limitations:**
- **Strength:** Provides a theoretical basis for choosing SDF implementations. It highlights that "fast" functions are not always optimal [S1].
- **Limitation:** Requires accurate knowledge of the SDF's error bounds to predict performance. The trade-off is non-linear and scene-dependent [S1].

**When to Use:**
- When optimizing implicit surface rendering performance [S1].
- When choosing between a closed-form approximate SDF and an iterative exact SDF [S1].

**When Not to Use:**
- When the SDF error is negligible compared to other bottlenecks (e.g., shading, memory bandwidth) [G].
- When the scene geometry does not exhibit large disparities between bound and exact distances [S1].

## Background
Originating from the foundational work on sphere tracing by Hart and North [S1, S2], this analysis was presented in "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" (1992/1994) [S1, S2]. The paper established sphere tracing as a competitive implicit surface renderer and detailed the impact of distance estimate accuracy on convergence rates [S2]. It is widely cited in graphics literature for establishing the performance characteristics of ray marching algorithms [S1, S2].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
