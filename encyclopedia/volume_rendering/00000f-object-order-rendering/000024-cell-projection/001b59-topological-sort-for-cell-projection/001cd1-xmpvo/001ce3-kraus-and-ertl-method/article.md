# Kraus and Ertl Method
## Overview
The Kraus and Ertl method handles visibility cycles in convex meshes during cell projection volume rendering [S1, S2, unofficial]. It addresses cases where a global topological sort of cells is impossible due to cyclic visibility dependencies, while a local sort still exists for each pixel center [S1, S2, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Projected images of convex, non-overlapping cells on the screen [S1, S2, unofficial].
- Output: A correctly sorted and composited image that resolves visibility cycles [S1, S2, unofficial].
**Data Structures:**
The method relies on the output of the basic MPVO sorting algorithm to identify cycle pixels [S2, unofficial]. Not given in the sources [S1, S2, unofficial].
**Steps:**
1. Apply the basic MPVO sorting algorithm to detect pixels where the cell ordering forms a cycle [S1, S2, unofficial].
2. Isolate these pixels and their associated cells [S1, S2, unofficial].
3. Use special hardware compositing methods to scan-convert each isolated cycle [S1, S2, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kraus_and_ertl_method(mpvo_sort_results):
    output_buffer = {}
    cycle_pixels = []
    # Step 1 & 2: Detect and isolate cycles using MPVO output
    for pixel, sort_data in mpvo_sort_results.items():
        if sort_data.has_cycle:
            cycle_pixels.append(pixel)
    # Step 3: Hardware compositing for each cycle
    for pixel in cycle_pixels:
        cycle_cells = sort_data.cycle_cells
        # [G] The source does not specify the exact hardware compositing
        # or scan-conversion algorithm used to resolve the cycle.
        output_buffer[pixel] = cycle_cells
    return output_buffer
```
**Parameters:**
Not given in the sources [S1, S2, unofficial].
**Edge Cases and Pitfalls:**
- Rendering costs per cycle increase quadratically in the number of cells involved in the cycle [S1, S2, unofficial].
- The method is designed for convex meshes [S1, S2, unofficial].
## Operating characteristics
- **Time Complexity:** Rendering costs per cycle increase quadratically in the number of cells involved in the cycle [S1, S2, unofficial].
- **Memory Complexity:** Not given in the sources [S1, S2, unofficial].
- **Strengths:** Handles visibility cycles gracefully for convex non-overlapping cells [S1, S2, unofficial]. For convex non-overlapping cells, there is always a visibility sort for each pixel center even if there is no global visibility sort of the cells [S1, S2, unofficial].
- **Limitations:** Quadratic cost per cycle [S1, S2, unofficial]. Limited to convex meshes in its original form [S1, S2, unofficial].
- **When to use:** When rendering convex meshes with potential visibility cycles where standard MPVO fails [S1, unofficial].
- **When not to use:** When rendering non-convex meshes without extension [S1, unofficial].
## Background
The method was introduced by Kraus and Ertl [S1, S2, unofficial]. The method could be extended to nonconvex meshes using the SXMPVO algorithm [S1, S2, unofficial].
## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S2] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
