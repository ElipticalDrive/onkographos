# Zeno Implementation

Also called: Zeno, Zeno Implementation for Implicit Surfaces

## Overview
The Zeno implementation is a specific software architecture for sphere tracing that minimizes the interface required to define new implicit surfaces [S1, S2]. It solves the problem of rapid prototyping for mathematical visualization by allowing new primitives to be added using only a distance bound function, while optionally bypassing explicit normal calculations via numerical gradients [S1, S2]. It is distinguished from general sphere tracing implementations by its specific emphasis on the sufficiency of the positive distance bound for visualization and its built-in six-sample gradient approximation [S1, S2].

## How it works
The Zeno implementation follows the standard sphere tracing loop defined in [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md), with specific constraints on the scene description and shading interfaces.

### Inputs and Outputs
*   **Input**: A camera ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the normalized direction.
*   **Input**: A scene defined by a signed distance bound function $f(\mathbf{p})$.
*   **Output**: The intersection distance $t_{hit}$ and the surface normal $\mathbf{n}$ at the hit point $\mathbf{p}_{hit} = \mathbf{o} + t_{hit}\mathbf{d}$.

### Data Structures
The implementation does not require complex scene graphs or acceleration structures for the primitives themselves, relying instead on the mathematical properties of the SDF [S1, S2]. The primary data structure is the function pointer or closure for $f(\mathbf{p})$.

### Steps
1.  **Ray Marching**:
    *   Initialize $t = t_{near}$ (typically 0 or a small positive value) [S1, S34, unofficial].
    *   Loop while $t < t_{max}$ and $f(\mathbf{o} + t\mathbf{d}) > \epsilon$:
        *   Evaluate $d = f(\mathbf{o} + t\mathbf{d})$ [S1, S34, unofficial].
        *   Advance $t \leftarrow t + d$ [S1, S33, unofficial].
    *   If $f(\mathbf{o} + t\mathbf{d}) \le \epsilon$, a hit is recorded at $t_{hit} = t$ [S1, S39, unofficial].

2.  **Normal Estimation**:
    *   The implementation allows the user to omit an explicit normal function. If omitted, the normal is computed using a six-sample numerical gradient approximation of the distance bound gradient [S1, S2].
    *   The gradient $\nabla f(\mathbf{p})$ is approximated as:
        $$ \nabla f(\mathbf{p}) \approx \left[ f(\mathbf{p} + \delta \mathbf{x}) - f(\mathbf{p} - \delta \mathbf{x}), \quad f(\mathbf{p} + \delta \mathbf{y}) - f(\mathbf{p} - \delta \mathbf{y}), \quad f(\mathbf{p} + \delta \mathbf{z}) - f(\mathbf{p} - \delta \mathbf{z}) \right] $$
        where $\mathbf{x}, \mathbf{y}, \mathbf{z}$ are the basis vectors and $\delta$ is a small step size [S1, S2].
    *   The normal is normalized: $\mathbf{n} = \frac{\nabla f(\mathbf{p}_{hit})}{\|\nabla f(\mathbf{p}_{hit})\|}$ [S2055, S4619].

3.  **Shading**:
    *   Standard lighting models are applied using $\mathbf{n}$ and the hit position [S1].

### Parameters
*   **$\epsilon$**: Convergence threshold. Not given in the sources, but typically $0.001$ [S1, S39, unofficial].
*   **$\delta$**: Step size for numerical gradient. Not given in the sources.
*   **$t_{max}$**: Maximum ray distance. Not given in the sources.

### Edge Cases and Pitfalls
*   **Negative Distance Bound**: The negative part of the signed distance bound is only necessary for Constructive Solid Geometry (CSG) and blending operations [S1, S2]. For simple visualization of functions that are zero-valued inside the surface, the negative part is not needed [S1, S2].
*   **Performance**: The six-sample numerical gradient approximation is inefficient compared to analytical normals, but has a negligible impact on overall rendering performance because the majority of time is spent on ray intersection [S1, S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zeno_sphere_trace(ray_origin, ray_direction, scene_sdf, t_near=0.0, t_max=100.0, epsilon=1e-3, delta=1e-4):
    """
    Implements the Zeno sphere tracing method.
    
    Args:
        ray_origin: 3D vector, camera origin o.
        ray_direction: 3D vector, normalized direction d.
        scene_sdf: Function p -> float, the signed distance bound f(p).
        t_near: Start distance.
        t_max: Maximum ray distance.
        epsilon: Convergence threshold.
        delta: Step size for numerical gradient.
        
    Returns:
        hit_point: 3D vector or None if no hit.
        normal: 3D vector or None if no hit.
    """
    t = t_near
    while t < t_max:
        p = ray_origin + t * ray_direction
        d = scene_sdf(p)
        
        if d < epsilon:
            # Hit detected
            hit_point = p
            # Compute normal using six-sample numerical gradient
            # Basis vectors
            x_axis = (1.0, 0.0, 0.0)
            y_axis = (0.0, 1.0, 0.0)
            z_axis = (0.0, 0.0, 1.0)
            
            # Approximate gradient components
            grad_x = scene_sdf(p + delta * x_axis) - scene_sdf(p - delta * x_axis)
            grad_y = scene_sdf(p + delta * y_axis) - scene_sdf(p - delta * y_axis)
            grad_z = scene_sdf(p + delta * z_axis) - scene_sdf(p - delta * z_axis)
            
            normal = normalize((grad_x, grad_y, grad_z))
            return hit_point, normal
        
        # Safe step
        t += d
        
    return None, None

def normalize(v):
    """Standard vector normalization."""
    length = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    if length == 0:
        return (0.0, 0.0, 0.0)
    return (v[0]/length, v[1]/length, v[2]/length)
```

## Operating characteristics
*   **Time Complexity**: Dominated by the number of SDF evaluations per ray. The numerical gradient adds a constant overhead of 6 extra SDF evaluations per hit, but this is negligible compared to the ray marching cost [S1, S2].
*   **Memory Complexity**: Low. No scene acceleration structures are required for the primitives themselves [S1, S2].
*   **Strengths**: Simplicity of implementation; ease of adding new primitives by defining only a distance bound [S1, S2].
*   **Limitations**: Numerical gradient approximation is less accurate than analytical normals [S1, S2]. The method assumes the SDF is a valid distance bound (Lipschitz constant 1) for the step size to be safe [S50, unofficial].
*   **When to use**: Visualization of mathematical tasks and investigation of new implicit surfaces, such as homotopies [S1, S2].
*   **When not to use**: Production rendering requiring high-performance normals or complex CSG operations where the negative distance bound is critical and analytical normals are preferred.

## Background
The Zeno implementation was developed as part of the Zeno rendering system [S1, S2]. It was used to visualize mathematical tasks, including a homotopy that removes a 720° twist from a ribbon without moving either end, which formed the basis for the animated short "Air on the Dirac Strings" [Sandin et al, 1993] [S1, S2]. This homotopy was based heavily on interpolated quaternion rotations and was easily incorporated into Zeno as a domain transformation [S2]. The implementation was described in the paper "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" [S1, S2].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
