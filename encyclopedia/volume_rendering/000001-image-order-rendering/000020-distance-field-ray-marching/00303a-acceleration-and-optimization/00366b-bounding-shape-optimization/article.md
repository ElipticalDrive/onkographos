# Bounding Shape Optimization

Also called: Bounding Shape Culling

## Overview
Bounding Shape Optimization reduces redundant density evaluations in volumetric rendering by skipping steps in empty space using pre-defined geometric boundaries [S1, unofficial][S2, unofficial]. Unlike [Beam Optimization](932-beam-optimization.md) which restricts rays to narrow paths, or [Block Culling](5431-block-culling.md) which uses grid-based structures, this method relies on explicit closed geometry (e.g., spheres, boxes) to define the volume extent [S2, unofficial]. It is most effective for scenes with mostly empty density fields where the non-zero density is confined to specific, well-defined regions [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, bounding shape definition (e.g., sphere center $\mathbf{c}$ and radius $r$), and the volumetric density field function $\rho(\mathbf{x})$.
**Outputs:** Final pixel color and opacity.

**Data Structures:**
The bounding shape is defined by its implicit equation. For a sphere, the parameters are the center $\mathbf{c} \in \mathbb{R}^3$ and radius $r \in \mathbb{R}$.

**Steps:**
1.  **Define Boundaries:** Establish the geometric shape(s) where density is non-zero. The closer the boundary is to the actual start of the density field, the fewer "useless" steps are performed [S1, unofficial].
2.  **Ray-Shape Intersection:** Use a surface rendering technique to find the intersection points of the ray with the bounding shape. This determines the entry and exit points of the ray within the volume [S1, unofficial].
    *   For a sphere, solve for $t$ in $||\mathbf{o} + t\mathbf{d} - \mathbf{c}||^2 = r^2$.
3.  **Skip Empty Space:** If the ray does not intersect the bounding shape, skip the volume rendering entirely for that ray [S1, unofficial].
4.  **Volumetric Rendering:** Perform classical volumetric rendering only for the segment of the ray inside the intersected bounding shape [S1, unofficial].
    *   During this phase, step sizes may increase as the ray progresses through the medium to account for reduced impact of imprecisions, providing a performance boost [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_with_bounding_shape(ray_origin, ray_direction, sphere_center, sphere_radius, density_field):
    # Step 2: Ray-Sphere Intersection
    oc = ray_origin - sphere_center
    a = dot(ray_direction, ray_direction)
    b = 2.0 * dot(oc, ray_direction)
    c = dot(oc, oc) - sphere_radius * sphere_radius
    discriminant = b * b - 4 * a * c

    # If no intersection, skip rendering
    if discriminant < 0:
        return background_color

    sqrt_disc = sqrt(discriminant)
    t1 = (-b - sqrt_disc) / (2 * a)
    t2 = (-b + sqrt_disc) / (2 * a)

    # Ensure we start from the nearest intersection point
    t_start = max(0.0, t1)
    t_end = t2

    # If the ray starts inside and exits immediately, or no valid segment
    if t_start >= t_end:
        return background_color

    # Step 4: Volumetric Rendering inside bounds
    color = vec3(0.0)
    opacity = 0.0
    t = t_start
    step_size = initial_step_size # [G] Initialized to a small positive value

    while t < t_end and opacity < 1.0:
        pos = ray_origin + t * ray_direction
        density = density_field(pos)

        if density > 0:
            # Classical volume integration step
            alpha = 1.0 - exp(-density * step_size)
            color += alpha * density_field_color(pos) * (1.0 - opacity)
            opacity += alpha * (1.0 - opacity) # [G] Standard front-to-back compositing approximation
        
        # Adaptive step size: increase as we go through the medium [S1, unofficial]
        step_size = adaptive_step_size(t, t_end) 
        
        t += step_size

    return color
```

**Parameters:**
*   **Bounding Shape Choice:** The specific geometry used. Sphere Marching is selected in the reference implementation for its simplicity and efficiency in intersection calculation [S1, unofficial].
*   **Container Definition Fineness:** Finer container definitions (tighter bounds) improve volume render efficiency by reducing the number of samples taken in empty space [S2, unofficial].

**Edge Cases:**
*   **Ray Origin Inside Volume:** The intersection logic must correctly identify the exit point if the ray starts inside the bounding shape.
*   **Tightness:** Loose bounding shapes result in more "useless" steps. The optimization gain is directly proportional to how tightly the shape encloses the density [S1, unofficial].

## Subtopics

### [Convex Polyhedron Bounding](8027-convex-polyhedron-bounding.md)
Convex Polyhedron Bounding constructs a convex polyhedron around volumetric objects to enable efficient space-leaping during ray traversal. Unlike standard bounding boxes which provide loose axis-aligned approximations, this method creates a tighter geometric enclosure, reducing the number of density evaluations in empty space for scenes with few distinct objects. It serves as a specialized variant of Bounding Shape Optimization tailored for complex object geometries where axis-aligned bounds are insufficient.

### Container-based Ray Marching
Container-based Ray Marching uses closed geometry as a bounding container to restrict ray marching to regions where density actually exists [S2, unofficial]. The core principle is that using closed geometry as a bounding container is more efficient than marching through empty space [S2, unofficial]. Finer container definitions improve volume render efficiency compared to coarse or non-existent containers [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for intersection test; $O(N)$ for rendering within the bounds, where $N$ is the number of samples inside the bounding shape. Without the optimization, complexity is $O(M)$ where $M$ is the total ray length in the volume.
*   **Memory Complexity:** $O(1)$ for the bounding shape parameters.
*   **Performance:** Significant performance boost for scenes with mostly empty density fields [S1, unofficial]. The gain depends on the tightness of the bounding shape [S1, unofficial].
*   **Strengths:** Simple to implement; effective for sparse volumes; allows for adaptive step sizes within the volume [S1, unofficial].
*   **Limitations:** Requires pre-defining bounding shapes; less effective for dense, unbounded, or highly complex noise fields where tight bounding is difficult [S1, unofficial].
*   **When to Use:** Scenes with distinct, separable volumetric objects (e.g., dust clouds, solar flares) where empty space dominates [S1, unofficial].
*   **When Not to Use:** Dense, continuous volumes without clear empty regions.

## Background
This technique originates from research into real-time rendering of procedural noise fields, such as interstellar dust clouds and solar flares, where the hypothesis of mostly empty density fields allows for significant optimization [S1, unofficial]. It is also discussed in the context of production volume rendering, emphasizing the use of closed geometry containers to improve efficiency [S2, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Production Volume Rendering - Clemson University, https://jtessen.people.clemson.edu/reports/papers_files/ProdVolRender.pdf, fetched 2026-10-27 (unofficial)
