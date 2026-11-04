# Distance Accuracy vs Performance Trade-off

Also called: Distance Estimate Accuracy, Distance accuracy vs performance, Sphere Tracing Performance Analysis, Underestimate vs Exact SDF Performance

## Overview
This method analyzes the inverse relationship between the accuracy of a Signed Distance Function (SDF) evaluation and the computational cost of sphere tracing. It establishes that underestimating the distance to the surface forces the ray marcher to take smaller steps, significantly increasing the number of iterations required for convergence. The core insight is that a slower, exact SDF evaluation can outperform a faster, inaccurate distance bound if the inaccuracy forces a disproportionate increase in step count.

## How it works
The method relies on the geometric constraint that the step size $s$ in sphere tracing must satisfy $s \le d(\mathbf{p})$, where $d(\mathbf{p})$ is the distance estimate at the current ray position $\mathbf{p}$.

**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- SDF function $d(\mathbf{p})$ which may be an approximation or exact.
- Convergence threshold $\epsilon$.

**Outputs:**
- Intersection point or null.
- Number of steps taken.

**Data Structures:**
- Current ray position $\mathbf{p}$ (vector).
- Step counter $n$ (integer).

**Numbered Steps:**
1. Initialize $\mathbf{p} = \mathbf{o}$ and $n = 0$.
2. Evaluate distance estimate $d_{est} = d(\mathbf{p})$.
3. If $d_{est} < \epsilon$, terminate successfully. If $d_{est} > \text{max\_dist}$, terminate unsuccessfully.
4. Advance ray: $\mathbf{p} \leftarrow \mathbf{p} + d_{est} \cdot \mathbf{d}$.
5. Increment $n$.
6. Repeat from step 2.

**Formulas:**
The relationship between distance accuracy and step count is derived from the convergence rate. If the distance estimate is halved, the number of steps doubles [S1, S2].
$$ N_{steps} \propto \frac{1}{d_{est}} $$
Where $N_{steps}$ is the number of iterations and $d_{est}$ is the average distance estimate magnitude.

**Parameters:**
- $\epsilon$: Convergence threshold. Typical values depend on scene scale, but the trade-off analysis holds regardless of specific $\epsilon$ as long as it is constant relative to the distance estimates [S1].

**Edge Cases and Pitfalls:**
- **Severe Underestimation:** If $d_{est}$ is significantly smaller than the true distance $d_{true}$, the ray takes many small steps, wasting computation. For example, a bound that returns at worst 1% of the true distance will require roughly 100x more steps than an exact function [S1, S2].
- **Exact vs. Approximate Cost:** The total cost is $Cost = N_{steps} \times Cost_{eval}$. A slower exact function may have higher $Cost_{eval}$ but much lower $N_{steps}$, resulting in lower total time.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_trace_with_accuracy_tradeoff(ray_origin, ray_direction, sdf_function, epsilon, max_steps):
    """
    Demonstrates the trade-off by comparing step counts for different SDF accuracies.
    """
    p = ray_origin
    steps = 0
    
    while steps < max_steps:
        # Evaluate distance. The accuracy of this call determines performance.
        # If sdf_function is a fast bound, it may return d_est << d_true.
        # If sdf_function is exact, it returns d_est ~= d_true.
        dist = sdf_function(p)
        
        if dist < epsilon:
            return True, steps
        
        # Advance by the estimated distance
        p = p + dist * ray_direction
        steps += 1
        
    return False, steps
```

## Subtopics

### [Adaptive Convergence Threshold](2727-adaptive-convergence-threshold.md)
Adaptive Convergence Threshold is a stopping condition for distance-field ray marching that determines when the ray has sufficiently converged to the surface to halt further computation. It solves the trade-off between rendering accuracy and computational cost by dynamically calculating a threshold $\epsilon$ based on camera parameters, ensuring that the ray front ends of neighboring pixels do not intersect. Unlike fixed thresholds, this method scales with the scene geometry and camera setup to prevent both dilation (loss of detail) and erosion (incomplete geometry).

## Operating characteristics
**Time Complexity:**
The number of steps $N$ is inversely proportional to the accuracy of the distance estimate. If the estimate is $k$ times smaller than the true distance, the step count increases by approximately $k$ [S1, S2].

**Measured Performance:**
- **Sphere Experiments:** Halving the distance estimate doubles the number of steps [S1, S2].
- **Noise Functions:** In experiments with fractal noise, the rendering time for $1/f^2$ noise was 7.3 times that of single noise, despite an expected 12x increase due to 6 invocations per evaluation. For $1/f$ noise, rendering time was 27.5 times single noise time (expected 36x) [S2]. This indicates that while evaluation cost scales linearly with complexity, the step count reduction from better distance estimates (or other optimizations) can mitigate total time.
- **Ellipsoid Case:** For an ellipsoid with axes 100, 100, 1, a fast signed distance bound (returning at worst 1% of the true distance) performs worse than an exact SDF using Newton iterations, despite the exact function being slower per evaluation [S1, S2].

**Strengths and Limitations:**
- **Strength:** Provides a theoretical basis for choosing SDF implementations. It highlights that "fast" functions are not always optimal.
- **Limitation:** Requires accurate knowledge of the SDF's error bounds to predict performance. The trade-off is non-linear and scene-dependent.

**When to Use:**
- When optimizing implicit surface rendering performance.
- When choosing between a closed-form approximate SDF and an iterative exact SDF.

**When Not to Use:**
- When the SDF error is negligible compared to other bottlenecks (e.g., shading, memory bandwidth).
- When the scene geometry does not exhibit large disparities between bound and exact distances.

## Background
Originating from the foundational work on sphere tracing by Hart and North [S1, S2], this analysis was presented in "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" (1992/1994). The paper established sphere tracing as a competitive implicit surface renderer and detailed the impact of distance estimate accuracy on convergence rates. It is widely cited in graphics literature for establishing the performance characteristics of ray marching algorithms.

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
