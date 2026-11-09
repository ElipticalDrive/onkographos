# Average Importance Compositing


## Overview
Average Importance Compositing is a rendering technique that modulates the visibility of objects along a viewing ray based on their relative importance, allowing less important objects to remain visible but sparse rather than being completely removed [S2, unofficial]. It differs from Maximum Importance Projection (MImP) by preserving the spatial relationships and high-curvature areas of occluding context objects, which MImP may obscure [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   A set of $n$ objects intersected by a viewing ray $r$ [S2, unofficial].
    *   An importance value $I_o$ for each object $o$ [S2, unofficial].
    *   Footprint images for each object, which define the spatial coverage of the object in the image plane [S1, unofficial].
*   **Outputs:**
    *   A compositing result for each pixel, where the contribution of each object is modulated by a sparseness factor and its footprint value [S2, unofficial].

**Data Structures**
*   **Footprint Image:** A 2D image (grayscale) corresponding to the object's projection [S2, unofficial]. Each pixel value $F_o$ represents the footprint coverage, ranging from 0 (no coverage) to 1 (full coverage), with values below 1 indicating the transition area [S2, unofficial].
*   **Importance Value:** A scalar $I_o$ associated with each object [S2, unofficial].

**Steps**
1.  **Footprint Lookup:** For each pixel in the final image, determine which objects cover that pixel by looking up the footprint images of all intersected objects [S1, unofficial].
2.  **Sum Importance:** Calculate the sum of importance values for all objects covering the current pixel. Let $n_r$ be the number of objects intersected by the ray at the current pixel location. The sum is $\sum_{i=1}^{n_r} I_i$ [S2, unofficial].
3.  **Calculate Sparseness:** For each object $o$ covering the pixel, calculate its sparseness factor $S_o$ using the formula:
    $$ S_o = \frac{I_o}{\sum_{i=1}^{n_r} I_i} $$
    where $I_o$ is the importance of object $o$ and $I_i$ is the importance of the $i$-th intersected object [S2, unofficial].
4.  **Modulate Contribution:** Multiply the object's importance $I_o$ by its footprint value $F_o$ at the current pixel and by the sparseness factor $S_o$ [S2, unofficial]. Note that footprint values below 1 are part of the transition area, ensuring smooth boundaries [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_importance_compositing(ray_pixel_coords, objects, footprints):
    """
    ray_pixel_coords: (x, y) coordinates of the current pixel.
    objects: List of objects intersected by the ray. Each object has .id and .importance.
    footprints: Dictionary mapping object_id to a 2D list representing the grayscale footprint image.
    
    Returns:
    result: The accumulated compositing weight for the pixel.
    """
    total_importance = 0.0
    covering_objects = []
    
    # Step 1 & 2: Lookup footprints and sum importance
    for obj in objects:
        # Inline footprint lookup: retrieve value at pixel coordinates
        # Assumes footprints[obj.id] is a 2D list indexed as [y, x]
        footprint_val = footprints[obj.id][int(ray_pixel_coords[1])][int(ray_pixel_coords[0])]
        
        # Only consider objects with non-zero coverage
        if footprint_val > 0.0:
            total_importance += obj.importance
            covering_objects.append((obj, footprint_val))
            
    # Handle edge case: no objects cover the pixel
    if total_importance == 0.0:
        return 0.0

    # Step 3 & 4: Calculate sparseness and modulate contribution
    result = 0.0
    for obj, footprint_val in covering_objects:
        # Calculate sparseness factor S_o = I_o / sum(I_i)
        sparseness = obj.importance / total_importance
        
        # Modulate contribution: importance * footprint * sparseness
        contribution = obj.importance * footprint_val * sparseness
        result += contribution
        
    return result
```

**Parameters**
*   **Footprint Smoothing:** The footprint images must be smoothed to avoid sharp transitions [S2, unofficial]. The source notes that binary footprints generate "jagged object boundaries" and that footprints should be converted to grayscale to achieve smooth transitions [S2, unofficial]. The specific smoothing kernel or growth method is not detailed in the provided text [G].

**Edge Cases and Pitfalls**
*   **Sharp Transitions:** Using binary footprints results in visible jaggy object boundaries [S2, unofficial].
*   **Zero Sum:** If no objects cover a pixel, the sum of importance is zero, leading to division by zero. This must be handled by returning a background color or zero contribution [G].

## Operating characteristics
*   **Complexity:** The method requires iterating over all objects intersected by a ray to sum their importance and then iterating again to compute the final contribution. This is $O(n)$ per pixel, where $n$ is the number of intersected objects [G].
*   **Strengths:** It allows for a sparse representation of occluding objects while keeping high-curvature areas visible, providing better spatial relationship clarity than MImP [S2, unofficial].
*   **Limitations:** Without smoothing, it produces jagged boundaries [S2, unofficial]. It requires storing and accessing footprint images for all objects, increasing memory usage compared to methods that do not use footprints [G].
*   **When to use:** Choose when preserving spatial relationships and high-curvature details of less important context objects is prioritized over complete occlusion [S2, unofficial].

## Background
Average Importance Compositing was introduced as part of the importance-driven volume rendering framework to handle the visibility of multiple objects along a ray by considering their relative importance [S2, unofficial]. It was developed as an alternative to Maximum Importance Projection (MImP), which completely removes less important objects [S2, unofficial].

## Sources
[S1] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S2] (PDF) Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
