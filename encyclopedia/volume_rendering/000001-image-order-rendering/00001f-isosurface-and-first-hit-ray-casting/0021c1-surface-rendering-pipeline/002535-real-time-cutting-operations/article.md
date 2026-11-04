# Real Time Cutting Operations

Also called: Cutting Planes via Depth Buffers

## Overview
Real Time Cutting Operations is a technique for clipping volume surfaces using cutting planes by maintaining near and far depth buffers per pixel, avoiding full re-rendering when planes change [S1, unofficial]. It sets itself apart from siblings by pre-computing visibility intervals via scan-conversion of planes into depth buffers, allowing the renderer to discard intersections outside these intervals without re-tracing the entire volume [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of cutting planes $C_i$ defined by normal $N_i$ and offset; a ray $R(p)$ for each pixel $p$ in the image plane; a volume dataset.
*   **Outputs:** A rendered image where surface points are displayed only if their depth lies within the computed near/far bounds.

**Data Structures:**
1.  **Near Depth Buffer ($D_{near}$):** Array of size equal to image resolution. Stores the maximum depth of "facing same direction" planes or parallel planes in negative halfspace.
2.  **Far Depth Buffer ($D_{far}$):** Array of size equal to image resolution. Stores the minimum depth of "facing opposite direction" planes.
3.  **Intersection Buffer:** Per-pixel storage for all surface intersections along the ray. Layout for pixel $(i,j)$:
    *   $k_{i,j}$: Count of intersections.
    *   $D(i,j)$: Array of depths for each intersection.
    *   $I(i,j)$: Array of intensities for each intersection.
    *   *Note:* The source notation $S(i,k)$ appears to refer to the count or a specific intersection attribute, but the text clarifies $k$ is the count and $D/I$ are arrays indexed by intersection number [S1, unofficial].

**Steps:**
1.  **Initialize Buffers:** Set all elements in $D_{near}$ to $0$. Set all elements in $D_{far}$ to $\text{MAX\_DEPTH}$ [S1, unofficial].
2.  **Classify Planes:** For each pixel $p$, classify each cutting plane $C_i$ relative to ray $R(p)$:
    *   Parallel: $R(p) \cdot N_i = 0$.
    *   Facing Same Direction: $R(p) \cdot N_i > 0$.
    *   Facing Opposite Direction: $R(p) \cdot N_i < 0$.
3.  **Update Depth Buffers:**
    *   If $C_i$ is parallel to $R(p)$ AND pixel $p$ is in the negative halfspace of the plane ($C_i(p) < 0$), set $D_{near}(p) \leftarrow \text{MAX\_DEPTH}$ and $D_{far}(p) \leftarrow 0$ [S1, unofficial].
    *   If $C_i$ faces the same direction as $R(p)$, compute intersection depth $d$. Update $D_{near}(p) = \max(D_{near}(p), d)$ [S1, unofficial].
    *   If $C_i$ faces the opposite direction to $R(p)$, compute intersection depth $d$. Update $D_{far}(p) = \min(D_{far}(p), d)$ [S1, unofficial].
4.  **Ray Casting and Intersection:** Cast rays into the volume. Store all surface intersections in the Intersection Buffer, recording depth $D(i,j)$ and intensity $I(i,j)$ for each intersection [S1, unofficial].
5.  **Visibility Test:** For each intersection at depth $d$ in pixel $p$, display the surface point if $D_{near}(p) \le d \le D_{far}(p)$. Discard points where $d < D_{near}(p)$ or $d > D_{far}(p)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def real_time_cutting_operations(cutting_planes, image_width, image_height, max_depth):
    # Initialize depth buffers
    D_near = [0.0] * (image_width * image_height)
    D_far = [max_depth] * (image_width * image_height)
    
    # Process each pixel
    for p in range(image_width * image_height):
        ray = get_ray_for_pixel(p)
        
        for plane in cutting_planes:
            # Classify plane relative to ray
            dot_product = dot(ray.direction, plane.normal)
            
            if abs(dot_product) < epsilon:
                # Parallel plane
                if evaluate_plane(plane, ray.origin + ray.direction * 0.0) < 0:
                    # Pixel in negative halfspace
                    D_near[p] = max_depth
                    D_far[p] = 0.0
            elif dot_product > 0:
                # Facing same direction
                depth = compute_intersection_depth(ray, plane)
                if depth is not None:
                    D_near[p] = max(D_near[p], depth)
            else:
                # Facing opposite direction
                depth = compute_intersection_depth(ray, plane)
                if depth is not None:
                    D_far[p] = min(D_far[p], depth)
                    
    # Ray casting and filtering
    intersections = []
    for p in range(image_width * image_height):
        ray = get_ray_for_pixel(p)
        # Cast ray and collect all intersections
        ray_intersections = cast_ray_into_volume(ray)
        
        for intersection in ray_intersections:
            if D_near[p] <= intersection.depth <= D_far[p]:
                intersections.append({
                    'pixel': p,
                    'depth': intersection.depth,
                    'intensity': intersection.intensity
                })
                
    return intersections
```

**Parameters:**
*   $\text{MAX\_DEPTH}$: Maximum depth value for the volume or scene. Not given in the sources.
*   $\epsilon$: Epsilon for floating point comparison for parallel detection. Not given in the sources.

**Edge Cases:**
*   **Parallel Planes:** If a plane is parallel to the ray and the pixel is in the negative halfspace, the near buffer is set to max and far to zero, effectively disabling visibility for that pixel [S1, unofficial].
*   **No Intersections:** If no intersections are found, nothing is displayed.
*   **Precision:** Floating point comparisons for parallel detection require an epsilon threshold [G].

## Operating characteristics
*   **Time Complexity:** Pre-processing depth buffers is $O(P \cdot C)$ where $P$ is number of pixels and $C$ is number of cutting planes. Ray casting is $O(R \cdot I)$ where $R$ is number of rays and $I$ is average intersections per ray. Visibility test is $O(I)$ per ray.
*   **Memory Complexity:** $O(P)$ for depth buffers plus $O(R \cdot I)$ for intersection buffer.
*   **Strengths:** Allows interactive specification of cutting planes without re-tracing the entire volume for every change [S1, unofficial].
*   **Limitations:** Requires storing all intersections before filtering, which can be memory intensive for complex volumes [S1, unofficial].
*   **When to use:** When interactive cutting is required and the volume has sparse or well-defined surfaces [S1, unofficial].
*   **When not to use:** When memory is constrained or the volume has extremely dense intersections [G].

## Background
The method was introduced in "Fast Cutting Operations on Three Dimensional Volume Datasets" [S1, unofficial]. It addresses the need for efficient clipping in volume rendering by decoupling plane updates from ray casting.

## Sources
[S1] Fast Cutting Operations on Three Dimensional Volume Datasets, https://zenodo.org/records/14173364, fetched 2026-10-26 (unofficial)
