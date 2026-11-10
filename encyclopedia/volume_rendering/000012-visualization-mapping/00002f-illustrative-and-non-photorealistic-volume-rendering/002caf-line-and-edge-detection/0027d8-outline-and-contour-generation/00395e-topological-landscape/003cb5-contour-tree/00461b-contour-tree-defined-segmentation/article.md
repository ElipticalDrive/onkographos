# Contour-Tree-Defined Segmentation

Also called: Topological Zone Segmentation

## Overview
Contour-Tree-Defined Segmentation partitions a scalar volume into topologically distinct regions, called topological zones, by mapping the volume's geometry to its contour tree. It solves the problem of assigning transfer functions to complex, noisy, or multi-component volumes by allowing independent transfer functions for each zone, avoiding artifacts associated with tetrahedral interpolation. It differs from [Reeb Graph](9315-reeb-graph.md) by focusing on the full hierarchy of critical points and their arcs to define volumetric zones rather than just equivalence classes of contours, and from [Topological Zone Transfer Function Application](11654-topological-zone-transfer-function-application.md) by providing the segmentation mechanism itself.

## How it works
**Inputs:** A scalar field defined on a hexahedral grid (typically with trilinear interpolation).
**Outputs:** A segmentation of the volume into topological zones, each associated with a specific transfer function, and the final rendered image.

**Data Structures:**
1.  **Contour Tree:** A graph where nodes represent critical points (minima, maxima, saddles) and arcs represent the evolution of contour components between critical values.
2.  **Topological Zone:** A volume region swept out by a contour as the isovalue varies from the creation critical value to the destruction critical value of the corresponding arc.
3.  **Branch Decomposition:** A hierarchical representation of the contour tree used to simplify the structure and group arcs into larger units to avoid oversegmentation.

**Steps:**

1.  **Compute the Contour Tree:**
    Compute the contour tree of the underlying trilinear interpolant. This is done using a variation of the method by Pascucci and Cole-McLaughlin [S1].
    *   Construct a widget in each hexahedral cell. The mesh edges are augmented by edges connecting body saddles, face saddles, and vertices [S1].
    *   Apply the graph-theoretic algorithm to the union of cellwise join and split trees [S1].
    *   *Note:* Any graph accurately representing the topology of the trilinear interpolant may be substituted [S1].

2.  **Simplify the Contour Tree:**
    Simplify the computed contour tree to remove symbolic perturbation artifacts [S1]. This step ensures robustness against numerical noise.

3.  **Assign Transfer Functions:**
    Assign a transfer function to each topological zone (each arc in the simplified tree). This allows independent control over the appearance of different topological features, including noise [S1].

4.  **Hierarchical Storage (Optional but recommended for large data):**
    For medium to large datasets, store the hierarchy of the simplified contour tree using branch decomposition [S1].
    *   Branch decomposition concatenates sequences of arcs into single units, unifying their topological zones.
    *   This avoids "oversegmentation" by considering that segmentation is only necessary if regions overlap in value range [S1].
    *   It permits user interaction on a coarse, simplified tree with propagation of transfer functions [S1].

5.  **Rendering and Opacity Lookup:**
    Apply the trilinear interpolant over hexahedral cells to determine opacity.
    *   For cells spanning multiple topological zones, find the correct opacity by looking up values directly in the relevant transfer functions [S1].
    *   *Pitfall:* Do not interpolate opacities over tetrahedra, as this leads to artifacts in cells spanning multiple zones [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_tree_segmentation(volume_data, grid):
    # 1. Compute Contour Tree
    # Construct widget in each cube: augment mesh edges with edges 
    # between body saddles, face saddles, and vertices
    widget_graph = build_hexahedral_widgets(grid)
    
    # Compute contour tree using variation of Pascucci and Cole-McLaughlin
    # applied to union of cellwise join and split trees
    contour_tree = compute_contour_tree(widget_graph)
    
    # 2. Simplify Contour Tree
    # Remove symbolic perturbation
    simplified_tree = simplify_contour_tree(contour_tree)
    
    # 3. Hierarchical Storage (for large datasets)
    # Store hierarchy using branch decomposition
    # This concatenates arcs into units to avoid oversegmentation
    branch_decomposition = compute_branch_decomposition(simplified_tree)
    
    # 4. Assign Transfer Functions
    # Assign independent transfer function to each topological zone (arc)
    zone_transfer_functions = assign_transfer_functions(simplified_tree)
    
    # 5. Render
    for each hexahedral_cell in grid:
        # Determine which topological zones this cell spans
        zones_in_cell = get_zones_spanning_cell(cell, simplified_tree)
        
        # For each point in the cell, determine opacity
        # Use trilinear interpolation over hexahedral cells
        # Look up opacity directly in the relevant transfer function
        # Do NOT interpolate over tetrahedra
        opacity = lookup_opacity_trilinear(cell, zones_in_cell, zone_transfer_functions)
        
        render_pixel(opacity)
```

## Subtopics

### Contour-Based Segmentation for FT Specification
A technique using contours to separate portions of a volume before transfer function (FT) specification, allowing manual specification on the isolated region [S2, unofficial].
*   **Definition:** A user-driven method where contours are drawn to isolate specific regions, which are then assigned transfer functions manually [S2, unofficial].
*   **Steps:**
    1.  User draws contours to separate portions of the volume [S2, unofficial].
    2.  The desired region is isolated [S2, unofficial].
    3.  User manually specifies the transfer function for the isolated region [S2, unofficial].
*   **Comparison:** This is a manual, interactive approach, whereas Contour-Tree-Defined Segmentation is an automatic, topology-driven method that assigns transfer functions based on the contour tree structure [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Computing the contour tree is the dominant cost. The method uses a graph-theoretic algorithm on the union of cellwise join and split trees [S1]. Branch decomposition adds overhead but simplifies interaction [S1].
*   **Memory Complexity:** Requires storing the contour tree and branch decomposition. For large datasets, the hierarchical representation helps manage complexity [S1].
*   **Strengths:**
    *   Allows independent transfer functions for individual topological zones [S1].
    *   Uses trilinear interpolation over hexahedral cells, avoiding tetrahedral artifacts [S1].
    *   Looks up opacities directly in the transfer function, reducing artifacts in cells spanning multiple zones [S1].
    *   Branch decomposition avoids oversegmentation and supports coarse interaction [S1].
*   **Limitations:**
    *   Topological complexity can be high, especially in noisy or physically sampled data [S1].
    *   Requires careful handling of symbolic perturbation [S1].
*   **When to use:** When the volume has complex topological structures that need distinct visual treatment, and when tetrahedral interpolation artifacts are a concern.
*   **When not to use:** For simple volumes where a single transfer function suffices, or when computational resources are extremely limited and contour tree computation is prohibitive.

## Background
The method builds on the work of Takeshima et al. [S1], who used topological zones for segmentation. It extends this by allowing independent transfer functions, using trilinear interpolation, and improving topological simplification [S1]. The contour tree computation variation is based on Pascucci and Cole-McLaughlin [S1], who applied graph-theoretic algorithms to the union of cellwise join and split trees [S1]. The concept of topological zones follows Cox et al. [S1]. Branch decomposition for hierarchical representation is described by Pascucci et al. [S1].

## Sources
[S1] Topology-Controlled Volume Rendering, https://doi.org/10.1109/tvcg.2007.47, fetched 2026-10-27
[S2] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
