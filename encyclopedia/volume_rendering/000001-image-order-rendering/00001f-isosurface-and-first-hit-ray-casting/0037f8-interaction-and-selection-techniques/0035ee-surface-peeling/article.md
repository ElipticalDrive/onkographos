# Surface Peeling

Also called: Peeling Buffer

## Overview
Surface peeling is an interaction technique in isosurface rendering that exposes internal structures by skipping the first $N$ intersection points within specific image plane regions, known as peeling windows [S1, unofficial]. It differs from cropping, which truncates the volume with a bounding box, by allowing selective exposure of deeper layers without altering the volume data or view frustum [S1, unofficial].

## How it works
**Inputs:**
- Isosurface geometry (implicit function or voxel grid).
- Peeling buffer: an integer array initialized to 0, where each element corresponds to a pixel or region in the image plane [S1, unofficial].
- Peeling windows: user-defined regions on the image plane where peeling is active [S1, unofficial].

**Outputs:**
- Rendered image showing isosurface layers deeper than the skipped count [S1, unofficial].

**Data Structures:**
- **Peeling Buffer**: An integer array $B$ of size $W \times H$ (image resolution). $B[x, y]$ stores the number of intersections to skip for pixel $(x, y)$. Initialized to 0 [S1, unofficial].

**Steps:**
1. Initialize the peeling buffer $B$ with zeros [S1, unofficial].
2. For each peeling window defined by the user:
   - Identify pixels $(x, y)$ within the window [S1, unofficial].
   - Increment $B[x, y]$ by 1 for each layer to expose (i.e., if exposing the second layer, increment by 1; for the third, increment by 2, etc.) [S1, unofficial].
3. For each pixel $(x, y)$ in the image plane:
   - Cast a ray through the volume [S1, unofficial].
   - Count the number of ray-isosurface intersections, denoted as $C$ [S1, unofficial].
   - If $C \le B[x, y]$, skip the intersection (do not render) [S1, unofficial].
   - Otherwise, render the intersection [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize peeling buffer
peeling_buffer = [[0 for _ in range(width)] for _ in range(height)]

# Define peeling windows and update buffer
for window in peeling_windows:
    for x, y in window.pixels:
        peeling_buffer[y][x] += 1  # Increment for each layer to expose

# Rendering pass
for y in range(height):
    for x in range(width):
        ray = cast_ray(x, y)
        intersections = find_isosurface_intersections(ray)
        count = len(intersections)
        skip_count = peeling_buffer[y][x]
        
        if count <= skip_count:
            continue  # Skip rendering this intersection
        else:
            render_intersection(intersections[skip_count])  # Render the first non-skipped intersection
```

**Parameters:**
- Peeling value: The number of intersections to skip per pixel, stored in the peeling buffer [S1, unofficial].

**Edge Cases:**
- If the number of intersections is less than or equal to the peeling value, no intersection is rendered for that pixel [S1, unofficial].
- Precision issues may arise if the ray traversal does not accurately count intersections [S1, unofficial].

## Subtopics

### Opacity Peeling
A technique employed by Rezk-Salama and Kolb in the rendering pipeline to reveal interior hidden spatial structures [S2]. It is similar to surface peeling but may involve opacity-based adjustments rather than strict intersection skipping [S2].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of pixels and the number of intersections per ray [S1, unofficial].
- **Memory Complexity**: Proportional to the image resolution for the peeling buffer [S1, unofficial].
- **Strengths**: Efficient single-pass solution for exposing internal structures [S2].
- **Limitations**: Limited to isosurface rendering and may not work well with complex volume data [S1, unofficial].

## Background
Surface peeling was introduced as an interaction technique for isosurface rendering to allow users to explore internal structures of volumetric data [S1, unofficial]. It is often used in conjunction with cropping for brief exploration of volumetric datasets [S1, unofficial].

## Sources
[S1] Efficient and Effective Volume Visualization with Enhanced Isosurface Rendering, https://doi.org/10.48550/arxiv.1202.5360, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
