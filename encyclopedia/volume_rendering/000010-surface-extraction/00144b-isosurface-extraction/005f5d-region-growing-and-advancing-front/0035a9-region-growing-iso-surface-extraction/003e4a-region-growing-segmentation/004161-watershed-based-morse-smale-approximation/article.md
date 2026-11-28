# Watershed-based Morse-Smale Approximation

Also called: MS complex segmentation, Morse-Smale Complex Segmentation, Morse-Smale complex approximation, Morse-Smale complex approximation via watershed segmentation, Watershed segmentation for Morse-Smale complex, flow-based analysis for protein surface segmentation

## Overview
This method approximates the discrete Morse-Smale complex (MS complex) of a scalar field by intersecting watershed segmentations of the field and its complement, thereby avoiding the computationally expensive explicit extraction of critical points [S1, unofficial]. It partitions a surface into monotonic regions based on a scalar function (e.g., atomic density), enabling the segmentation of features like protrusions and cavities in protein surfaces [S2, unofficial]. Unlike sibling methods that may rely on explicit critical point detection or graph cuts, this approach leverages the topological duality between a function and its inverse to define the complex structure implicitly through flow-based segmentation [S1, unofficial].

## How it works
**Inputs:**
1. A normalized discrete scalar field $f$ defined on a domain (e.g., a tessellated boundary mesh of a protein surface) [S1, unofficial].
2. The domain surface geometry (implicit in the field definition) [S2, unofficial].

**Outputs:**
1. An approximation of the discrete Morse-Smale complex, represented as a partition of the surface into monotonic regions [S1, unofficial].
2. Boundaries of peaks and valleys (protrusions and cavities) derived from the complex [S2, unofficial].

**Data Structures:**
*   **Scalar Field $f$:** A discrete function mapping vertices or cells to scalar values (e.g., atomic density).
*   **Watershed Segmentation:** A partition of the domain into basins of attraction for local minima (for $f$) or maxima (for $1-f$).
*   **MS Complex Cell:** A set of points flowing towards a fixed minimum/maximum pair via steepest descent/ascent paths [S2, unofficial].

**Steps:**
1.  **Pre-processing:** Ensure $f$ satisfies the conditions of a Morse function. This is necessary because the approximation is valid only if $f$ is a Morse function, which does not generally hold for discrete scalar fields due to degenerate critical points or flat regions [S1, unofficial]. Suitable pre-processing techniques (e.g., perturbation) must be applied to enforce this condition [S1, unofficial]. [G] In practice, this often involves adding small random noise to break ties and ensure distinct critical points.
2.  **Watershed on $f$:** Perform watershed segmentation on the normalized discrete scalar field $f$. This identifies basins of attraction for local minima, effectively segmenting the surface based on steepest-descent flow towards minima [S1, unofficial].
3.  **Watershed on $1-f$:** Construct the inverted field $g = 1 - f$ (or $-f$). Perform watershed segmentation on this inverted field. This identifies basins of attraction for local maxima of $f$ (minima of $g$), segmenting the surface based on steepest-ascent flow towards maxima [S1, unofficial].
4.  **Intersection:** Compute the intersection of the two resulting watershed segmentations. The intersection of the basins from $f$ and $1-f$ corresponds to an approximation of the discrete Morse-Smale complex [S1, unofficial]. Each cell in the resulting complex consists of points that flow towards a fixed minimum/maximum pair via paths of steepest descent/ascent [S2, unofficial].
5.  **Boundary Extraction:** Compute paths connecting saddles within each cell of the MS complex. These paths collectively form the boundaries of peaks and valleys of the surface [S2, unofficial].
6.  **Segmentation:** Use the boundaries formed by the saddle paths to define the final segmentation of features (e.g., protrusions and cavities) [S2, unofficial].

**Formulas:**
*   **Descending Manifold:** The descending manifold of a maximum $m$ consists of all points $p$ such that the steepest-ascent path beginning at $p$ terminates at $m$ [S2, unofficial].
*   **Ascending Manifold:** Steepest-descent paths beginning at all points associated with a minimum terminate at the minimum [S2, unofficial].
*   **MS Cell:** A cell in the MS complex partition consists of points that flow towards a fixed minimum/maximum pair via paths of steepest descent/ascent [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def watershed_ms_approximation(surface_mesh, scalar_field):
    """
    Approximates the Morse-Smale complex using watershed segmentation.
    
    Args:
        surface_mesh: The tessellated boundary mesh (vertices, edges, faces).
        scalar_field: Discrete scalar values f(v) for each vertex v.
        
    Returns:
        ms_complex: A partition of the surface into monotonic regions.
        boundaries: List of paths forming boundaries of peaks/valleys.
    """
    # Step 1: Pre-processing to ensure Morse function property
    # [G] Add small random perturbation to scalar_field to break degeneracies
    # and ensure distinct critical points, as discrete fields rarely satisfy
    # the Morse condition naturally.
    perturbed_field = scalar_field + random_noise(scale=epsilon)
    
    # Step 2: Watershed segmentation on f
    # Identify basins of attraction for local minima
    basins_min = watershed_segmentation(perturbed_field, surface_mesh)
    
    # Step 3: Watershed segmentation on 1-f (inverted field)
    # Identify basins of attraction for local maxima
    inverted_field = 1.0 - perturbed_field
    basins_max = watershed_segmentation(inverted_field, surface_mesh)
    
    # Step 4: Intersection of watershed segmentations
    # The intersection approximates the discrete Morse-Smale complex
    # Each region in the intersection corresponds to a cell in the MS complex
    ms_complex = intersect_segmentations(basins_min, basins_max)
    
    # Step 5: Compute saddle paths within each cell
    # [G] Identify saddle points (critical points of index 1)
    # Trace steepest ascent/descent paths connecting saddles within each cell
    boundaries = []
    for cell in ms_complex:
        saddle_paths = compute_saddle_paths(cell, surface_mesh, perturbed_field)
        boundaries.extend(saddle_paths)
        
    return ms_complex, boundaries

def watershed_segmentation(field, mesh):
    """
    Performs watershed segmentation on a scalar field over a mesh.
    Returns a map from each vertex to its basin ID (local minimum index).
    """
    # [G] Standard implementation:
    # 1. Identify local minima in the field.
    # 2. Initialize a priority queue with all vertices, keyed by field value.
    # 3. Process vertices in increasing order of field value.
    # 4. For each vertex, assign it to the basin of the neighbor with the 
    #    lowest field value that has already been processed.
    # 5. If a vertex connects to multiple basins, it becomes a boundary point 
    #    (or is assigned to one based on tie-breaking rules).
    # This effectively simulates steepest-descent flow.
    pass

def compute_saddle_paths(cell, mesh, field):
    """
    Computes paths connecting saddle points within a specific MS complex cell.
    """
    # [G] Identify saddle points within the cell.
    # Trace paths along the gradient flow from saddles to the cell's 
    # associated minimum and maximum.
    # The union of these paths forms the boundary of the cell.
    pass
```

**Parameters:**
*   **Noise Scale ($\epsilon$):** Small value for pre-processing perturbation. Not given in the sources.
*   **Field Normalization:** The field $f$ should be normalized, typically to $[0, 1]$ or similar, before inversion $1-f$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Morse Functions:** The approximation is invalid if $f$ is not a Morse function. Discrete scalar fields often have degenerate critical points (e.g., flat regions, multiple minima with equal values). Pre-processing is required [S1, unofficial].
*   **Closed Trajectories:** Detection and extraction of closed trajectories in flow fields is a major challenge [S1, unofficial].
*   **Boundary Approximation:** Separation and attachment lines are commonly extracted on cell-wise projections of the field on the tessellated boundary mesh. Lines are approximated by segments created from intersections of these lines with edges of boundary mesh elements [S1, unofficial].

## Subtopics

### Valley Cell Construction
Focuses on constructing specific cells (valleys) within the MS complex. It differs by targeting specific topological features rather than the full complex intersection. [Valley Cell Construction](9747-valley-cell-construction.md)

### Graph-based Isosurface Segmentation (Min-Cut)
Uses graph cuts to segment isosurfaces. It differs by optimizing a global energy function rather than relying on flow-based watershed intersection. [Graph-based Isosurface Segmentation (Min-Cut)](9748-graph-based-isosurface-segmentation-min-cut.md)

### Restricted Region Growing
Grows regions under strict constraints. It differs by not using the dual watershed approach for topological approximation. [Restricted Region Growing](9750-restricted-region-growing.md)

### Global/Local Segmentation
Combines global and local segmentation strategies. It differs by potentially using explicit critical point extraction or hybrid methods. [Global/Local Segmentation](10490-global-local-segmentation.md)

### Watershed Segmentation
The base algorithm used in this method. It differs by not intersecting with the inverted field to form the MS complex. [Watershed Segmentation](13875-watershed-segmentation.md)

### Adaptive Region Growing Cost Functions
Uses adaptive costs for region growing. It differs by not relying on the topological duality of the MS complex. [Adaptive Region Growing Cost Functions](14344-adaptive-region-growing-cost-functions.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources. However, it avoids explicit critical point extraction, which is described as "computationally expensive" [S1, unofficial]. The complexity is dominated by the watershed segmentation steps.
*   **Memory Complexity:** Not given in the sources. Requires storage for the scalar field, watershed labels, and the resulting complex structure.
*   **Strengths:** Avoids explicit critical point extraction; provides a natural segmentation of features like protrusions and cavities based on topological flow; generates multi-level detail representations by canceling critical point pairs [S2, unofficial].
*   **Limitations:** Requires pre-processing to ensure the Morse function property; sensitive to noise and discretization artifacts; detection of closed trajectories is challenging [S1, unofficial].
*   **When to Use:** When segmenting protein surfaces or similar scalar fields where topological features (peaks/valleys) correspond to meaningful structures (protrusions/cavities) and explicit critical point extraction is too expensive or unreliable.
*   **When Not to Use:** When the scalar field is not a good approximation of a Morse function and pre-processing is insufficient; when closed trajectories are prevalent and must be accurately captured.

## Background
The method is based on the construction of the Morse-Smale complex (MS complex), a topological data structure that partitions a surface into monotonic regions [S2, unofficial]. The MS complex is derived from the critical points (maxima, minima, saddles) of a scalar function and their associated ascending and descending manifolds [S2, unofficial]. The watershed-based approximation was proposed to avoid the challenges of explicit critical point extraction, particularly in scattered or discrete data sets [S1, unofficial]. The technique has been applied to the visualization and analysis of protein surfaces, where atomic density functions are used to identify functional features like cavities and protrusions [S2, unofficial]. Efficient algorithms for computing the MS complex and generating multi-level detail representations by canceling critical point pairs have also been developed [S2, unofficial].

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
[S2] Interactive Processing and Visualization of Image Data forBiomedical and Life Science Applications, https://www.osti.gov/biblio/910338, fetched 2026-11-01 (unofficial)
