# Average Importance Compositing


## Overview
Average Importance Compositing is a rendering technique that modulates the visibility of objects along a viewing ray based on their relative importance, allowing less important objects to remain visible but sparse rather than being completely removed [S1, unofficial]. It differs from Maximum Importance Projection (MImP) by preserving the spatial relationships and high-curvature areas of occluding context objects, which MImP may obscure or distort [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   A set of $n$ objects intersected by a viewing ray $r$.
    *   An importance value $I_o$ for each object $o$.
    *   Footprint images for each object, which define the spatial coverage of the object in the image plane. These footprints are grayscale images with values in the range $[0, 1]$, where values below 1 represent the transition area [S1, unofficial].
*   **Outputs:**
    *   A compositing result for each pixel, where the contribution of each object is modulated by a sparseness factor and its footprint value [S1, unofficial].

**Data Structures**
*   **Footprint Image:** A 2D image (grayscale) corresponding to the object's projection. Each pixel value $F$ represents the coverage strength, ranging from 0 (no coverage) to 1 (full coverage) [S1, unofficial].
*   **Importance Value:** A scalar $I_o$ associated with each object [S1, unofficial].

**Steps**
1.  **Footprint Lookup:** For each pixel in the final image, determine which objects cover that pixel by looking up the footprint images of all intersected objects [S1, unofficial].
2.  **Sum Importance:** Calculate the sum of importance values for all objects covering the current pixel. Let $n_r$ be the number of objects intersected by the ray at the current pixel location. The sum is $\sum_{i=1}^{n_r} I_i$ [S1, unofficial].
3.  **Calculate Sparseness:** For each object $o$ covering the pixel, calculate its sparseness factor $S_o$ using the formula:
    $$ S_o = \frac{I_o}{\sum_{i=1}^{n_r} I_i} $$
    where $I_o$ is the importance of object $o$ and $I_i$ is the importance of the $i$-th intersected object [S1, unofficial].
4.  **Modulate Contribution:** Multiply the object's importance $I_o$ by its footprint value $F_o$ at the current pixel and by the sparseness factor $S_o$. The final contribution is proportional to $I_o \cdot F_o \cdot S_o$ [S1, unofficial]. Note that footprint values below 1 are part of the transition area, ensuring smooth boundaries [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_importance_compositing(ray, objects, footprints):
    """
    ray: The viewing ray.
    objects: List of objects intersected by the ray.
    footprints: Dictionary mapping object_id to its grayscale footprint image.
    
    Returns:
    result: The compositing result for the pixel.
    """
    # Step 1 & 2: Sum importance of all covering objects
    total_importance = 0.0
    covering_objects = []
    
    for obj in objects:
        # Lookup footprint value at current pixel coordinates
        # Assume get_footprint_value returns a float in [0, 1]
        footprint_val = get_footprint_value(obj.id, footprints[obj.id], ray.pixel_coords)
        
        # Only consider objects that have non-zero footprint coverage
        if footprint_val > 0:
            total_importance += obj.importance
            covering_objects.append((obj, footprint_val))
            
    # Handle edge case: no objects cover the pixel
    if total_importance == 0:
        return 0.0

    # Step 3 & 4: Calculate sparseness and modulate
    result = 0.0
    for obj, footprint_val in covering_objects:
        # Calculate sparseness factor
        sparseness = obj.importance / total_importance
        
        # Modulate contribution: importance * footprint * sparseness
        # The source states: "The object importance is additionally always 
        # multiplied by the footprint value... The final importance compositing 
        # is analogue to... Equation 2"
        # Equation 2 defines sparseness So. The compositing uses So.
        # The text implies the final weight is So * Footprint. 
        # Since So = Io / Sum(I), the term becomes Io * Footprint / Sum(I).
        
        contribution = obj.importance * footprint_val * sparseness
        result += contribution
        
    return result
```

**Parameters**
*   **Footprint Smoothing:** The footprint images must be smoothed to avoid sharp transitions. The source notes that binary footprints generate "jaggy object boundaries" and that footprints should be "grayscale instead of binary" to achieve smooth transitions [S1, unofficial]. The specific smoothing kernel or growth method is not detailed in the provided text [G].

**Edge Cases and Pitfalls**
*   **Sharp Transitions:** Using binary footprints results in visible jaggy object boundaries [S1, unofficial].
*   **Zero Sum:** If no objects cover a pixel, the sum of importance is zero, leading to division by zero. This must be handled by returning a background color or zero contribution [G].

## Operating characteristics
*   **Complexity:** The method requires iterating over all objects intersected by a ray to sum their importance and then iterating again to compute the final contribution. This is $O(n)$ per pixel, where $n$ is the number of intersected objects.
*   **Strengths:** It allows for a sparse representation of occluding objects while keeping high-curvature areas visible, providing better spatial relationship clarity than MImP [S1, unofficial].
*   **Limitations:** Without smoothing, it produces jagged boundaries [S1, unofficial]. It requires storing and accessing footprint images for all objects, increasing memory usage compared to methods that do not use footprints.

## Background
Average Importance Compositing is part of the importance-driven volume rendering framework. It was introduced as a method to handle the visibility of multiple objects along a ray by considering their relative importance [S1, unofficial]. It is an alternative to Maximum Importance Projection (MImP), which completely removes less important objects [S1, unofficial].

## Sources
[S1] (PDF) Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
