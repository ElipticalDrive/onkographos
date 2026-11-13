# Surface Peeling

Also called: Peeling Buffer

## Overview
Surface peeling is an interaction technique used to expose internal structures within a volume by selectively skipping the first $N$ ray-isosurface intersection points within specific regions of the image plane [S1, unofficial]. It differs from general isosurface rendering by introducing a per-pixel skip count that discards outer layers, allowing the viewer to see deeper geometric shells without modifying the underlying volume data or transfer functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset, a defined isosurface value, a set of "peeling windows" (rectangular regions on the image plane), and a peeling depth parameter $N$ for each window.
*   **Output:** A rendered image where isosurface layers within the peeling windows are hidden up to depth $N$, revealing subsequent layers.

**Data Structures:**
*   **Peeling Buffer:** An integer array with dimensions matching the image plane resolution ($W \times H$). Each element `buffer[x][y]` stores the number of intersection points to skip for the ray passing through pixel $(x, y)$ [S1, unofficial].
    *   **Initialization:** All elements are initialized to 0 [S1, unofficial].
    *   **Update:** For each pixel $(x, y)$ inside a defined peeling window, the value is incremented by the desired peeling depth $N$ (or by 1 for each layer peeled) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize Buffer:** Create an integer array `peeling_buffer` of size $W \times H$ and set all values to 0.
2.  **Define Windows:** Identify the rectangular regions (peeling windows) on the image plane where peeling should occur.
3.  **Set Peeling Values:** For each pixel $(x, y)$ within a peeling window, set `peeling_buffer[x][y] = N`, where $N$ is the number of layers to skip [S1, unofficial]. Pixels outside windows remain 0.
4.  **Ray Casting Loop:** For each pixel $(x, y)$ in the image plane:
    a.  Cast a ray through the volume.
    b.  Perform ray traversal and compute all intersections with the isosurface along the ray.
    c.  Maintain a counter `intersection_count`, initialized to 0.
    d.  For each intersection point found:
        i.  Increment `intersection_count`.
        ii. Check if `intersection_count <= peeling_buffer[x][y]`.
        iii. If true, skip rendering this intersection (discard its color/normal).
        iv. If false, render this intersection (compute shading, color, and depth) and terminate the ray for this pixel (assuming first-hit shading for the visible surface) [S1, unofficial].
5.  **Composite:** Accumulate the rendered color for each pixel into the final image buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
peeling_buffer = IntegerArray(width, height, 0)

# Define peeling windows
# Example: window covers pixels x in [x1, x2], y in [y1, y2]
# N is the number of layers to skip
for x in range(x1, x2 + 1):
    for y in range(y1, y2 + 1):
        peeling_buffer[x][y] = N

# Rendering
for x in range(width):
    for y in range(height):
        ray = cast_ray_through_pixel(x, y)
        intersections = find_isosurface_intersections(ray)
        
        intersection_count = 0
        visible_color = transparent_color
        
        for intersection in intersections:
            intersection_count += 1
            
            # Skip if within the peeling depth
            if intersection_count <= peeling_buffer[x][y]:
                continue
            
            # Render the first visible intersection
            color = compute_shading(intersection)
            visible_color = color
            break # Stop after finding the first visible surface layer
            
        image_buffer[x][y] = visible_color
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** If a ray intersects fewer than $N$ times within a peeling window, no surface is rendered for that pixel, potentially resulting in a transparent or background-colored pixel. This must be handled to avoid visual artifacts if the interior structure is expected [S1, unofficial].
*   **Precision:** Accurate ray-isosurface intersection calculation is critical. Inaccurate intersection positions may lead to incorrect voxel identification or skipped layers [S1, unofficial].

## Subtopics

### [Opacity Peeling](8428-opacity-peeling.md)
Opacity peeling is an interaction technique used to reveal interior hidden spatial structures within a volume by selectively discarding or modifying the opacity of surface layers along the viewing ray. It serves as a specific implementation of surface peeling, differing from general surface peeling by focusing on the attenuation of ray-marched contributions rather than explicit geometric intersection skipping. While surface peeling explicitly skips the first $N$ intersection points, opacity peeling typically achieves the "peeling" effect by adjusting the transfer function or compositing weights to ignore the outermost layers, thereby exposing deeper structures without requiring explicit geometric primitives.

## Operating characteristics
*   **Time Complexity:** The core ray casting complexity remains $O(R \cdot M)$, where $R$ is the number of rays and $M$ is the average number of intersections per ray. The peeling logic adds $O(1)$ overhead per intersection for the counter check.
*   **Memory Complexity:** Requires $O(W \cdot H)$ additional memory for the peeling buffer, where $W$ and $H$ are the image dimensions.
*   **Strengths:** Provides intuitive, interactive exploration of internal volume structures without altering the volume data or transfer functions. It is computationally inexpensive compared to full volume clipping or boolean operations.
*   **Limitations:** Only works for isosurface rendering (not direct volume rendering). It assumes distinct, separable isosurface layers. It does not provide depth information for the skipped layers, which may be required for some interaction tasks.
*   **When to Use:** When users need to quickly inspect internal structures of a complex isosurface without the computational cost of full volume clipping or the complexity of transfer function editing.
*   **When Not to Use:** When the volume does not have a clear isosurface structure, or when depth information for the hidden layers is required.

## Background
Surface peeling was introduced as an efficient interaction technique for volume visualization, particularly in the context of isosurface rendering. It allows for the brief exploration of internal structures by cropping and peeling layers [S1, unofficial]. The technique relies on the accurate identification of intersecting voxels and the use of buffers to store interaction metadata, such as peeling counts [S1, unofficial].

## Sources
[S1] Efficient and Effective Volume Visualization with Enhanced Isosurface Rendering, https://doi.org/10.48550/arxiv.1202.5360, fetched 2026-10-26 (unofficial)
