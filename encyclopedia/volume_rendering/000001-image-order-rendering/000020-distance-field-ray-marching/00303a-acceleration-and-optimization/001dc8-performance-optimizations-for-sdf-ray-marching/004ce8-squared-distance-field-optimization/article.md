# Squared Distance Field Optimization

Also called: SDF Squared Distance Trick, Squared Distance Optimization

## Overview
Squared Distance Field Optimization is a technique to avoid the computational cost of square root operations in Signed Distance Field (SDF) raymarching by using squared distances for sphere comparisons, leveraging the monotonicity of the squaring function [S1, unofficial]. It sets itself apart from standard SDF ray marching by replacing Euclidean distance metrics with squared Euclidean metrics for specific primitives (typically spheres) to eliminate `sqrt` calls, which are often expensive on GPU architectures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $O$ and direction $D$, and a scene containing at least one sphere (defined by center $C$ and radius $r$) and other objects with known distance functions.
*   **Output:** The minimum distance to the closest surface along the ray, or the intersection point, computed without evaluating square roots for the spherical components.

**Data Structures:**
No specific data structure changes are required beyond the standard representation of spheres. The optimization is purely algebraic within the distance query function.

**Steps:**
1.  **Compute Squared Sphere Distance:** Calculate the squared distance from the query point $P$ to the surface of the sphere. This is done by computing the squared Euclidean distance from $P$ to the sphere center $C$, and subtracting the squared radius $r^2$ [S1, unofficial].
    $$ d_{sphere}^2 = \|P - C\|^2 - r^2 $$
    Where $\|P - C\|^2 = (P_x - C_x)^2 + (P_y - C_y)^2 + (P_z - C_z)^2$ [S1, unofficial].

2.  **Square Other Distances:** For any other object in the scene that provides a standard Euclidean distance $d_{other}$, square this value to convert it to a squared distance metric: $d_{other}^2$ [S1, unofficial].

3.  **Minimize Squared Distances:** Determine the closest surface by taking the minimum of the squared sphere distance and the squared distances of all other objects [S1, unofficial].
    $$ d_{min}^2 = \min(d_{sphere}^2, d_{other1}^2, d_{other2}^2, \dots) $$
    This yields the same ordering as minimizing the unsquared distances because the squaring function $f(x) = x^2$ is monotonically increasing for $x \ge 0$ [S1, unofficial].

4.  **Propagate Squared Metric:** Continue using squared distances for subsequent comparisons and logic within the ray marching loop to avoid taking square roots as far as possible [S1, unofficial]. If the final result requires a Euclidean distance (e.g., for shading or step size), take the square root only at the very end [S1, unofficial].

**Pseudocode:**
```python
def squared_distance_to_sphere(point, center, radius):
    """
    Computes the squared signed distance from point to sphere surface.
    Returns positive if outside, negative if inside.
    """
    diff = point - center
    squared_dist_to_center = dot(diff, diff)
    squared_radius = radius * radius
    return squared_dist_to_center - squared_radius

def ray_march_step_squared(ray_origin, ray_dir, scene_objects):
    """
    Performs one step of ray marching using squared distance optimization.
    Note: This is a simplified conceptual step. Standard ray marching 
    requires solving for t along the ray, not just point evaluation.
    The optimization applies to the distance field evaluation function.
    """
    # Initialize minimum squared distance to infinity
    min_sq_dist = float('inf')
    
    for obj in scene_objects:
        if obj.type == 'sphere':
            # Use squared distance formula
            sq_dist = squared_distance_to_sphere(ray_origin, obj.center, obj.radius)
        else:
            # Get standard Euclidean distance from other SDFs
            euclidean_dist = obj.sdf(ray_origin)
            sq_dist = euclidean_dist * euclidean_dist
            
        if sq_dist < min_sq_dist:
            min_sq_dist = sq_dist
            
    # Return squared distance. 
    # If step size is needed, sqrt(min_sq_dist) must be taken here.
    return min_sq_dist
```

**Parameters:**
*   `sphere_radius`: The radius of the sphere. Used as `radius * radius` in the calculation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Ray Marching Step Size:** Using the squared distance directly as the ray march step size (`currDst`) is incorrect. The distance moved along the ray is not equal to the squared distance unless the ray hits the center of the sphere exactly [S1, unofficial]. The square root must be taken to get the actual Euclidean distance for stepping [S1, unofficial].
*   **Intersection Only:** If the goal is purely intersection testing (boolean hit/miss) and not distance-based shading or marching, direct ray-sphere intersection formulas are preferred over distance fields entirely [S1, unofficial].
*   **Precision:** Squaring large distances can lead to floating-point precision issues or overflow. This method is most beneficial when distances are moderate and `sqrt` latency is the bottleneck [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of `sqrt` operations from $O(N)$ per ray step (where $N$ is the number of primitives) to potentially $O(1)$ or $O(K)$ where $K$ is the number of non-spherical primitives. This improves performance on hardware where `sqrt` is expensive relative to multiplication [S1, unofficial].
*   **Memory Complexity:** No additional memory overhead.
*   **Strengths:** Significant performance gain in scenarios with many spherical primitives where `sqrt` is a bottleneck.
*   **Limitations:** Cannot be used directly for step size calculation; requires careful handling of mixed distance metrics (squared vs. Euclidean) in the same scene. Not suitable if the actual Euclidean distance is needed at every step for logic other than step size.

## Background
This technique was discussed in the context of CUDA programming and ray tracing optimization on Stack Overflow in 2019 [S1, unofficial]. It leverages the mathematical property that minimizing $x^2$ is equivalent to minimizing $x$ for non-negative $x$, allowing developers to avoid costly square root instructions in GPU shaders [S1, unofficial].

## Sources
[S1] The most effective way of computing distance to sphere in CUDA?, https://stackoverflow.com/questions/55930232/the-most-effective-way-of-computing-distance-to-sphere-in-cuda, fetched 2026-10-27 (unofficial)
