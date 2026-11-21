# Lipschitz Step Size Ray Marching

Also called: Hart's Sphere Tracing, Ray Marching with Lipschitz Step Size, Ray marching with Lipschitz bound, SDF ray marching with Lipschitz constant, Safe Raymarching with Lipschitz Bounds, Step-size ray marching

## Overview
Lipschitz Step Size Ray Marching is a robust algorithm for finding the first intersection between a ray and an implicit surface defined by a signed distance field (SDF) $f(\mathbf{p}) = 0$ [S1, S3]. It guarantees that the ray never penetrates or overshoots the surface by computing the maximum safe step size using a global Lipschitz constant $\lambda$ [S1, S3]. Unlike adaptive methods that estimate local gradients, this method relies on a pre-computed global bound, making it simple to implement and numerically stable, though potentially less efficient for complex scenes with varying feature scales [S1, S3].

## How it works
### Inputs and Outputs
- **Inputs**:
  - Ray origin $\mathbf{o}$ and normalized direction $\mathbf{u}$ [S1].
  - Ray domain bounds $[t_{min}, t_{max}]$ (intersection with the scene bounding volume $\Omega$) [S1].
  - Global Lipschitz constant $\lambda$ for the field function $f$ over $\Omega$ [S1].
  - Hit threshold $\mu$ (distance tolerance) [S1][S2, unofficial].
  - Maximum iteration count $N_{max}$ [S2, unofficial].
- **Outputs**:
  - Intersection distance $t$ along the ray, or a failure indicator if no intersection is found within bounds or iterations [S1].

### Data Structures
- **Ray**: Defined by origin $\mathbf{o} \in \mathbb{R}^3$ and direction $\mathbf{u} \in \mathbb{R}^3$ with $\|\mathbf{u}\| = 1$ [S1].
- **Field Function**: $f: \mathbb{R}^3 \to \mathbb{R}$, representing the signed distance to the implicit surface [S1].
- **Parameters**:
  - $\lambda$: Scalar, the global Lipschitz bound. Typical value: $1.0$ for normalized SDFs [S2, unofficial].
  - $\mu$: Scalar, the convergence threshold. Typical value: $0.01$ [S2, unofficial].
  - $N_{max}$: Integer, maximum steps. Typical value: $100$ [S2, unofficial].

### Mathematical Background
A function $f: \mathbb{R}^3 \to \mathbb{R}$ is Lipschitz continuous over a domain $\Omega$ if there exists a constant $\lambda \ge 0$ such that for all $\mathbf{p}, \mathbf{q} \in \Omega$:
$$
|f(\mathbf{p}) - f(\mathbf{q})| \leq \lambda \|\mathbf{p} - \mathbf{q}\|
$$
The constant $\lambda$ is the minimum bound satisfying this inequality [S1].

### Algorithm Steps
1. **Initialization**:
   - Compute the intersection of the ray with the domain $\Omega$ to determine $[t_{min}, t_{max}]$ [S1].
   - Set current distance $t = t_{min}$ [S1].

2. **Iteration Loop**:
   - While $t < t_{max}$ and iteration count $< N_{max}$:
     1. Evaluate the field function at the current point on the ray: $\mathbf{p} = \mathbf{o} + t\mathbf{u}$, then $d = f(\mathbf{p})$ [S1].
     2. **Termination Check**: If $|d| < \mu$, the ray has converged to the surface. Return $t$ as the intersection distance [S1].
     3. **Step Calculation**: Compute the step size $s$ using the Lipschitz bound:
        $$
        s = \frac{|d|}{\lambda}
        $$
        This step size guarantees that the next point is not closer to the surface than the current distance allows, preventing overshoot [S1, S3].
     4. **Advance**: Update $t \leftarrow t + s$ [S1].

3. **Failure**:
   - If the loop terminates without satisfying the hit condition, return failure (no intersection found within bounds or iterations) [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lipschitz_ray_march(ray_origin, ray_direction, t_min, t_max, field_func, lambda_val, mu, max_iterations):
    """
    Ray march using Lipschitz step size.
    
    Args:
        ray_origin: Vector3, origin of the ray.
        ray_direction: Vector3, normalized direction of the ray.
        t_min: float, minimum distance along ray (start of domain).
        t_max: float, maximum distance along ray (end of domain).
        field_func: function, f(p) -> float, returns signed distance.
        lambda_val: float, global Lipschitz constant.
        mu: float, hit threshold.
        max_iterations: int, maximum number of steps.
        
    Returns:
        float: Intersection distance t, or None if not found.
    """
    t = t_min
    for i in range(max_iterations):
        # Check domain bounds before evaluation
        if t > t_max:
            return None
            
        # 1. Evaluate field at current position
        p = ray_origin + ray_direction * t
        d = field_func(p)
        
        # 2. Check for intersection
        if abs(d) < mu:
            return t
        
        # 3. Compute step size
        step = abs(d) / lambda_val
        
        # 4. Advance along ray
        t = t + step
        
    return None
```

### Edge Cases and Pitfalls
- **Grazing Rays**: The algorithm may require an arbitrarily large number of steps for rays that graze the surface, as the step size $s$ becomes very small relative to the remaining distance [S3].
- **Large Radius Differences**: When applied to primitives with significantly different radii (e.g., SCALIS primitives), the Lipschitz constant per primitive is inversely proportional to the prescribed radius, creating unnecessarily small steps globally [S3].
- **Density Fields**: When used with density fields instead of distance fields, the algorithm suffers from null gradients near kernel support boundaries and difficulty in computing tight Lipschitz bounds for N-ary summation operators [S3].
- **Overestimation of $\lambda$**: If $\lambda$ is significantly larger than the true local Lipschitz constant, the step size $s$ becomes smaller than necessary, increasing the number of iterations without improving accuracy [S1].

## Subtopics

### [Distance Accuracy vs Performance Trade-off](1546-distance-accuracy-vs-performance-trade-off.md)
Discusses the balance between step size precision and computational cost.

### [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md)
Extends this method by computing local Lipschitz bounds to improve efficiency.

### [Gradient-based Sphere Tracing with Lipschitz Constraints](1936-gradient-based-sphere-tracing-with-lipschitz-constraints.md)
Uses gradient information to refine step sizes while maintaining safety guarantees.

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of iterations required to converge. $N$ depends on the scene complexity, ray angle, and the tightness of $\lambda$ [S1, S3].
- **Memory Complexity**: $O(1)$ auxiliary space, excluding the storage for the field function [S1].
- **Strengths**:
  - Guaranteed no penetration of the surface [S1, S3].
  - Simple implementation, no pre-processing required [S1].
  - Robust for direct content creation in shaders [S3].
- **Limitations**:
  - Inefficient for grazing rays [S3].
  - For SCALIS primitives with large radius differences, the Lipschitz constant per primitive is inversely proportional to the prescribed radius, creating unnecessarily small steps globally [S3].
  - Not suitable for density fields with complex kernel compositions without modification [S3].
- **When to Use**: Ideal for simple SDFs and real-time shaders where simplicity and robustness are prioritized over maximum performance [S1, S3].
- **When Not to Use**: Scenes with large variations in feature scale, grazing ray scenarios, or density fields with complex kernel functions [S3].

## Background
The algorithm was introduced by Hart in 1996, often referred to as "Hart's Sphere Tracing", and formalized with the Lipschitz bound guarantee [S3]. It belongs to the ray-marching family and is widely used in real-time rendering, including games and shaders, due to its simplicity and robustness [S3]. Extensions such as Segment Tracing have been proposed to address its limitations by using local Lipschitz bounds [S1, S3].

## Sources
[S1] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
[S2] skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, https://github.com/skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
