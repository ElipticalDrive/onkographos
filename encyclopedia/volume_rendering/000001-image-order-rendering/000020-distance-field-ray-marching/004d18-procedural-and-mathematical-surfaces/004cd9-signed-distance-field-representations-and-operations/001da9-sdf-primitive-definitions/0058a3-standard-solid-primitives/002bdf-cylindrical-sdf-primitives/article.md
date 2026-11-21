# Cylindrical SDF Primitives

Also called: Cap Cylinder SDF, Ray Marching SDF Geometry

## Overview
Cylindrical SDF Primitives define the signed distance to a cylinder (often capped) using analytic geometry, enabling efficient ray marching for solid objects or volumetric light shaping. Unlike generic mesh-based distance fields, these primitives provide exact, differentiable distance values with minimal computational overhead, allowing for infinite resolution at any scale. They are distinct from [Distance to Natural Quadrics](1559-distance-to-natural-quadrics.md) by their specific cylindrical topology and from [Standard Solid Primitives](16223-standard-solid-primitives.md) by their reliance on axis-aligned or arbitrary-axis cylinder definitions rather than bounding boxes or spheres.

## How it works
The method computes the Euclidean distance from a query point $p$ to the surface of a cylinder defined by an origin, an axis direction, a radius, and optionally a height (for capped cylinders).

### Inputs and Outputs
*   **Input**: Query point $p \in \mathbb{R}^3$.
*   **Input**: Cylinder parameters:
    *   $c$: Axis origin point.
    *   $a$: Unit axis direction vector ($|a|=1$).
    *   $r$: Radius of the cylinder.
    *   $h$: Half-height of the cylinder (optional, for capped cylinders).
*   **Output**: Scalar signed distance $d$. $d < 0$ inside, $d = 0$ on surface, $d > 0$ outside.

### Data Structures
No complex data structures are required. The primitive is stateless and evaluates analytically. Parameters are stored as floats/vectors.

### Steps and Formulas

#### 1. Uncapped Cylinder SDF
For an infinite cylinder, the distance is the distance to the axis minus the radius.

1.  Compute the vector from the axis origin to the query point:
    $$ v = p - c $$
2.  Project $v$ onto the axis direction to find the closest point on the infinite axis:
    $$ t = \text{dot}(v, a) $$
    $$ p_{\text{axis}} = c + a \cdot t $$
3.  Compute the distance from the query point to the axis:
    $$ d_{\text{axis}} = \text{length}(p - p_{\text{axis}}) $$
4.  Return the signed distance:
    $$ d = d_{\text{axis}} - r $$

#### 2. Capped Cylinder SDF
For a finite cylinder with half-height $h$, the distance must account for the end caps. The standard approach uses a 2D distance function in the plane containing the axis and the query point.

1.  Transform the query point into the cylinder's local coordinate system where the axis aligns with the Y-axis (or Z-axis, depending on convention). Let the local coordinates be $(x, y, z)$. If the global axis is $a$, we can compute the distance to the axis and the height along the axis.
    *   Distance to axis: $d_{\text{axis}} = \text{length}(p - (c + a \cdot \text{dot}(p-c, a)))$
    *   Height along axis: $h_{\text{local}} = \text{dot}(p - c, a)$
2.  Define the 2D distance to the rectangle defined by the radius $r$ and half-height $h$. The "cap cylinder" function often computes the distance to the outer shell and inner void if creating a hollow cylinder, or simply the distance to the capped surface.
    
    The general formula for a capped cylinder (solid) is:
    $$ q = \begin{bmatrix} d_{\text{axis}} - r \\ |h_{\text{local}}| - h \end{bmatrix} $$
    $$ d = \min(\max(q.x, q.y), 0.0) + \text{length}(\max(q, 0.0)) $$
    
    Where:
    *   $q.x$ is the radial distance from the surface.
    *   $q.y$ is the vertical distance from the caps.
    *   $\max(q, 0.0)$ is the component-wise maximum.
    *   If $q.x < 0$ and $q.y < 0$, the point is inside, and $d = \min(q.x, q.y)$.
    *   If one is positive, $d$ is the distance to the nearest edge/corner.

    *Note: Source [S1, unofficial] provides a specific implementation for a "capCylinder" function used in a hollow cylinder context:*
    $$ d = \text{abs}(\text{vec2}(\text{length}(p.yz), p.x)) - \text{vec2}(r, h) $$
    $$ d_{\text{final}} = \min(\max(d.x, d.y), 0.) + \text{length}(\max(d, \text{vec2}(0.))) $$
    This assumes the cylinder is aligned with the X-axis, with radius $r$ and half-height $h$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sd_cylinder_uncapped(p, axis_origin, axis_dir, radius):
    """
    Computes SDF for an infinite cylinder.
    p: vec3, query point
    axis_origin: vec3, point on the cylinder axis
    axis_dir: vec3, unit vector along the cylinder axis
    radius: float, radius of the cylinder
    """
    # Vector from axis origin to query point
    p_to_origin = p - axis_origin
    
    # Projection length onto the axis
    projection_length = dot(p_to_origin, axis_dir)
    
    # Closest point on the axis
    closest_point_on_axis = axis_origin + axis_dir * projection_length
    
    # Distance from point to the axis
    distance_to_axis = length(p - closest_point_on_axis)
    
    # Signed distance
    return distance_to_axis - radius

def sd_cylinder_capped(p, axis_origin, axis_dir, radius, half_height):
    """
    Computes SDF for a capped cylinder.
    p: vec3, query point
    axis_origin: vec3, center of the cylinder base (or midpoint depending on convention)
    axis_dir: vec3, unit vector along the cylinder axis
    radius: float, radius of the cylinder
    half_height: float, half the total height of the cylinder
    """
    # Transform p to local space aligned with axis
    # We need the distance to the axis and the height along the axis
    
    p_to_origin = p - axis_origin
    
    # Height along the axis
    height_along_axis = dot(p_to_origin, axis_dir)
    
    # Closest point on the infinite axis
    closest_point_on_axis = axis_origin + axis_dir * height_along_axis
    
    # Distance to the axis
    dist_to_axis = length(p - closest_point_on_axis)
    
    # 2D distance to the rectangle (radius, half_height)
    # q.x is radial distance from surface
    # q.y is vertical distance from caps
    q_x = dist_to_axis - radius
    q_y = abs(height_along_axis) - half_height
    
    # Calculate distance to the rounded rectangle
    d = min(max(q_x, q_y), 0.0) + length(max(vec2(q_x, q_y), 0.0))
    
    return d

def compute_normal_cylinder(p, axis_origin, axis_dir, radius, epsilon=0.0001):
    """
    Approximates the normal using central finite differences.
    """
    # Gradient approximation
    dx = sd_cylinder_capped(p + vec3(epsilon, 0, 0), axis_origin, axis_dir, radius, 1.0) - sd_cylinder_capped(p, axis_origin, axis_dir, radius, 1.0)
    dy = sd_cylinder_capped(p + vec3(0, epsilon, 0), axis_origin, axis_dir, radius, 1.0) - sd_cylinder_capped(p, axis_origin, axis_dir, radius, 1.0)
    dz = sd_cylinder_capped(p + vec3(0, 0, epsilon), axis_origin, axis_dir, radius, 1.0) - sd_cylinder_capped(p, axis_origin, axis_dir, radius, 1.0)
    
    return normalize(vec3(dx, dy, dz))
```

### Parameters and Typical Values
*   **Radius ($r$)**: Depends on scene scale. Example: $0.4255$ [S1, unofficial].
*   **Half-Height ($h$)**: Depends on scene scale. Example: $0.5$ [S1, unofficial].
*   **Epsilon ($\epsilon$)**: For normal calculation, typically $0.0001$ [Parent]. Source [S1, unofficial] uses $0.001$.
*   **Smooth Edge Width**: When using SDFs for volumetric light shaping, a smooth transition is often applied. Example: $0.1$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Axis Normalization**: The axis direction vector must be normalized before use in dot products to ensure correct projection lengths.
*   **Non-Uniform Scaling**: If the cylinder is scaled non-uniformly, the SDF is no longer exact. Distance compensation is required [Parent].
*   **Precision**: For very large or very small cylinders, floating-point precision may affect the accuracy of the distance calculation.
*   **Hollow Cylinders**: To create a hollow cylinder (like a pipe), the SDF can be computed as the difference between an outer cylinder and an inner cylinder: $d = \max(-d_{\text{inner}}, d_{\text{outer}})$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per evaluation. The calculation involves a constant number of arithmetic operations.
*   **Memory Complexity**: $O(1)$. No auxiliary data structures are needed.
*   **Performance**: Extremely fast, suitable for real-time ray marching. Source [S2, unofficial] uses it for volumetric light shaping in post-processing, indicating low overhead.
*   **Strengths**: Exact distance, infinite resolution, easy to transform, efficient for ray marching.
*   **Limitations**: Only applies to cylindrical geometries. Complex shapes require CSG operations or blending, which can introduce artifacts.

## Background
Cylindrical SDFs are a fundamental building block in procedural geometry and ray marching. They are widely used in demoscene productions, real-time rendering, and volumetric lighting effects. Source [S2, unofficial] highlights their use in shaping volumetric light in post-processing, referencing techniques popularized by artists like Kenny Pirman. Source [S1, unofficial] demonstrates their use in WebGPU-based voxel cone tracing for defining solid geometry.

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] On Shaping Light: Real-Time Volumetric Lighting with Post-Processing ..., https://blog.maximeheckel.com/posts/shaping-light-volumetric-lighting-with-post-processing-and-raymarching/, fetched 2026-10-28 (unofficial)
