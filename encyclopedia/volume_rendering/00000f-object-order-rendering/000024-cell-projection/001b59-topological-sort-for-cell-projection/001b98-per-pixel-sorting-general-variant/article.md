# Per-Pixel Sorting (General Variant)

Also called: Wilhelms et al. Per-Pixel Sort, Wilhelms et al. Sorting

## Overview
Per-Pixel Sorting (General Variant) is an image-space visibility ordering method that sorts all cell faces projecting onto a specific pixel at render time, just prior to compositing [S2, unofficial][S3, unofficial]. It solves the problem of determining the correct back-to-front rendering order for transparent volumes in scenes where a global topological sort is impossible due to visibility cycles [S2, unofficial]. Unlike its sibling [Exact Visibility Ordering for Cell Projection Volume Rendering (SXMPVO)](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md), which sorts only exterior faces and exploits spatial coherence, this variant sorts *all* faces (including interior ones) independently or incrementally per pixel [S2, unofficial]. It is distinguished from [Mammen's Technique](1279-mammen-s-technique.md) by its broader application to general cell projection rather than just transparent fragments in front of opaque ones, though it shares similar data structures [S3, unofficial].

## How it works
**Inputs:** A set of convex polyhedral cells forming a volume mesh; a viewpoint; a rasterization grid (screen).
**Outputs:** A sorted list of face fragments for each pixel, ordered from back to front.

**Data Structures:**
The method adapts scan-line visible surface algorithms using the following structures [S1][S3, unofficial]:
1.  **Y-Buckets:** A list of lists indexed by scan line $y$. Each bucket contains faces that first cross that scan line.
2.  **Active List:** A list of faces currently overlapping the current scan line $y$.
3.  **X-Buckets:** A list of lists indexed by pixel column $x$ within the current scan line. Each bucket contains faces covering the first pixel of that column on the current scan line.
4.  **Z-Sorted List:** A list of faces at the current pixel $(x, y)$, sorted by their depth $z$ value at that pixel.

**Steps:**
1.  **Initialization:** Create empty Y-Buckets for all scan lines.
2.  **Scan Line Processing:** Iterate through each scan line $y$ from top to bottom (or bottom to top, depending on coordinate system).
    3.  **Update Active List:** For the current scan line $y$, add all faces found in Y-Bucket $y$ to the Active List. Remove faces from the Active List that no longer overlap the current scan line.
    4.  **Pixel Processing:** Iterate through each pixel column $x$ on the current scan line.
        5.  **Update X-Buckets:** For the current pixel $(x, y)$, add all faces found in X-Bucket $x$ to the local processing set.
        6.  **Filter Active List:** Identify faces in the Active List that cover the current pixel $(x, y)$.
        7.  **Sort:** Sort the identified faces by their depth $z$ at the pixel center $(x, y)$. If the previous pixel $(x-1, y)$ was processed, the sort can be updated incrementally rather than performed from scratch, taking partial advantage of coherence in $x$ [S1][S3, unofficial].
        8.  **Compositing:** Composite the sorted faces from back to front.
        9.  **Cleanup:** Prepare for the next pixel. If not using incremental update, clear the Z-Sorted List.

**Formulas:**
*   Depth $z$ at pixel $(x, y)$ for a planar face with vertices $\mathbf{v}_i = (x_i, y_i, z_i)$:
    $$ z(x, y) = \frac{z_1(x_2 y_3 - x_3 y_2) + z_2(x_3 y_1 - x_1 y_3) + z_3(x_1 y_2 - x_2 y_1)}{x_1(y_2 - y_3) + x_2(y_3 - y_1) + x_3(y_1 - y_2)} $$
    (Assuming a triangular face; generalized for polygons via barycentric interpolation or plane equation $Ax + By + Cz + D = 0$ where $z = -(Ax + By + D)/C$) [G].

**Parameters:**
*   Scan line order: Not specified.
*   Sort stability: Not specified.

**Edge Cases and Pitfalls:**
*   **Visibility Cycles:** This method handles visibility cycles gracefully because a local visibility sort always exists for each pixel center, even if no global sort exists [S2, unofficial].
*   **Coherence:** The method takes partial advantage of coherence in $x$ (between adjacent pixels) but not in $y$ (between adjacent scan lines) [S1][S3, unofficial].
*   **Performance:** Since all faces are sorted, not just exterior ones, the method spends significantly more effort on per-pixel sorting, as exterior faces are typically a small fraction of total faces in most meshes [S2, unofficial].
*   **Hardware vs. Software:** Hardware proposals (e.g., R-Buffer) suggest scanning all polygons once and storing fragments in an unordered buffer for multiple depth comparison passes [S3, unofficial]. However, software implementations often repeat sorts from scratch independently at each pixel, which is less efficient than incremental updates [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_pixel_sort_general_variant(faces, screen_width, screen_height):
    # Initialize Y-Buckets
    y_buckets = [[] for _ in range(screen_height)]
    
    # Populate Y-Buckets: assign each face to the first scan line it crosses
    for face in faces:
        min_y, max_y = face.get_y_range()
        if min_y < screen_height and max_y >= 0:
            first_scan_line = max(0, min_y)
            y_buckets[first_scan_line].append(face)
            
    # Process each scan line
    for y in range(screen_height):
        active_list = []
        
        # Add new faces to active list
        for face in y_buckets[y]:
            active_list.append(face)
            
        # Remove faces that no longer overlap this scan line
        active_list = [f for f in active_list if f.overlaps_scan_line(y)]
        
        # Process each pixel in the scan line
        for x in range(screen_width):
            # Identify faces covering this pixel
            covering_faces = [f for f in active_list if f.covers_pixel(x, y)]
            
            # Sort faces by depth z at (x, y)
            # Optimization: If x > 0, use incremental sort based on previous pixel's order
            if x > 0:
                covering_faces = incremental_sort(covering_faces, previous_pixel_faces)
            else:
                covering_faces.sort(key=lambda f: f.get_z_at(x, y))
                
            # Composite faces back-to-front
            for face in covering_faces:
                composite_pixel(x, y, face)
                
            # Store current pixel's faces for incremental sort of next pixel
            previous_pixel_faces = covering_faces
            
    return rendered_image
```

## Subtopics

### [Mammen's Technique](1279-mammen-s-technique.md)
Mammen's Technique is a software algorithm to sort per-pixel transparent fragments in front of the front-most opaque one. It addresses the visibility ordering problem in object-order rendering where transparent surfaces must be composited from back to front [G]. Unlike incremental methods that exploit spatial coherence between adjacent pixels, Mammen's Technique performs independent, complete sorts for each pixel from scratch.

### [Nearly-Sorted Object-Space Visibility Ordering](3983-nearly-sorted-object-space-visibility-ordering.md)
Nearly-Sorted Object-Space Visibility Ordering (k-NSS) is a relaxation of exact visibility sorting where primitives are ordered such that rasterized fragments are at most $k$ positions out of order in the sorted sequence. This allows for the use of faster, approximate object-space sorting algorithms (such as centroid sorting or coarse BSP traversal) while deferring the guarantee of correct compositing to an exact sorting step during fragment processing. It differs from its parent [Per-Pixel Sorting (General Variant)](1061-per-pixel-sorting-general-variant.md) by performing the initial ordering in object space rather than per-pixel, and from siblings like [Mammen's Technique](1279-mammen-s-technique.md) by explicitly bounding the displacement error $k$ to enable linear-time correction.

### [Prioritized-Layered Projection (PLP)](9043-prioritized-layered-projection-plp.md)
PLP is an approximate, from-point, object-space visibility algorithm designed for large datasets that requires minimal preprocessing. It solves the problem of efficiently selecting a subset of geometry to render within a strict per-frame budget by using heuristics to estimate cell visibility likelihood. Unlike Teller et al.'s from-region algorithm, which requires extensive preprocessing and assumes axis-aligned cells, PLP makes no geometric assumptions and operates with very little setup overhead.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$ in the worst case, where $N$ is the number of faces and $P$ is the number of pixels, as each pixel may require sorting all overlapping faces. Incremental updates in $x$ reduce the constant factor but do not change the asymptotic complexity for dense scenes [S1][S3, unofficial].
*   **Memory Complexity:** Requires substantial memory proportional to the projected area, in pixels, of *all* faces in the mesh, not just the boundary [S3, unofficial]. This contrasts with methods like SXMPVO which only require memory proportional to the boundary [S3, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Handles visibility cycles gracefully; robust for non-convex meshes where global sorting fails [S2, unofficial].
*   **Limitations:** High memory usage; high computational cost due to sorting all faces (including interior ones); poor coherence in $y$ [S1][S2, unofficial][S3, unofficial].
*   **When to use:** When a global topological sort is impossible and memory is not a constraint; for scenes with complex visibility cycles.
*   **When not to use:** For large datasets with many interior faces; when memory bandwidth is limited; when global sorting is possible (use [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md) or [SXMPVO](944-exact-visibility-ordering-for-cell-projection-volume-renderi.md) instead).

## Background
The method was introduced by Wilhelms et al. [S1][S3, unofficial]. It is based on techniques similar to scan-line visible surface algorithms [S1][S3, unofficial]. Hardware proposals, such as the R-Buffer by Wittenbrink, aimed to implement this efficiently by scanning polygons once and storing fragments in a recirculating buffer for multiple depth comparison passes [S3, unofficial]. However, the R-Buffer hardware does not exist, and software implementations are less efficient than incremental methods [S3, unofficial]. The method is often compared to Mammen's Technique, which is similar but focuses on transparent fragments in front of opaque ones [S3, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://doi.org/10.1109/tvcg.2004.45, fetched 2026-10-25
[S2] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S3] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
