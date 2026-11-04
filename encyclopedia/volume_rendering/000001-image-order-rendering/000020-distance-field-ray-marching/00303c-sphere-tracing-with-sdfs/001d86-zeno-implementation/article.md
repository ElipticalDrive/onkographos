# Zeno Implementation

Also called: Zeno, Zeno Implementation for Implicit Surfaces

## Overview
The Zeno implementation is a specific software architecture for sphere tracing that minimizes the coding effort required to add new implicit surfaces by relying on a numerical gradient approximation for shading [S1, S2]. It solves the problem of boilerplate code in ray tracers by allowing primitives to be defined solely by a signed distance function (SDF), automatically deriving normals via finite differences [S1, S2]. It differs from standard sphere tracing implementations by explicitly decoupling the intersection logic from the shading logic, using a fixed six-sample stencil for gradients rather than analytical derivatives [S1, S2].

## How it works
**Inputs and Outputs:**
- Input: A signed distance function $f(\mathbf{p})$ that returns the distance from point $\mathbf{p}$ to the implicit surface [S1, S2].
- Input: A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1, S2].
- Output: Intersection point $\mathbf{p}$ and surface normal $\mathbf{n}$ [S1, S2].

**Data Structures:**
- The implementation does not require explicit storage of surface normals or analytical derivative functions [S1, S2].
- The SDF $f(\mathbf{p})$ is the only required user-defined function for visualization [S1, S2].

**Numbered Steps:**

1. **Ray Marching:** Perform standard sphere tracing using the SDF $f(\mathbf{p})$ to find the intersection point $\mathbf{p}$ along the ray [S1, S2].
   - Initialize $t = 0$.
   - Loop:
     - Evaluate $d = f(\mathbf{o} + t\mathbf{d})$.
     - If $d < \epsilon$ (where $\epsilon$ is a small threshold), break and set $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
     - If $d > t_{max}$, break and return no intersection.
     - $t \leftarrow t + d$.

2. **Normal Approximation:** If shading is required, compute the surface normal $\mathbf{n}$ using a six-sample numerical gradient approximation of the SDF gradient [S1, S2].
   - Let $\mathbf{p}$ be the intersection point.
   - Let $h$ be a small step size (e.g., machine epsilon or a small constant relative to scene scale).
   - Compute the gradient $\nabla f(\mathbf{p})$ using central differences along the three axes:
     $$
     \nabla f(\mathbf{p}) \approx \begin{bmatrix}
     \frac{f(\mathbf{p} + h\mathbf{x}) - f(\mathbf{p} - h\mathbf{x})}{2h} \\
     \frac{f(\mathbf{p} + h\mathbf{y}) - f(\mathbf{p} - h\mathbf{y})}{2h} \\
     \frac{f(\mathbf{p} + h\mathbf{z}) - f(\mathbf{p} - h\mathbf{z})}{2h}
     \end{bmatrix}
     $$
     where $\mathbf{x} = (1,0,0)$, $\mathbf{y} = (0,1,0)$, and $\mathbf{z} = (0,0,1)$ [S1, S2].
   - Normalize the gradient to obtain the normal: $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [S1, S2].

3. **Shading:** Use $\mathbf{n}$ for standard lighting calculations [S1, S2].

**Parameters:**
- $h$: Step size for numerical gradient. Not given in the sources. [G] Typical values are small, e.g., $10^{-4}$ to $10^{-6}$, or derived from the SDF's scale.
- $\epsilon$: Threshold for intersection detection. Not given in the sources. [G] Typically a small value like $10^{-4}$ or relative to ray length.

**Edge Cases and Pitfalls:**
- **CSG and Blending:** The negative part of the signed distance bound is only necessary for constructive solid geometry (CSG) and blending operations [S1, S2]. For simple visualization of functions zero-valued inside the surface, the negative part is not needed [S1, S2].
- **Performance:** The numerical gradient approximation is inefficient compared to analytical derivatives, but its impact on overall rendering performance is negligible because most time is spent on ray intersection [S1, S2].
- **Precision:** The numerical gradient may suffer from precision issues if $h$ is too small (catastrophic cancellation) or too large (poor approximation). Not given in the sources. [G]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zeno_sphere_trace(ray_origin, ray_direction, sdf, max_steps=1000, max_dist=100.0, epsilon=1e-4):
    """
    Performs sphere tracing using the Zeno implementation's numerical gradient for normals.
    
    Args:
        ray_origin: (x, y, z) tuple or array
        ray_direction: (x, y, z) tuple or array, assumed normalized
        sdf: Function that takes (x, y, z) and returns signed distance
        max_steps: Maximum number of march steps
        max_dist: Maximum total distance to march
        epsilon: Threshold for intersection
        
    Returns:
        (hit_point, normal) if hit, else (None, None)
    """
    t = 0.0
    for _ in range(max_steps):
        if t > max_dist:
            return None, None
        
        p = (ray_origin[0] + t * ray_direction[0],
             ray_origin[1] + t * ray_direction[1],
             ray_origin[2] + t * ray_direction[2])
        
        d = sdf(p[0], p[1], p[2])
        
        if d < epsilon:
            # Intersection found, compute normal
            normal = compute_numerical_gradient(p, sdf)
            return p, normal
        
        t += d
    
    return None, None

def compute_numerical_gradient(p, sdf, h=1e-4):
    """
    Computes the gradient of the SDF at point p using a six-sample central difference.
    
    Args:
        p: (x, y, z) tuple or array
        sdf: Function that takes (x, y, z) and returns signed distance
        h: Step size for finite differences
        
    Returns:
        Normalized gradient vector (nx, ny, nz)
    """
    x, y, z = p
    
    # Central differences
    dx = (sdf(x + h, y, z) - sdf(x - h, y, z)) / (2 * h)
    dy = (sdf(x, y + h, z) - sdf(x, y - h, z)) / (2 * h)
    dz = (sdf(x, y, z + h) - sdf(x, y, z - h)) / (2 * h)
    
    # Normalize
    length = (dx*dx + dy*dy + dz*dz) ** 0.5
    if length < 1e-10:
        return (0.0, 0.0, 1.0) # Fallback, though degenerate
    
    return (dx / length, dy / length, dz / length)
```

## Subtopics

### [Raymarched SDF Volume Rendering](3663-raymarched-sdf-volume-rendering.md)
Renders volumetric data using SDFs, focusing on density accumulation rather than surface intersection. Choose when rendering fog, clouds, or medical volumes.

### [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md)
Uses a fixed step size for ray marching, simpler but less efficient for complex surfaces. Choose when performance is not critical or surfaces are uniformly simple.

### [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md)
Specific implementation for fractal geometry, often using recursive distance bounds. Choose when rendering fractals like the Menger sponge.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of ray march steps. The numerical gradient adds a constant factor of 6 SDF evaluations per intersection, but this is negligible compared to the cost of ray intersection [S1, S2].
- **Memory Complexity:** $O(1)$ auxiliary memory, as no explicit normal storage or analytical derivative functions are required [S1, S2].
- **Measured Performance:** Runs at speeds comparable to other implicit surface rendering algorithms [S1, S2].
- **Strengths:** Simplicity of implementation; easy to add new primitives by defining only an SDF [S1, S2].
- **Limitations:** Numerical gradient is less accurate than analytical derivatives; not suitable for applications requiring high-precision normals (e.g., precise shadow mapping) [S1, S2].
- **When to Use:** Visualization of mathematical tasks and investigation of new implicit surfaces [S1, S2].
- **When Not to Use:** When analytical normals are required for high-quality shading or when performance is critical and SDF evaluation is expensive [S1, S2].

## Background
The Zeno implementation was developed as part of the Zeno rendering system, which implemented sphere tracing for antialiased ray tracing of implicit surfaces [S1, S2]. It was used for the animated short "Air on the Dirac Strings" (1993), which visualized a homotopy removing a 720° twist from a ribbon using interpolated quaternion rotations [S1, S2]. The system was designed to facilitate the visualization of mathematical tasks and the investigation of new implicit surfaces [S1, S2].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
