# Zeno Implementation

Also called: Zeno, Zeno Implementation for Implicit Surfaces

## Overview
The Zeno Implementation is a sphere tracing method designed for rapid prototyping of implicit surfaces [S1, S2]. It simplifies geometry definition by requiring only a signed distance bound function, while optionally replacing explicit surface normal functions with a numerical gradient approximation [S1, S2].

## How it works
The method follows standard sphere tracing principles [G], with Zeno-specific optimizations for primitive inclusion and shading [S1, S2].

### Inputs and Outputs
- **Inputs**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), a distance bound function $f(\mathbf{p})$, and rendering parameters ($\text{MAX\_STEPS}$, $\text{HIT\_THRESHOLD}$, $\text{MAX\_RANGE}$) [G].
- **Outputs**: Intersection point $\mathbf{p}$, hit status (boolean), and surface normal $\mathbf{n}$ [G].

### Data Structures
No complex scene graphs are required. The scene is defined by the composition of distance bound functions [S1, S2].
- **Distance Bound Function**: A function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$ that returns the distance to the nearest surface [G].
- **Gradient Approximation**: Requires access to $f(\mathbf{p})$ at 6 specific offsets [S1, S2].

### Steps
1. **Initialize Ray**: Set accumulated distance $t = 0$.
2. **Sphere Trace Loop**:
   a. Evaluate distance bound: $dist = f(\mathbf{o} + t\mathbf{d})$.
   b. **Hit Check**: If $dist \le \text{HIT\_THRESHOLD}$, compute the exact hit point $\mathbf{p} = \mathbf{o} + (t + dist)\mathbf{d}$, compute the normal, and terminate [G].
   c. **Advance**: $t \leftarrow t + dist$.
   d. **Range Check**: If $t > \text{MAX\_RANGE}$, terminate (miss) [G].
   e. **Step Count**: If iteration count $> \text{MAX\_STEPS}$, terminate (miss) [G].
3. **Shading (Normal Computation)**:
   If a hit is detected, compute the surface normal $\mathbf{n}$ using a six-sample numerical gradient approximation of the distance bound gradient [S1, S2]. The gradient $\nabla f(\mathbf{p})$ is approximated by evaluating $f(\mathbf{p})$ at $\mathbf{p} \pm \epsilon \hat{x}$, $\mathbf{p} \pm \epsilon \hat{y}$, and $\mathbf{p} \pm \epsilon \hat{z}$, where $\hat{x}, \hat{y}, \hat{z}$ are the basis vectors and $\epsilon$ is a small step size [S1, S2]. The normal is the normalized gradient:
   $$ \mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|} $$
   This approach avoids the need for an explicit analytic normal function [S1, S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zeno_sphere_trace(o, d, distance_bound, max_steps=40, hit_threshold=0.001, max_range=100.0, grad_epsilon=0.001):
    t = 0.0
    for i in range(max_steps):
        p = o + d * t
        dist = distance_bound(p)
        
        # Hit check
        if dist <= hit_threshold:
            hit_p = p + d * dist
            normal = compute_numerical_normal(distance_bound, hit_p, grad_epsilon)
            return hit_p, True, normal
            
        # Advance
        t += dist
        
        # Range check
        if t > max_range:
            return o + d * t, False, None
            
    return o + d * t, False, None

def compute_numerical_normal(distance_bound, p, epsilon):
    # Six-sample gradient approximation
    dx = (distance_bound(p + vec3(epsilon, 0, 0)) - distance_bound(p - vec3(epsilon, 0, 0))) / (2 * epsilon)
    dy = (distance_bound(p + vec3(0, epsilon, 0)) - distance_bound(p - vec3(0, epsilon, 0))) / (2 * epsilon)
    dz = (distance_bound(p + vec3(0, 0, epsilon)) - distance_bound(p - vec3(0, 0, epsilon))) / (2 * epsilon)
    
    grad = vec3(dx, dy, dz)
    return normalize(grad)
```

### Parameters
- **$\text{HIT\_THRESHOLD}$**: Small epsilon for surface detection. Typical value: 0.001 [S1, S2].
- **$\text{MAX\_STEPS}$**: Safety cap for iteration count. Typical value: 40 [G].
- **$\text{MAX\_RANGE}$**: Maximum distance from ray origin. Typical value: 100.0 [G].
- **$\text{GRAD\_EPSILON}$**: Step size for numerical gradient. Typical value: 0.001 [G].

### Edge Cases and Pitfalls
- **Penetration**: The implementation explicitly handles cases where $dist < 0$, which occurs if the ray starts inside the surface or overshoots due to precision issues. The negative part of the distance bound is only necessary for CSG and blending operations, not for simple visualization of zero-valued functions [S1, S2].
- **Performance**: The numerical gradient approximation is computationally inefficient compared to analytic normals. However, the sources state that since most time is spent on ray intersection, the impact on overall rendering performance is negligible [S1, S2].

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the average number of sphere tracing steps. The shading step adds a constant overhead of 6 distance evaluations per hit [G].
- **Memory Complexity**: $O(1)$ auxiliary memory per ray, as no scene graph or acceleration structures are required [G].
- **Strengths**: Extremely simple to implement for new implicit surfaces; only a distance bound function is required. Ideal for mathematical visualization and prototyping [S1, S2].
- **Limitations**: Numerical gradient approximation is less accurate than analytic normals and slightly slower per hit. The system supports explicit normal functions for shading when higher precision is needed [S1, S2].

## Background
The Zeno Implementation was developed as part of the Zeno rendering system, described in the paper "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" by John Hart, et al. [S1, S2]. It was used for the animated short "Air on the Dirac Strings" (Sandin et al., 1993), which visualized a homotopy removing a 720° twist from a ribbon using interpolated quaternion rotations [S1, S2]. The system was designed to facilitate the investigation of new implicit surfaces and mathematical tasks by simplifying the inclusion of new primitives [S1, S2].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
