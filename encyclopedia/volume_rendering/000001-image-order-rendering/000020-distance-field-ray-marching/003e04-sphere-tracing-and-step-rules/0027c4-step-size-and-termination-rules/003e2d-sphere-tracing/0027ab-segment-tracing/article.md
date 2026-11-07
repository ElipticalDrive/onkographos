# Segment Tracing
## Overview
Segment Tracing is a ray marching algorithm that computes local Lipschitz bounds on candidate segments to determine adaptive step sizes, reducing the number of field function queries compared to global Lipschitz bound methods like Sphere Tracing [S1]. It works for a particular class of hierarchical skeletal implicit surfaces, leveraging the composition of distance functions and kernel functions to compute directional Lipschitz bounds [S2, S3]. Unlike Sphere Tracing, which uses a global bound that limits step size efficiency, Segment Tracing allows larger, safer steps by adapting to the local gradient and field values [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $\delta(t)$, a hierarchical skeletal implicit surface model where primitives and operators support local Lipschitz bound queries, an initial candidate step distance $\varepsilon_0$, an amplification factor $\kappa$, and a threshold $\mu$ [S1].
*   **Output:** The intersection parameter $t$ along the ray, or the domain exit parameter [S1].

**Data Structures:**
The method requires the implicit model to implement specific queries for each primitive and operator node:
1.  Field function evaluation $f(p)$ [S1].
2.  Local Lipschitz bound over a segment $\lambda(e)$, where $e$ is a segment of the ray [S1].
3.  Local Lipschitz bound over a sphere $\lambda(S)$ (used in some implementations for tighter bounds) [S1].

**Numbered Steps:**
1.  Compute the intersection between the ray and the domain $[t_-, t_+] = \Delta \cap \Omega$, and initialize $t = t_-$ [S1].
2.  Set the initial candidate stepping distance $\varepsilon_0$ [S1].
3.  While $t < t_+$:
    1.  Compute the field function value $f(t)$ at the current position [S1].
    2.  If $|f(t)| < \mu$, an intersection has occurred; terminate and return $t$ [S1].
    3.  Define the candidate segment $e(t, \varepsilon_i) = [\delta(t), \delta(t + \varepsilon_i)]$ [S1].
    4.  Compute the local directional Lipschitz bound $\lambda(e(t, \varepsilon_i))$ for the candidate segment [S1]. This step leverages the fact that primitives are defined as a composition of a distance function and a kernel function [S2].
    5.  Evaluate $f(p_i^*)$ (where $p_i^*$ is the current point) and compute the safe stepping increment $s(t, \varepsilon_i)$ using the formula:
        $$s(t, \varepsilon_i) = \min \left( \frac{|f(t)|}{\lambda(e(t, \varepsilon_i))}, \varepsilon_i \right)$$
        where $s$ is the effective stepping distance, $f(t)$ is the field function value, $\lambda(e)$ is the local Lipschitz bound on segment $e$, and $\varepsilon_i$ is the candidate distance [S1].
    6.  Update the position: $t \leftarrow t + s(t, \varepsilon_i)$ [S1].
    7.  Compute the next candidate stepping distance $\varepsilon_{i+1} = \kappa \cdot s(t, \varepsilon_i)$, where $\kappa$ is the geometric amplification factor [S1].

**Parameters:**
*   $\mu$: The threshold value from which an intersection is considered to have occurred ($|f(t)| < \mu$) [S1].
*   $\varepsilon_0$: The initial candidate step distance. Its value has little impact on performance; if too large, the initial Lipschitz bound is large, leading to small subsequent steps [S1].
*   $\kappa$: The geometric amplification factor for the next candidate step. Typical value is $\kappa \approx 2$. Must be $> 1$ [S1].

**Pitfalls and Edge Cases:**
*   If $\varepsilon_0$ is too large, the computed Lipschitz bound $\lambda(e(t_0, \varepsilon_0))$ is large (upper bound), causing the next candidate distance $\varepsilon_1$ to be small. If $\varepsilon_0 \to \infty$, $\lambda(e)$ becomes the global bound [S1].
*   If the computation of local bounds is too complex or computationally intensive for a given operator or primitive, the algorithm automatically reverts to the standard Sphere Tracing framework locally [S1].
*   The method works only for a particular class of hierarchical skeletal implicit surfaces and is not directly applicable to a wide range of volumes [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def segment_tracing(ray_origin, ray_direction, domain_min, domain_max, 
                    field_function, local_lipschitz_segment, 
                    epsilon_0, kappa, mu):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    domain_min: float (t_min)
    domain_max: float (t_max)
    field_function: function(p) -> float
    local_lipschitz_segment: function(t, epsilon) -> float
    epsilon_0: float
    kappa: float (> 1)
    mu: float
    """
    t = domain_min
    epsilon = epsilon_0
    
    while t < domain_max:
        # Evaluate field function at current position
        p = ray_origin + t * ray_direction
        f_val = field_function(p)
        
        # Check for intersection
        if abs(f_val) < mu:
            return t
        
        # Compute local Lipschitz bound for candidate segment [t, t + epsilon]
        lambda_bound = local_lipschitz_segment(t, epsilon)
        
        # Compute safe stepping increment
        if lambda_bound == 0:
            step = epsilon
        else:
            step = min(abs(f_val) / lambda_bound, epsilon)
        
        # Clamp step to domain boundary to prevent overshoot and fix boundary precision
        step = min(step, domain_max - t)
        
        # Update position
        t = t + step
        
        # Amplify candidate step for next iteration
        epsilon = kappa * step
        
    return domain_max # No intersection within domain
```

## Subtopics

### Sphere Tracing
The parent method using a global Lipschitz bound. Choose when simplicity is preferred and local bound computation is not feasible. [1674-sphere-tracing.md] [G]

### Cone Intersection
Uses cone intersection tests for tighter bounds. Choose when higher precision is needed for complex primitives. [1553-cone-intersection.md] [G]

### Quadric Tracing
Uses quadric approximations for step size. Choose for quadric-based implicit surfaces. [1866-quadric-tracing.md] [G]

### Chamfer-L1 Distance Computation
Uses L1 distance fields. Choose for grid-based volumes where L1 metrics are appropriate. [2752-chamfer-l1-distance-computation.md] [G]

### Over-relaxation Sphere Tracing
Uses over-stepping heuristics. Choose for a simple speedup over standard Sphere Tracing without local bound computation. [3805-over-relaxation-sphere-tracing.md] [G]

### Enhanced Sphere Tracing
Reduces queries by a factor of 1.2. Choose for a minor improvement over Sphere Tracing. [3811-enhanced-sphere-tracing.md] [G]

### Non-Linear Sphere Tracing
Uses non-linear transformations. Choose for specific non-linear field representations. [4079-non-linear-sphere-tracing.md] [G]

## Operating characteristics
*   **Time Complexity**: Increases the overall complexity of the shader due to the implementation of extra functions for computing the Lipschitz bound over a segment, a ray, or a spherical domain [S1].
*   **Memory Complexity**: Does not require accelerating data structures such as voxel grids or octrees for storing the local Lipschitz bound [S1].
*   **Measured Performance**: Reduces field function queries by up to three orders of magnitude compared to Sphere Tracing, resulting in accelerations of up to $\times 10$, despite the extra computational cost of Lipschitz bound queries [S1].
*   **Scaling**: Performance gains are most significant for hierarchical skeletal implicit surfaces where local bounds can be computed efficiently [S1].
*   **Strengths**: Significantly reduces the number of field function queries; no need for precomputed acceleration structures; adapts to local gradient variations [S1].
*   **Limitations**: Requires primitives and operators to implement complex local Lipschitz bound queries; not directly applicable to a wide range of volumes; overhead from bound computation can negate benefits if not optimized [S1, S3].
*   **When to use**: When rendering hierarchical skeletal implicit surfaces and field function evaluation is expensive [S1].
*   **When not to use**: When the implicit model does not support local Lipschitz bound queries, or when the overhead of computing these bounds exceeds the savings from fewer field evaluations [S1].

## Background
Introduced in [S1], the method was developed to address the limitations of Sphere Tracing, which uses a global Lipschitz bound that limits efficiency. The method leverages the structure of hierarchical skeletal implicit surfaces, where primitives are defined as a composition of a distance function and a kernel function, allowing for the computation of local directional Lipschitz bounds [S2]. It has been used in the context of ray tracing compactly supported primitives combined in a Blobtree [S2].

## Sources
[S1] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
