# Lipschitz Step Size Ray Marching

Also called: Hart's Sphere Tracing, Ray Marching with Lipschitz Step Size, Ray marching with Lipschitz bound, SDF ray marching with Lipschitz constant, Safe Raymarching with Lipschitz Bounds, Step-size ray marching

## Overview
Lipschitz Step Size Ray Marching is an adaptive ray marching algorithm that finds the first intersection of a ray with an implicit surface defined by $f(\mathbf{p}) = 0$ by guaranteeing no surface penetration through the use of a global Lipschitz bound [S1, S3]. It sets itself apart from naive fixed-step marching by using the field value to determine the maximum safe step size, ensuring convergence without overshooting the root [S2, S4, unofficial]. Unlike [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md) which computes local bounds, this method relies on a single global constant, making it simpler but potentially less efficient for complex scenes [S1].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and normalized direction $\mathbf{u}$.
- Implicit surface function $f: \mathbb{R}^3 \to \mathbb{R}$, where the surface is $S = \{\mathbf{p} \in \mathbb{R}^3 \mid f(\mathbf{p}) = 0\}$ [S1].
- Global Lipschitz constant $\lambda$ for $f$ over the domain $\Omega$ [S1].
- Intersection threshold $\mu$ [S1].
- Maximum number of iterations `maxIterations` [S2, unofficial].
- Minimum hit distance `minDistance` [S2, unofficial].

**Outputs:**
- Parametric distance $t$ to the first intersection, or a failure indicator if no intersection is found within `maxIterations`.

**Data Structures:**
- Scalar $t$: Current parametric distance along the ray.
- Scalar $f_t$: Value of the field function at the current ray position.

**Definitions:**
- The Lipschitz condition states that for all $\mathbf{p}, \mathbf{q} \in \Omega$:
  $$ |f(\mathbf{p}) - f(\mathbf{q})| \leq \lambda \|\mathbf{p} - \mathbf{q}\| $$
  where $\lambda$ is the minimum bound satisfying this equation [S1].
- For a 1D function $g(t) = f(\mathbf{o} + t\mathbf{u})$, the Lipschitz assumption implies $|\frac{\partial g}{\partial t}| \leq 1$ if $\lambda=1$ [S4, unofficial]. Generally, the step size is scaled by $\lambda$.

**Parameters:**
- $\lambda$: Global Lipschitz bound. Typically 1.0 for normalized SDFs, but must be calculated or bounded for other fields [S1][S2, S4, unofficial].
- $\mu$: Threshold for intersection. Typically 0.01 [S2, unofficial].
- `maxIterations`: Maximum march steps. Typically 100 [S2, unofficial].
- `minDistance`: Minimum distance to consider a hit. Typically 0.01 [S2, unofficial].

**Algorithm Steps:**
1. Compute the intersection of the ray with the domain $\Omega$, resulting in the interval $[t_{-}, t_{+}] = \Delta \cap \Omega$ [S1].
2. Initialize $t = t_{-}$ [S1].
3. Loop up to `maxIterations` times:
   a. Compute $f_t = f(\mathbf{o} + t\mathbf{u})$ [S1].
   b. If $|f_t| < \mu$, an intersection is detected. Return $t$ [S1].
   c. If $t + \frac{|f_t|}{\lambda} > t_{+}$, the step would exceed the domain. Clamp $t$ to $t_{+}$ or terminate with failure [S1, G].
   d. Increment $t$ by the step size $s(t) = \frac{|f_t|}{\lambda}$ [S1].
   e. If $t < \text{minDistance}$, continue to next iteration [S2, G, unofficial].
4. If the loop completes without returning, return failure [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lipschitz_ray_march(ray_origin, ray_direction, field_function, lambda_bound, 
                        domain_min, domain_max, threshold=0.01, max_iterations=100):
    # 1. Compute ray-domain intersection [t-, t+]
    # For a simple axis-aligned bounding box or sphere, this is standard ray-box/sphere intersection.
    # Here we assume a generic interval [t_min, t_max] along the ray.
    t_min, t_max = compute_ray_domain_intersection(ray_origin, ray_direction, domain_min, domain_max)
    
    t = t_min
    
    for i in range(max_iterations):
        # 2. Compute field value at current position
        pos = ray_origin + t * ray_direction
        f_val = field_function(pos)
        
        # 3. Check for intersection
        if abs(f_val) < threshold:
            return t
        
        # 4. Compute safe step size
        # s(t) = |f(t)| / lambda
        step = abs(f_val) / lambda_bound
        
        # 5. Advance t
        t_next = t + step
        
        # 6. Boundary check
        if t_next > t_max:
            # Step would exceed domain; clamp or fail
            # Depending on implementation, one might return t_max or fail
            return None 
            
        t = t_next
        
        # 7. Minimum distance check (optional, prevents hitting origin if f(0) is small but not zero)
        if t < 0.01: # minDistance
            continue
            
    return None # Max iterations reached
```

**Edge Cases and Pitfalls:**
- **Grazing Rays:** The algorithm may require an arbitrarily large number of steps for grazing rays, as the field value decreases slowly relative to the distance traveled [S3].
- **Large Radius Differences:** For primitives with large radius differences (e.g., SCALIS primitives), the Lipschitz constant per primitive is inversely proportional to the radius. Using a global bound derived from the smallest primitive creates unnecessarily small steps globally [S3].
- **Density Fields:** When applied to density fields instead of distance fields, the algorithm suffers from null gradients near kernel support boundaries and difficulty in computing a tight Lipschitz bound for N-ary summation operators [S3].
- **Lipschitz Bound Accuracy:** If $\lambda$ is overestimated, steps become smaller than necessary, reducing performance. If underestimated, the surface may be missed (penetration) [S1].

## Subtopics

### SDF Erosion via Circular Sphere Placement
A post-processing technique applied after ray marching hit detection to erode the surface. It involves instantiating a base erosion sphere and placing multiple smaller spheres in a circular pattern around the hit point to create a roughened or eroded geometry [S2, unofficial].
- **Steps:** After detecting a hit at point $p$, instantiate `erosionSphere` with radius 0.05. Loop $j$ from 0 to 9: compute `angle = (2 * \pi * j) / 10`, `radius = 0.1`. Set sphere translation: $tx = -p[0] + radius \cdot \cos(angle)$, $ty = -p[1] + radius \cdot \sin(angle)$, $tz = -p[2]$. Add sphere to `erosionSpheres` vector. Create a `DifferenceNode` between the original SDF and each erosion sphere [S2, unofficial].
- **Parameters:** `radius` (circle radius for placement): 0.1; `erosionSphereRadius`: 0.05; `numSpheres`: 10 [S2, unofficial].
- **When to choose:** When visual erosion or roughening of an SDF surface is required without modifying the underlying SDF function itself [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(k)$, where $k$ is the number of iterations until convergence or max iterations. $k$ depends on the smoothness of the field and the ratio of the field value to the Lipschitz constant.
- **Memory Complexity:** $O(1)$ auxiliary space, as it only stores current position and field value.
- **Measured Performance:** Performance is highly dependent on the tightness of $\lambda$. A loose $\lambda$ leads to small steps and high query counts [S1]. For compactly supported primitives with large radius variations, global Lipschitz bounds can cause significant performance degradation due to unnecessarily small steps [S3].
- **Strengths:** Simple to implement; guarantees no surface penetration (robustness); works directly with implicit surfaces without pre-processing [S1, S3].
- **Limitations:** Inefficient for grazing rays; inefficient for scenes with large scale variations if a global $\lambda$ is used; requires a valid global Lipschitz bound [S3].
- **When to use:** Simple implicit surfaces, real-time shaders where simplicity and robustness are prioritized over maximum performance, or when local Lipschitz bounds are difficult to compute [S3].
- **When not to use:** Complex scenes with large scale variations (use [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md) or [Segment Tracing](1462-safe-raymarching-step-size.md) variants); density fields with complex kernel sums [S3].

## Background
The method originates from the use of a global Lipschitz constant for signed distance fields to define a robust ray-tracing algorithm, known as sphere tracing, introduced by Hart in 1996 [S3]. It belongs to the ray-marching family and is widely used for direct content creation in shaders [S3]. The theoretical basis relies on the Lipschitz condition, which bounds the rate of change of the function, ensuring that a step size derived from the function value and the constant $\lambda$ will not overshoot the root [S1][S4, unofficial].

## Sources
[S1] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
[S2] skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, https://github.com/skanderzoghlami/QT-Project-Surfaces-Implicites-Marching-Cube-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S4] Understanding segment tracing - the faster alternative to ..., https://www.reddit.com/r/GraphicsProgramming/comments/1jhcd6m/understanding_segment_tracing_the_faster/, fetched 2026-10-27 (unofficial)
