# Average Importance Compositing

Also called: AImC

## Overview
Average Importance Compositing (AImC) is an importance-driven volume rendering technique that partially suppresses less important objects along a viewing ray rather than removing them completely, allowing sparse representation of occluding structures while preserving high-curvature areas and spatial relationships [S2, S3, S4, unofficial]. Unlike Maximum Importance Projection (MImP), which completely removes less important objects, this method assigns a sparseness level to each object based on the fraction of its importance relative to the sum of importance of all intersected objects, ensuring that context objects remain visible but attenuated [S2, S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of segmented objects (or features) along a viewing ray $r$, each with an associated importance value $I_o$ [S3, S4, unofficial]. Footprint images for each object, which can be binary or grayscale [S1][S3, S4, unofficial].
*   **Output:** A rendered pixel color and opacity, where the contribution of each object is modulated by its calculated sparseness factor $S_o$ [S3, S4, unofficial].

**Data Structures:**
*   **Object Importance ($I_o$):** A scalar value representing the visual or semantic importance of object $o$ [S3, S4, unofficial].
*   **Footprint Image:** A 2D image (or mask) corresponding to the projection of object $o$ [S1][S3, unofficial]. In the basic binary approach, values are 0 or 1 [S3, S4, unofficial]. In the smooth transition approach, values are in $[0, 1]$ derived from a distance transform [S1][S3, S4, unofficial].
*   **Ray Intersection List:** A list of objects intersected by ray $r$, indexed $i = 1 \dots n_r$ [S3, S4, unofficial].

**Steps:**
1.  **Cast Ray:** Cast a ray $r$ through the volume [S1].
2.  **Identify Intersections:** Determine the set of $n_r$ objects intersected by ray $r$ [S3, S4, unofficial].
3.  **Sum Importance:** Calculate the total importance $I_{sum}$ along the ray by summing the importance values of all intersected objects:
    $$ I_{sum} = \sum_{i=1}^{n_r} I_i $$
    [S1][S3, S4, unofficial]
4.  **Calculate Sparseness:** For each object $o$ intersected by the ray, calculate its sparseness factor $S_o$:
    $$ S_o = \frac{I_o}{I_{sum}} $$
    [S1][S3, S4, unofficial]
5.  **Evaluate Footprint Coverage:** For each pixel, lookup the footprint value $F_o(p) \in [0, 1]$ for each object [S1][S3, S4, unofficial].
6.  **Modulate Optical Properties:** Combine the object's color $C_o$ and opacity $\alpha_o$ with the sparseness factor $S_o$ and footprint value $F_o(p)$ [S3, S4, unofficial]. The effective opacity $\alpha'_o$ and color $C'_o$ are modulated, typically as $\alpha'_o = \alpha_o \cdot S_o \cdot F_o(p)$ [S1][S3, S4, unofficial]. Color saturation can also be modulated by $S_o$ to attract focus [S4, unofficial].
7.  **Composite:** Perform standard front-to-back over-operator compositing with these modulated values [S3, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_importance_compositing(ray_origin, ray_direction, step_size, objects, pixel_u, pixel_v):
    """
    ray_origin: tuple (x, y, z)
    ray_direction: tuple (dx, dy, dz)
    step_size: float
    objects: list of dicts with keys:
             - 'id': unique identifier
             - 'importance': float I_o
             - 'footprint_func': callable(u, v) returning [0, 1]
             - 'color': tuple (r, g, b)
             - 'opacity': float alpha_o
             - 'bbox': tuple (min_x, min_y, min_z, max_x, max_y, max_z)
    pixel_u, pixel_v: float screen coordinates for footprint lookup
    Returns: list [r, g, b, alpha]
    """
    # Step 1: Identify intersected objects and sum importance
    intersected = []
    total_importance = 0.0
    for obj in objects:
        if ray_intersects_aabb(ray_origin, ray_direction, obj['bbox']):
            intersected.append(obj)
            total_importance += obj['importance']
            
    # Handle edge case: no objects or zero importance
    if total_importance == 0:
        return [0.0, 0.0, 0.0, 0.0]
        
    # Step 2: Compute sparseness factors per object
    sparseness = {}
    for obj in intersected:
        sparseness[obj['id']] = obj['importance'] / total_importance
        
    # Step 3: Ray traversal and compositing
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    
    t = 0.0
    while t < 1.0 and accumulated_opacity < 1.0:
        # Determine which object is at current sample
        current_obj = None
        for obj in intersected:
            if point_in_aabb(ray_origin + ray_direction * t, obj['bbox']):
                current_obj = obj
                break
                
        if current_obj is not None:
            # Get footprint value at current pixel coordinates
            footprint_val = current_obj['footprint_func'](pixel_u, pixel_v)
            
            # Modulate opacity: alpha'_o = alpha_o * S_o * F_o(p)
            s_o = sparseness[current_obj['id']]
            modulated_opacity = current_obj['opacity'] * s_o * footprint_val
            
            if modulated_opacity > 1e-6:
                remaining = 1.0 - accumulated_opacity
                accumulated_color[0] += current_obj['color'][0] * modulated_opacity * remaining
                accumulated_color[1] += current_obj['color'][1] * modulated_opacity * remaining
                accumulated_color[2] += current_obj['color'][2] * modulated_opacity * remaining
                accumulated_opacity += modulated_opacity * remaining
                
        t += step_size
        
    return accumulated_color + [accumulated_opacity]

def ray_intersects_aabb(origin, direction, bbox):
    min_x, min_y, min_z, max_x, max_y, max_z = bbox
    tmin = (min_x - origin[0]) / direction[0] if direction[0] != 0 else -float('inf')
    tmax = (max_x - origin[0]) / direction[0] if direction[0] != 0 else float('inf')
    if tmin > tmax: tmin, tmax = tmax, tmin
    tymin = (min_y - origin[1]) / direction[1] if direction[1] != 0 else -float('inf')
    tymax = (max_y - origin[1]) / direction[1] if direction[1] != 0 else float('inf')
    if tymin > tymax: tymin, tymax = tymax, tymin
    if (tmin > tymax) or (tymin > tmax): return False
    if tymin > tmin: tmin = tymin
    if tymax < tmax: tmax = tymax
    tzmin = (min_z - origin[2]) / direction[2] if direction[2] != 0 else -float('inf')
    tzmax = (max_z - origin[2]) / direction[2] if direction[2] != 0 else float('inf')
    if tzmin > tzmax: tzmin, tzmax = tzmax, tzmin
    if (tmin > tzmax) or (tzmin > tmax): return False
    if tzmin > tmin: tmin = tzmin
    return tmax >= 0

def point_in_aabb(point, bbox):
    min_x, min_y, min_z, max_x, max_y, max_z = bbox
    return min_x <= point[0] <= max_x and min_y <= point[1] <= max_y and min_z <= point[2] <= max_z
```

**Parameters:**
*   **Sparseness Factor ($S_o$):** The normalized importance fraction $I_o / \sum I_i$ [S1][S3, S4, unofficial]. Example values reported in results are 0.75, 0.5, and 0.25 [S4, S5, unofficial].
*   **Footprint Type:** Binary or Grayscale (Distance Transform) [S1][S3, S4, unofficial]. Grayscale is preferred for smooth transitions [S1][S3, S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Sharp Transitions:** Using binary footprints results in jaggy object boundaries and sharp transitions in visibility [S3, S4, unofficial].
*   **Zero Importance:** If the sum of importance is zero, division by zero occurs [G].
*   **Occlusion Artifacts:** The image-space approach does not evaluate whether a suppressed context object sample is in front or behind the important object; sparseness is constant for all samples of one object along a ray, suppressing parts behind the focus object [S1].

## Subtopics

### Importance Metric I1
An importance metric based on accumulated blending weights, defined as the sum of blending weights of a Gaussian across all intersecting rays ($I_1^i = \sum_{j=1}^{K} w_{ij}$) [S6]. This metric is particularly effective for indoor scenes with low-uncertainty Gaussians but may preserve background elements in outdoor scans if not filtered [S6].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of intersected objects per ray $n_r$ for the importance summation and sparseness calculation [G].
*   **Memory Complexity:** Requires storage for footprint images for each object [G].
*   **Performance:** The method is computationally more expensive than MImP due to the need to sum importance values and calculate sparseness for all objects along the ray, rather than just selecting the maximum [S3, S4, unofficial].
*   **Strengths:** Preserves spatial relationships between focus and context objects; allows high-curvature areas of occluding objects to remain visible; provides smooth transitions with grayscale footprints [S2, S3, S4, unofficial].
*   **Limitations:** Sharp transitions with binary footprints; requires precomputed importance values and footprints; early implementations were not interactive [S2, S4, unofficial].
*   **When to Use:** When it is critical to see the spatial arrangement of multiple objects and the relative importance of occluding structures, rather than just the single most important object [S2, S4, unofficial].

## Background
Average Importance Compositing was introduced by Ivan Viola and M. Eduard Gröller in their 2004 paper "Importance-Driven Volume Rendering" [S3, S4, unofficial]. It is part of a family of importance-driven rendering techniques that aim to automatically guide the viewer's attention in volume visualization [S3, S4, unofficial]. The method was implemented as a plugin in the J-Vision medical workstation and demonstrated on datasets such as the Leopard Gecko and Monster Study [S4, S5, unofficial]. It builds on the concept of importance-driven rendering, where visual cues like opacity and color saturation are modulated based on object importance [S3, S4, unofficial].

## Sources
[S1] Importance-Driven Feature Enhancement in Volume Visualization, https://doi.org/10.1109/tvcg.2005.62, fetched 2026-11-05
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S3] (PDF) Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
[S4] Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-ImpX/Viola-2004-ImpX-paper.pdf, fetched 2026-10-29 (unofficial)
[S5] PDF Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/vis/adapt/2004_idvr/idvr.pdf, fetched 2026-10-29 (unofficial)
[S6] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
