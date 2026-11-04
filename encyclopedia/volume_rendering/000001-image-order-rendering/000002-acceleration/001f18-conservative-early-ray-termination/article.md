# Conservative Early Ray Termination

Also called: ERT in scan conversion phase, ERT-table scheme

## Overview
Conservative Early Ray Termination (CERT) is an optimization for the scan conversion phase of volume rendering that prunes the processing of cells by determining if a ray is already sufficiently opaque before the cell is rasterized [S1, unofficial]. It differs from standard Early Ray Termination (ERT) [1019-early-ray-termination.md], which typically requires sequential accumulation from the near plane, by using a lookup table (ERT-table) and subscreen-based aggregation to make visibility decisions based on partially accumulated opacity without per-pixel overhead [S1, unofficial]. This allows the technique to be applied during the parallel scan conversion phase where global ray state is not readily available [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (unstructured grid cells), camera parameters, screen resolution, opacity threshold $\tau$.
*   **Output:** Rendered image; reduction in scan conversion time by skipping cells.

**Data Structures:**
The core data structure is the **ERT-table**, which aggregates termination status per subscreen rather than per cell or per pixel.
*   **Screen Subdivision:** The screen is subdivided into $N$ subscreens $S$.
*   **ERT-table Entry:** For each subscreen $S$, the entry contains:
    1.  A one-bit flag indicating whether conditions C1 and C2 (defined below) are satisfied for that subscreen.
    2.  A value `max_term_Z`: the maximum depth value (rearmost depth) among all pixels in the subscreen that have a partially accumulated ray segment with opacity exceeding the threshold $\tau$.

**Parameters:**
*   **Opacity Threshold ($\tau$):** The threshold for "sufficiently high" opacity. In the source evaluation, $\tau = 0.9$ [S1, unofficial].

**Algorithm Steps:**
1.  **Subdivision:** Subdivide the screen into subscreens. Initialize the ERT-table entries for all subscreens.
2.  **Partial Composition:** During ray marching or segment processing, perform partial composition of ray segments. If a partially composed segment $s_{ij}$ has opacity $O_{ij} \ge \tau$, mark the corresponding pixel as "terminated" for the purpose of CERT.
3.  **Update ERT-table:** For each subscreen $S$:
    *   Check if there exists a nonempty ray-segment list for all pixels in $S$ (Condition C1).
    *   Check if there exists a partially composed ray segment with opacity $\ge \tau$ in each of these lists (Condition C2).
    *   If C1 and C2 are met, set the one-bit flag for $S$ to true.
    *   Calculate `max_term_Z` for $S$: the maximum depth $z$ among all pixels in $S$ that have a terminated segment. If no pixels are terminated, `max_term_Z` is undefined or effectively $-\infty$.
4.  **Cell Visibility Check:** For a cell $c$ to be skipped:
    *   Compute the projection area $R$ of cell $c$ on the screen.
    *   Identify the subscreens $S$ that overlap with $R$.
    *   Verify Condition C3: For every subscreen $S$ overlapping $R$, the ERT-table flag must be true, and the `max_term_Z` of that subscreen must be less than the depth of the nearest vertex of cell $c$ ($nearest_c$).
    *   Formally, the conditions to neglect scan conversion of cell $c$ are:
        *   (C1) There exists a nonempty ray-segment list for all pixels corresponding to the projection area $R$ for cell $c$.
        *   (C2) There exists a partially composed ray segment whose opacity is sufficiently high ($\ge \tau$) in each of these ray-segment lists.
        *   (C3) At least one of such ray-segments in each of the lists is located in front of the cell $c$ (i.e., its depth is less than the nearest vertex of $c$).
    *   If C1, C2, and C3 are satisfied for all relevant subscreens, terminate the scan conversion of cell $c$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
THRESHOLD = 0.9

# Data Structures
class ERTTableEntry:
    flag: bool  # True if C1 and C2 are satisfied for this subscreen
    max_term_z: float  # Rearmost depth of terminated segments in this subscreen

def update_ert_table(subscreens, ray_segment_lists):
    """
    Update the ERT-table based on current ray segment states.
    ray_segment_lists: dict mapping (subscreen_id, pixel_id) -> list of ray segments
    """
    for subscreen in subscreens:
        subscreen_id = subscreen.id
        pixels_in_subscreen = subscreen.get_pixels()
        
        all_pixels_have_segments = True
        all_pixels_have_terminated_segment = True
        max_term_z = -float('inf')
        
        for pixel in pixels_in_subscreen:
            segments = ray_segment_lists.get((subscreen_id, pixel.id), [])
            
            if not segments:
                all_pixels_have_segments = False
                break
            
            # Find the front-most segment with opacity >= THRESHOLD
            terminated_segment = None
            for seg in segments:
                if seg.opacity >= THRESHOLD:
                    # We want the front-most (smallest z) terminated segment
                    # But max_term_z is the rearmost (largest z) among ALL terminated segments
                    # Wait, the source says: "max_term_Z ... is the rearmost depth among all 
                    # the subscreen's per-pixel-front-most partially accumulated ray-segments 
                    # whose opacity exceed the threshold."
                    # This implies for each pixel, we find the front-most terminated segment,
                    # then take the max of those depths across the subscreen.
                    if terminated_segment is None or seg.z < terminated_segment.z:
                        terminated_segment = seg
            
            if terminated_segment is None:
                all_pixels_have_terminated_segment = False
                break
            
            if terminated_segment.z > max_term_z:
                max_term_z = terminated_segment.z
        
        if all_pixels_have_segments and all_pixels_have_terminated_segment:
            subscreen.ert_entry.flag = True
            subscreen.ert_entry.max_term_z = max_term_z
        else:
            subscreen.ert_entry.flag = False
            subscreen.ert_entry.max_term_z = -float('inf')

def should_skip_cell(cell, subscreens, ert_table):
    """
    Determine if a cell's scan conversion can be skipped.
    """
    projection_area = cell.get_projection_area()
    overlapping_subscreens = subscreens.get_overlapping_subscreens(projection_area)
    
    for subscreen in overlapping_subscreens:
        entry = ert_table[subscreen.id]
        
        # Check C1 and C2 via the flag
        if not entry.flag:
            return False
        
        # Check C3: max_term_z < nearest vertex of cell
        if entry.max_term_z >= cell.nearest_vertex.z:
            return False
            
    return True
```

**Edge Cases and Pitfalls:**
*   **Per-pixel Overhead:** A naive implementation checking per-pixel ray-segment lists incurs significant overhead. The ERT-table aggregation is essential to mitigate this [S1, unofficial].
*   **Conservatism:** CERT checks a sufficient but not necessary condition for termination. This approximation may reduce the effectiveness of ERT compared to an exact check, but it enables the technique in the scan conversion phase [S1, unofficial].
*   **Partial Composition:** The efficiency of ERT relies on partial composition increasing the opacity of ray segments, making it more likely to exceed the threshold [S1, unofficial].

## Subtopics

### Weak Sharing of ERT Information
A technique where Working Nodes (WNs) in a parallel system exchange their ERT-table information to increase the accuracy and completeness of termination decisions [S1, unofficial].
*   **Mechanism:** If WN$_a$ terminates scan conversion due to ERT, WN$_b$ (holding cells behind WN$_a$) can also terminate its scan conversion based on the shared ERT-table data [S1, unofficial].
*   **Parameter:** The frequency of ERT information exchange is set far below the frequency of ERT information updates at each WN to avoid interprocessor communication overhead [S1, unofficial].
*   **Performance:** In evaluations, sharing decreased scan conversion time by at most 43% for Viewpoint A and 50% for Viewpoint B. Effective parallel performance improvement was 7% for Viewpoint A and 30% for Viewpoint B [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The overhead of maintaining the ERT-table is reduced by subscreen aggregation compared to per-pixel checks. The exact complexity depends on the number of subscreens and pixels per subscreen.
*   **Measured Performance:** On a dataset of 307,565 unstructured tetrahedral cells with 62,475 vertices, CERT provided a speedup of 3.86 times in sequential execution on a single Working Node (WN) [S1, unofficial].
*   **Strengths:** Enables early termination in the scan conversion phase where global ray state is unavailable; reduces scan conversion time significantly.
*   **Limitations:** It is an approximation (conservative), so it may not terminate as many cells as an exact ERT check. The effectiveness depends on the opacity threshold and the distribution of opaque structures.
*   **When to Use:** When using a scan conversion-based volume rendering method (like Cell-Projection) and parallel execution overhead prevents per-pixel ray state tracking.

## Background
*   **Origin:** Introduced in the paper "Cell-projection Parallel Volume Rendering With Early Ray Termination" by Takeo Tomita et al. (2004) [S1, unofficial].
*   **Context:** Developed to address load imbalance in parallel volume rendering by applying ERT concepts to the scan conversion phase, which was previously difficult due to the lack of sequential ray accumulation state.
*   **Usage:** Applied in distributed parallel volume rendering systems, specifically within the CP-PVR (Cell-Projection Parallel Volume Rendering) framework.

## Sources
[S1] PDF Cell-projection Parallel Volume Rendering With Early Ray Termination, http://www.tomitaken.org/project/papers/ReVolver/2004/VIIP.pdf, fetched 2026-10-27 (unofficial)
