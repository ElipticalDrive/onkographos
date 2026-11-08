# Bounding Shape Optimization

Also called: Bounding Shape Culling

## Overview
Bounding Shape Optimization reduces the computational cost of distance-field ray marching by skipping density evaluations in empty space using pre-defined geometric boundaries [S1, unofficial]. Unlike standard acceleration structures that refine traversal within the volume, this method treats the volume as a sparse field defined by explicit shapes, performing classical volumetric rendering only when the ray intersects a defined boundary [S1, unofficial]. It sets itself apart from [Beam Optimization](932-beam-optimization.md) by operating on individual ray geometry rather than ray coherence, and from [Performance Optimizations for SDF Ray Marching](1629-performance-optimizations-for-sdf-ray-marching.md) by relying on explicit shape intersection rather than implicit field properties or step-size heuristics alone.

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), list of bounding shapes $\mathcal{B}$, volumetric density function $\rho(\mathbf{x})$.
**Outputs:** Final pixel color $C$.

**Data Structures:**
*   `BoundingShape`: A geometric primitive (e.g., Sphere, Box, Cylinder) capable of computing ray intersection distances.
*   `Sphere`: Defined by center $\mathbf{c}$ and radius $r$.

**Steps:**
1.  **Define Boundaries:** Identify regions where density is non-zero. Define bounding shapes $\mathcal{B}$ that enclose these regions [S1, unofficial]. The tighter the shape fits the non-zero density, the fewer "useless" steps are performed [S1, unofficial].
2.  **Ray-Shape Intersection:** For each ray, compute the intersection distance $t_{hit}$ to the nearest bounding shape in $\mathcal{B}$ using a surface rendering technique (e.g., Sphere Marching) [S1, unofficial].
    *   If no intersection is found ($t_{hit} = \infty$), the ray is in empty space. Return background color.
    *   If an intersection is found, the ray enters the volume at $t_{enter} = t_{hit}$.
3.  **Volumetric Rendering:** Perform classical volumetric rendering (ray marching) starting from $t_{enter}$ [S1, unofficial].
    *   Iterate along the ray direction $\mathbf{d}$, sampling density $\rho(\mathbf{x})$ and accumulating color/opacity.
    *   **Step Size Strategy:** The implementation described uses increasingly larger steps as the ray progresses through the medium, accepting imprecision for performance [S1, unofficial].
    *   Terminate when the ray exits the bounding shape or the accumulated opacity reaches a threshold.
4.  **Composite:** Combine the accumulated volume color with the background.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, bounding_shapes):
    # 1. Find nearest intersection with any bounding shape
    t_min = infinity
    for shape in bounding_shapes:
        t_hit = intersect_shape(ray_origin, ray_direction, shape)
        if 0 < t_hit < t_min:
            t_min = t_hit

    # If no intersection, ray is in empty space
    if t_min == infinity:
        return background_color

    # 2. Start volumetric rendering inside the bounding shape
    t = t_min
    accumulated_color = (0, 0, 0, 0)
    
    # 3. March through the volume
    # Note: Step size logic is implementation-specific; 
    # source suggests increasing step sizes [S1, unofficial]
    while t < t_min + shape.radius * 2: # Simplified exit condition
        pos = ray_origin + ray_direction * t
        density = evaluate_density(pos)
        
        if density > 0:
            # Classical volumetric rendering step
            color, alpha = sample_volume(pos, density)
            accumulated_color = composite(accumulated_color, color, alpha)
            
            if accumulated_color.a >= 1.0:
                break
        
        # [G] Standard ray marching step; source implies variable step size
        step_size = compute_step_size(t) 
        t += step_size

    return composite(accumulated_color, background_color)
```

**Parameters:**
*   **Bounding Shape Choice:** Sphere Marching is selected in the reference implementation [S1, unofficial].
*   **Step Size:** Not explicitly defined as a constant; the source notes "increasingly bigger steps" are used [S1, unofficial].

**Edge Cases:**
*   **Loose Bounds:** If bounding shapes are not tight to the density field, "useless" steps are performed, reducing performance gains [S1, unofficial].
*   **Deformation:** The source mentions deformation techniques (bending/stretching) for shapes like cubes or spheres, implying dynamic updates to bounding shapes may be required for animated volumes [S1, unofficial].

## Subtopics

### [Convex Polyhedron Bounding](8027-convex-polyhedron-bounding.md)
Uses convex polyhedra for tighter enclosure than axis-aligned boxes, reducing density evaluations for complex geometries.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of rays and $K$ is the number of steps *inside* the bounding shape. The cost of finding the intersection is $O(M)$ where $M$ is the number of bounding shapes.
*   **Memory Complexity:** $O(M)$ to store bounding shape parameters.
*   **Strengths:** Significant performance boost in scenes with mostly empty density fields by skipping the majority of the ray traversal [S1, unofficial]. Simple to implement using existing surface intersection routines.
*   **Limitations:** Requires explicit definition of non-zero density regions. Loose bounding shapes negate benefits. Does not handle complex internal structures within the volume efficiently if the volume is dense throughout the shape.
*   **When to Use:** Scenes with sparse volumetric data (e.g., dust clouds, solar flares) where large portions of the scene are empty [S1, unofficial].
*   **When Not to Use:** Dense volumes filling the entire scene or where defining tight bounding shapes is computationally prohibitive.

## Background
Originating from work on real-time rendering of interstellar dust clouds and solar flares, this method was developed to handle the hypothesis of "mostly empty density fields" [S1, unofficial]. The approach leverages the sparsity of the volume to optimize rendering performance by restricting expensive density evaluations to specific geometric regions [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
