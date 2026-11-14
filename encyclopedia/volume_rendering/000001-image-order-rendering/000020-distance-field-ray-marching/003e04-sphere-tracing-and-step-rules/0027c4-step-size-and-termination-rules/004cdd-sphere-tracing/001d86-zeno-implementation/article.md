# Zeno Implementation

Also called: Zeno, Zeno Implementation for Implicit Surfaces

## Overview
The Zeno implementation is a specific software architecture for sphere tracing that prioritizes ease of extension for new implicit surfaces over raw shading performance [S1, S2]. It solves the problem of rapidly prototyping mathematical implicit surfaces by allowing new primitives to be added by defining only a signed distance bound, while avoiding the manual derivation of surface normals [S1, S2]. It sets itself apart from standard sphere tracing implementations by replacing the explicit normal function with a fixed six-sample numerical gradient approximation, accepting the minor performance cost as negligible compared to ray intersection overhead [S1, S2].

## How it works
The method implements sphere tracing within the Zeno rendering system. It requires the user to provide a signed distance function (SDF) for ray intersection. Shading is handled via a numerical gradient approximation rather than an analytical normal function.

### Inputs and Outputs
*   **Input:** A signed distance bound function $d(\mathbf{p})$ that returns the distance from a point $\mathbf{p}$ to the implicit surface.
*   **Output:** A rendered image of the implicit surface.

### Data Structures
No specific complex data structures are required beyond standard ray and point representations. The core data is the distance bound function $d(\mathbf{p})$.

### Steps
1.  **Define Distance Bound:** Implement the function $d(\mathbf{p})$ for the implicit surface. This function must return the signed distance from any point $\mathbf{p}$ to the surface.
2.  **Ray Marching:** Perform sphere tracing along the ray. At each step, evaluate $d(\mathbf{p})$ to determine the step size.
    *   If $d(\mathbf{p}) > \epsilon$ (where $\epsilon$ is a small threshold), advance the ray position $\mathbf{p}$ by $d(\mathbf{p})$.
    *   If $d(\mathbf{p}) \le \epsilon$, the surface has been hit.
3.  **Gradient Approximation:** To compute the surface normal $\mathbf{n}$ at the hit point $\mathbf{p}$, use a six-sample numerical gradient approximation of the distance bound gradient $\nabla d(\mathbf{p})$ [S1, S2].
    *   The normal is calculated as:
        $$ \mathbf{n} = \frac{\nabla d(\mathbf{p})}{\|\nabla d(\mathbf{p})\|} $$
    *   The gradient $\nabla d(\mathbf{p})$ is approximated using finite differences across six samples (typically along the coordinate axes and potentially their negatives, though the exact stencil is not specified in the sources) [S1, S2].
4.  **Shading:** Use the computed normal $\mathbf{n}$ for standard shading calculations.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zeno_sphere_trace(ray_origin, ray_direction, distance_bound_func, epsilon=1e-4):
    """
    Implements sphere tracing for the Zeno system.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray (normalized).
        distance_bound_func: Function p -> float, returns signed distance bound.
        epsilon: Float, threshold for surface hit detection.
        
    Returns:
        hit_point: 3D vector, point of intersection, or None if no hit.
        normal: 3D vector, surface normal at hit point, or None if no hit.
    """
    t = 0.0
    max_steps = 1000  # Not given in sources, typical safeguard
    p = ray_origin + t * ray_direction
    
    for _ in range(max_steps):
        d = distance_bound_func(p)
        
        # Check for surface hit
        if d <= epsilon:
            # Compute normal using six-sample numerical gradient
            normal = compute_numerical_gradient_normal(distance_bound_func, p)
            return p, normal
        
        # Advance ray
        t += d
        p = ray_origin + t * ray_direction
        
        # Check for ray termination (e.g., max distance)
        # Not given in sources, implementation dependent
        
    return None, None

def compute_numerical_gradient_normal(distance_bound_func, p, h=1e-4):
    """
    Computes the surface normal using a six-sample numerical gradient approximation.
    
    Args:
        distance_bound_func: Function p -> float.
        p: 3D vector, point at which to compute gradient.
        h: Float, step size for finite differences.
        
    Returns:
        normal: 3D vector, normalized gradient.
    """
    # Approximate gradient using central differences
    # Standard central difference uses 2 samples per axis (6 total for 3D)
    # Gradient components:
    grad_x = (distance_bound_func(p + vec(h, 0, 0)) - distance_bound_func(p - vec(h, 0, 0))) / (2 * h)
    grad_y = (distance_bound_func(p + vec(0, h, 0)) - distance_bound_func(p - vec(0, h, 0))) / (2 * h)
    grad_z = (distance_bound_func(p + vec(0, 0, h)) - distance_bound_func(p - vec(0, 0, h))) / (2 * h)
    
    gradient = vec(grad_x, grad_y, grad_z)
    
    # Normalize
    length = sqrt(grad_x**2 + grad_y**2 + grad_z**2)
    if length == 0:
        return vec(0, 0, 1) # Fallback, not given in sources
    return gradient / length

def vec(x, y, z):
    return (x, y, z)

def sqrt(x):
    return x ** 0.5
```

### Parameters
*   **Epsilon:** Threshold for surface hit detection. Not given in sources.
*   **Gradient Step Size (h):** Step size for numerical gradient. Not given in sources.

### Edge Cases and Pitfalls
*   **CSG and Blending:** The negative part of the signed distance bound is only necessary for constructive solid geometry (CSG) and blending operations. It is not needed for visualizing functions that are zero-valued inside the implicit surface [S1, S2].
*   **Performance:** The numerical gradient approximation is less efficient than an analytical normal function, but its impact on rendering performance is negligible because most time is spent on ray intersection [S1, S2].

## Operating characteristics
*   **Time Complexity:** Dominated by ray intersection steps. The numerical gradient adds a constant overhead of 6 distance function evaluations per shading point.
*   **Memory Complexity:** Low, as no complex data structures are required.
*   **Strengths:** Simplicity of implementation for new implicit surfaces; ease of incorporating new primitives by defining only a distance bound.
*   **Limitations:** Slightly slower shading due to numerical gradient; less precise normals compared to analytical methods.
*   **When to Use:** Visualization of mathematical tasks and investigation of new implicit surfaces where rapid prototyping is more important than maximum rendering speed [S1, S2].
*   **When Not to Use:** Production rendering where performance is critical and analytical normals are available.

## Background
The Zeno implementation was developed as part of the Zeno rendering system, which implemented sphere tracing for antialiased ray tracing of implicit surfaces [S1, S2]. It was used for visualization of mathematical tasks, such as the animated short "Air on the Dirac Strings" (Sandin et al., 1993), which involved a homotopy removing a 720° twist from a ribbon using interpolated quaternion rotations [S1, S2]. The system demonstrated that sphere tracing simplifies the implementation of implicit surface ray tracers while running at speeds comparable to other implicit surface rendering algorithms [S1, S2].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
