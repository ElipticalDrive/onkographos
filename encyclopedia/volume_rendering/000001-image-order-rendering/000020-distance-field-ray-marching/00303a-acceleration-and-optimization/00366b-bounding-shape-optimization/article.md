# Bounding Shape Optimization

Also called: Bounding Shape Culling

## Overview
Bounding Shape Optimization reduces the computational cost of volumetric rendering by skipping density evaluations in empty space using pre-defined geometric boundaries [S1, unofficial]. Unlike naive ray marching which samples the entire ray length, this method restricts computation to regions where density is non-zero, significantly improving performance for scenes with mostly empty density fields [S1, unofficial]. It sets itself apart from [Beam Optimization](932-beam-optimization.md) by using explicit geometric primitives rather than beam-based culling, and from [Block Culling](5431-block-culling.md) by relying on continuous shape intersections rather than discrete grid blocks.

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, and a set of bounding shapes defining the volume extent.
**Outputs:** Rendered color and opacity for the ray segment inside the bounding shapes.

**Data Structures:**
The method relies on a list of bounding shapes. Each shape must support:
1.  Intersection testing with a ray $\mathbf{o} + t\mathbf{d}$.
2.  Return of entry and exit distances ($t_{entry}, t_{exit}$).

For the specific implementation cited in [S1, unofficial], the chosen shape is a sphere. A sphere is defined by center $\mathbf{c}$ and radius $r$.

**Steps:**
1.  **Define Boundaries:** Identify regions where density is non-zero and define a bounding shape (e.g., sphere, cylinder, box) around them [S1, unofficial]. The closer the shape is to the actual density field, the fewer "useless" steps are performed [S1, unofficial].
2.  **Ray-Shape Intersection:** Use a surface rendering technique to find the intersection points of the ray with the bounding shape [S1, unofficial]. For a sphere, this involves solving the quadratic equation derived from $|\mathbf{o} + t\mathbf{d} - \mathbf{c}|^2 = r^2$.
3.  **Skip Empty Space:** If the ray does not intersect the bounding shape, skip the ray entirely (return background color) [S1, unofficial]. If it intersects, calculate the segment $[t_{entry}, t_{exit}]$ inside the shape.
4.  **Volumetric Rendering:** Perform classical volumetric rendering (ray marching) only within the interval $[t_{entry}, t_{exit}]$ [S1, unofficial].
5.  **Step Size Adaptation (Optional):** As noted in [S1, unofficial], the implementation uses increasingly larger steps as the ray progresses through the medium to boost performance, assuming imprecisions have less impact further along the ray.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray_with_bounding_shape(ray_origin, ray_direction, bounding_shapes):
    """
    Renders a ray using bounding shape optimization.
    bounding_shapes: List of objects with method intersect(ray_origin, ray_direction)
                     returning (t_entry, t_exit) or None if no intersection.
    """
    # Step 2: Find the first intersecting bounding shape
    closest_entry = float('inf')
    closest_exit = float('inf')
    found_intersection = False

    for shape in bounding_shapes:
        t_entry, t_exit = shape.intersect(ray_origin, ray_direction)
        if t_entry is not None and t_entry < closest_entry:
            closest_entry = t_entry
            closest_exit = t_exit
            found_intersection = True

    # If no shape is hit, skip rendering
    if not found_intersection:
        return background_color

    # Step 4: Render only inside the bounding shape
    color = vec3(0.0)
    opacity = 0.0
    
    # Start marching from t_entry
    t = closest_entry
    
    # Step 5: Adaptive step size [G] - Initial step size
    step_size = 0.1 
    
    while t < closest_exit and opacity < 1.0:
        # Sample density at current position
        pos = ray_origin + t * ray_direction
        density = sample_density_field(pos)
        
        # Accumulate color and opacity
        # Assuming standard volume rendering equation:
        # C_out = C_in + C_sample * alpha_sample * (1 - alpha_in)
        # alpha_out = alpha_in + alpha_sample * (1 - alpha_in)
        
        # Simplified accumulation for demonstration
        if density > 0:
            sample_color = get_color_at(pos, density)
            sample_opacity = density * step_size # Alpha approximation
            
            # Blend sample into current accumulated color
            weight = sample_opacity * (1.0 - opacity)
            color += sample_color * weight
            opacity += weight
            
        # Increase step size as per [S1, unofficial] observation
        # [G] The exact formula for step size increase is not provided in sources.
        # A common heuristic is linear or exponential growth.
        step_size *= 1.1 
        
        t += step_size

    return color
```

**Parameters:**
*   **Bounding Shape Choice:** [S1, unofficial] selected Sphere Marching for implementation.
*   **Container Definition Fineness:** Finer container definitions improve volume render efficiency by reducing the volume of empty space traversed [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Loose Bounds:** If bounding shapes are not tight to the density field, the optimization yields fewer benefits due to increased "useless" steps [S1, unofficial].
*   **Multiple Intersections:** If a ray intersects multiple shapes, the renderer must handle the union of intervals or prioritize the closest one depending on the desired visual effect (e.g., composite vs. opaque). The sources do not specify handling for complex overlapping volumes beyond the single-shape example.
*   **Deformation:** The sources mention deformation techniques (bending/stretching) for density fields [S1, unofficial], implying that bounding shapes may need to be updated or defined in local space if the volume deforms.

## Subtopics

### [Convex Polyhedron Bounding](8027-convex-polyhedron-bounding.md)
Convex Polyhedron Bounding constructs a convex polyhedron around volumetric objects to enable efficient space-leaping during ray traversal. Unlike standard bounding boxes which provide loose axis-aligned approximations, this method creates a tighter geometric enclosure, reducing the number of density evaluations in empty space for scenes with few distinct objects. It serves as a specialized variant of Bounding Shape Optimization tailored for complex object geometries where axis-aligned bounds are insufficient.

### Container-based Ray Marching
Container-based Ray Marching uses closed geometry as a bounding container to restrict ray marching to regions where density actually exists, improving efficiency [S2, unofficial]. It is a technique to optimize volume rendering by defining the volume extent via closed geometry [S2, unofficial]. Finer container definitions improve volume render efficiency compared to coarse or non-existent containers [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of density evaluations from $O(L)$ (where $L$ is ray length) to $O(L_{inside})$, where $L_{inside}$ is the length of the ray segment within the bounding shape. The cost of intersection testing is $O(1)$ per shape.
*   **Memory Complexity:** Requires storage for the bounding shapes. For a single sphere, this is minimal (center and radius).
*   **Performance:** Significant performance boost for scenes with mostly empty density fields [S1, unofficial]. The exact speedup depends on the tightness of the bounds and the ratio of empty to dense space.
*   **Strengths:** Simple to implement; effective for sparse volumes; compatible with various surface rendering techniques for intersection [S1, unofficial].
*   **Limitations:** Less effective for dense, continuous volumes where the ray spends most of its time inside the density field; requires careful tuning of bound tightness [S1, unofficial].
*   **When to Use:** Ideal for scenes with distinct, sparse objects or procedural noise fields with large empty regions [S1, unofficial].
*   **When Not to Use:** Not suitable for dense, homogeneous volumes or when the overhead of intersection testing exceeds the savings from skipped samples.

## Background
Bounding Shape Optimization originates from techniques for rendering volumetric data with mostly empty density fields. It was notably discussed in the context of real-time rendering of interstellar dust clouds and solar flares [S1, unofficial]. The concept of using containers to limit ray marching is also present in production volume rendering literature [S2, unofficial]. The method leverages standard geometric intersection algorithms, often repurposing surface rendering techniques for volume culling [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Production Volume Rendering - Clemson University, https://jtessen.people.clemson.edu/reports/papers_files/ProdVolRender.pdf, fetched 2026-10-27 (unofficial)
