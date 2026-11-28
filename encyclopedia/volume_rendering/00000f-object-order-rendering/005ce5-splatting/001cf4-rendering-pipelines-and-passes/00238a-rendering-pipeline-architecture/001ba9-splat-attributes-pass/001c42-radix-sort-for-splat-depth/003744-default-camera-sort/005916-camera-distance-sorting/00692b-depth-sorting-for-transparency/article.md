# Depth Sorting for Transparency

Also called: Back-to-Front Sorting, Bin Sorting, Painter's Algorithm, Radix Sort for Depth, Sort-and-draw

## Overview
Depth sorting for transparency is a rendering technique that organizes gas (smoke) and obstacle data into bins based on their distance from the camera, then renders them in back-to-front order to ensure visually correct foreground/background interaction [S1, unofficial]. It solves the problem of incorrect alpha blending in OpenGL, which requires scene elements to be drawn from furthest to nearest to correctly composite transparent layers [S1, unofficial]. Unlike previous simulation methods that rendered all obstacle data before any gas information, this method renders them simultaneously by emptying bins for all element types in reverse distance order [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of scene elements (smoke particles/voxels and obstacle primitives like boxes or spheres), camera position $(C_x, C_y, C_z)$, minimum distance $D_0$, maximum distance $D_1$, and total number of bins $B$ [S1, unofficial].
*   **Outputs:** A sorted rendering order where elements are drawn back-to-front [S1, unofficial].

**Data Structures:**
*   **Depth Bins:** A vector of lists. Each list (bin) represents a range of distances from the camera [S1, unofficial].
*   **Element Representation:**
    *   **Smoke/Gas:** Stored as individual elements with position $(x, y, z)$ [S1, unofficial].
    *   **Box Obstacles:** Decomposed into six individual quad faces. For each quad, a representative position is calculated by averaging the coordinates of its four vertices:
        $$ x_{avg} = \frac{\sum_{i=1}^{4} x_i}{4}, \quad y_{avg} = \frac{\sum_{i=1}^{4} y_i}{4}, \quad z_{avg} = \frac{\sum_{i=1}^{4} z_i}{4} $$
        [S1, unofficial].
    *   **Other Obstacles:** Similar representative positions are calculated for other primitive types (e.g., spheres) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize Bins:** Create separate bin vectors for smoke and each obstacle type. The number of bins is $B$ [S1, unofficial].
2.  **Calculate Distance:** For each element, calculate the Euclidean distance to the camera:
    $$ d = \sqrt{(x - C_x)^2 + (y - C_y)^2 + (z - C_z)^2} $$
    For box quads, use $(x_{avg}, y_{avg}, z_{avg})$ instead of $(x, y, z)$ [S1, unofficial].
3.  **Assign to Bin:** Map the distance $d$ to a bin index $b$ using the formula:
    $$ b = \left\lfloor B \frac{d - D_0}{D_1 - D_0} \right\rfloor $$
    Clamp $b$ to the range $[0, B-1]$ if necessary [G].
4.  **Store Elements:** Place the element's data (coordinates, color, alpha) into the list corresponding to bin $b$ for its specific type [S1, unofficial].
5.  **Render:** Iterate through the bins from the greatest index ($B-1$) to the smallest (0). For each bin index, render all elements present in that bin for *all* element types (smoke and obstacles) simultaneously [S1, unofficial].

**Parameters:**
*   $B$: Total number of bins. Not given in the sources.
*   $D_0$: Minimum distance. Not given in the sources.
*   $D_1$: Maximum distance. Not given in the sources.
*   Environment Size: A cube with sides of 64 units was used in the reference implementation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_sort_and_render(elements, camera_pos, num_bins, min_dist, max_dist):
    """
    elements: dict mapping element_type to list of element objects.
              Each element has pos (x,y,z) and type ('smoke', 'box_face', etc.)
    camera_pos: tuple (Cx, Cy, Cz)
    num_bins: integer B
    min_dist: float D0
    max_dist: float D1
    """
    # Step 1: Initialize bins for each element type
    bins = {etype: [[] for _ in range(num_bins)] for etype in elements.keys()}
    
    # Step 2 & 3: Calculate distance and assign to bin
    for etype, elem_list in elements.items():
        for elem in elem_list:
            # Get representative position
            if etype == 'box_face':
                # Average of 4 vertices
                x_avg = sum(v.x for v in elem.vertices) / 4.0
                y_avg = sum(v.y for v in elem.vertices) / 4.0
                z_avg = sum(v.z for v in elem.vertices) / 4.0
                pos = (x_avg, y_avg, z_avg)
            else:
                pos = elem.pos
            
            # Calculate Euclidean distance
            dx = pos[0] - camera_pos[0]
            dy = pos[1] - camera_pos[1]
            dz = pos[2] - camera_pos[2]
            dist = (dx*dx + dy*dy + dz*dz) ** 0.5
            
            # Map to bin
            if max_dist == min_dist:
                bin_idx = 0
            else:
                bin_idx = int(num_bins * (dist - min_dist) / (max_dist - min_dist))
                # Clamp to valid range [G]
                bin_idx = max(0, min(num_bins - 1, bin_idx))
            
            bins[etype][bin_idx].append(elem)
            
    # Step 5: Render back-to-front
    for bin_idx in range(num_bins - 1, -1, -1):
        # Render all types in the same bin simultaneously
        for etype in bins.keys():
            for elem in bins[etype][bin_idx]:
                render_element(elem)

def render_element(elem):
    # Draw primitive to framebuffer with its stored color and alpha
    pass
```

## Subtopics

### Bin Sorting Transparency
Enhances standard bin sorting by generating separate bins for transparent and opaque primitives within each graphics processor [S4, unofficial]. Transparent primitives are rendered after all opaque ones, sorted back-to-front within their bins and composited simply [S4, unofficial]. For difficult geometric configurations like basket-weaves of transparent strips where Z-sorting fails, multiple Z values are stored per pixel to control compositing, and difficult primitives may be sent to renderers multiple times to ensure correct blending [S4, unofficial].

## Operating characteristics
*   **Complexity:** The bin/radix sort algorithm used has a complexity of $O(N)$ [S1, unofficial].
*   **Performance:** The method proved generally effective with no discernable errors in foreground/background interaction, despite changes in camera angle or distance [S1, unofficial].
*   **Strengths:** Ensures correct alpha blending for transparent smoke; allows simultaneous rendering of obstacles and gas; efficient $O(N)$ sorting [S1, unofficial].
*   **Limitations:** Requires decomposition of complex obstacles (e.g., boxes into quads) to calculate representative positions; precision depends on bin resolution $B$ and distance range $[D_0, D_1]$ [S1, unofficial].
*   **When to use:** When rendering transparent volumes (smoke, fire) interacting with opaque obstacles in a real-time OpenGL context [S1, unofficial].
*   **When not to use:** When exact depth precision within a bin is critical and $B$ is small; when obstacles cannot be easily decomposed into representative primitives [G].

## Background
This method was developed as part of a thesis on the efficient computation and rendering of smoke animation [S1, unofficial]. It was implemented to allow real-time simulation by reorganizing the scene for depth sorting, addressing the limitation that OpenGL requires back-to-front drawing order for correct transparency handling [S1, unofficial]. The implementation used a 64-unit cube environment and compared favorably to previous methods that rendered obstacles before gas, which caused visual errors [S1, unofficial].

## Sources
[S1] Efficient computation and rendering of smoke animation, http://hdl.handle.net/2346/16985, fetched 2026-10-26 (unofficial)
[S2] edgorman/Voxel-Engine, https://github.com/edgorman/Voxel-Engine, fetched 2026-10-27 (unofficial)
[S3] Volume Rendering, https://docslib.org/doc/4315965/volume-rendering, fetched 2026-10-28 (unofficial)
[S4] DTIC ADA208089: A Heterogeneous Multiprocessor Graphics System Using Processor-Enhanced Memories, https://archive.org/details/DTIC_ADA208089, fetched 2026-10-26 (unofficial)
