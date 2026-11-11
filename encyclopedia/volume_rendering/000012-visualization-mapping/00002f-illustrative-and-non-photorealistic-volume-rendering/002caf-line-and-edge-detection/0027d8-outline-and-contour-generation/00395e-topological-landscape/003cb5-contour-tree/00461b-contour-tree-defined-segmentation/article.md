# Contour-Tree-Defined Segmentation
Also called: Topological Zone Segmentation

## Overview
Contour-Tree-Defined Segmentation partitions a scalar volume into topological zones, where each zone corresponds to an arc in the contour tree of the underlying trilinear interpolant. It solves the problem of assigning independent transfer functions to distinct topological structures, allowing for the isolation of specific features based on their creation and destruction isovalue ranges. It sets itself apart from [Reeb Graph](9315-reeb-graph.md) by focusing on the full hierarchy of splits and joins in the contour tree rather than just the connectivity of level sets, and from [Contour-Based Segmentation for FT Specification](11723-contour-based-segmentation-for-ft-specification.md) by automating the segmentation via topological computation rather than requiring manual 2D sketching.

## How it works
**Inputs:** A scalar field defined on a regular grid of hexahedral cells, typically with trilinear interpolation.
**Outputs:** A segmentation of the volume into topological zones, each associated with a specific transfer function, and the final rendered image.

**Data Structures:**
1.  **Grid:** Hexahedral cells with scalar values at vertices.
2.  **Widget:** A local graph structure within each cube. Edges include the cube's mesh edges, plus augmented edges connecting body saddles, face saddles, and vertices.
3.  **Contour Tree:** A graph where nodes represent critical points (minima, maxima, saddles) and arcs represent intervals between critical values.
4.  **Simplified Tree:** The contour tree after simplification to remove noise (symbolic perturbation).
5.  **Branch Decomposition:** A hierarchical representation of the simplified contour tree to manage complexity and avoid oversegmentation.
6.  **Transfer Functions:** A mapping from each topological zone (arc in the simplified tree) to opacity/color parameters.

**Steps:**

1.  **Compute the Contour Tree:**
    Compute the contour tree of the trilinear interpolant. This is done using a variation of the method by Pascucci and Cole-McLaughlin [S1].
    *   Construct the widget in each cube as described above.
    *   Apply the graph-theoretic algorithm to the union of cellwise join and split trees derived from these widgets [S1].
    *   *Note:* Any graph accurately representing the topology of the trilinear interpolant may be substituted [S1].

2.  **Simplify the Contour Tree:**
    Apply topological simplification to the computed contour tree to remove symbolic perturbation (noise) [S1]. This step merges insignificant arcs, reducing the number of topological zones.

3.  **Assign Transfer Functions:**
    Assign a specific transfer function to each remaining arc (topological zone) in the simplified contour tree [S1].
    *   The hierarchy is stored using branch decomposition to allow coarse interactions and propagation of transfer function changes [S1].
    *   Branch decomposition concatenates sequences of arcs into single units if their regions do not overlap in value range, preventing oversegmentation [S1].

4.  **Volume Rendering with Zone-Specific Opacity:**
    For each ray cast through the volume:
    *   Determine which topological zone the current sample point belongs to.
    *   Look up the opacity directly from the transfer function assigned to that specific zone [S1].
    *   Apply the trilinear interpolant over the hexahedral cells to determine the scalar value at the sample point [S1].
    *   *Crucial Detail:* For cells spanning multiple topological zones, the method calculates the correct opacity by identifying the specific zone active at the sample point's scalar value, rather than interpolating opacities across tetrahedra [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_tree_segmentation_render(grid, transfer_functions_map):
    """
    grid: 3D array of scalar values at vertices.
    transfer_functions_map: dict mapping (zone_id) -> transfer_function_object.
    """
    # Step 1: Compute Contour Tree
    # Construct widgets for each hexahedral cell
    widgets = [construct_widget(cell) for cell in grid.cells]
    
    # Compute join and split trees for each cell
    join_trees = compute_join_trees(widgets)
    split_trees = compute_split_trees(widgets)
    
    # Union and compute global contour tree
    contour_tree = compute_contour_tree_from_union(join_trees, split_trees)
    
    # Step 2: Simplify
    simplified_tree = simplify_contour_tree(contour_tree)
    
    # Step 3: Assign Transfer Functions (Pre-computed or user-defined)
    # Branch decomposition is used to manage hierarchy
    branch_decomposition = compute_branch_decomposition(simplified_tree)
    
    # Map each arc in simplified_tree to a zone ID and its TF
    zone_to_tf = assign_transfer_functions(branch_decomposition, transfer_functions_map)
    
    # Step 4: Render
    image = empty_image()
    for ray in rays:
        accumulated_color = 0
        accumulated_opacity = 0
        for sample in ray.samples:
            scalar_value = trilinear_interpolate(grid, sample.position)
            
            # Determine which topological zone this scalar value belongs to
            # The zone is defined by the arc in the simplified tree 
            # spanning the critical values that create/destroy the contour at this value.
            zone_id = get_zone_id_for_scalar(simplified_tree, scalar_value)
            
            # Get the transfer function for this zone
            tf = zone_to_tf[zone_id]
            
            # Look up opacity directly from the zone's TF
            opacity = tf.get_opacity(scalar_value)
            color = tf.get_color(scalar_value)
            
            # Standard volume rendering accumulation
            # Note: No tetrahedral interpolation of opacity is performed
            weight = opacity * (1 - accumulated_opacity)
            accumulated_color += color * weight
            accumulated_opacity += weight
            
            if accumulated_opacity > 1.0:
                break
                
        image[ray.id] = accumulated_color
    return image

def construct_widget(cell):
    """
    Augments the cube's mesh edges with edges between body saddles, 
    face saddles, and vertices.
    """
    widget_edges = list(cell.mesh_edges)
    saddles = find_saddles_in_cell(cell)
    vertices = cell.vertices
    # Add edges between saddles and vertices as per [S1]
    for saddle in saddles:
        for vertex in vertices:
            widget_edges.append((saddle, vertex))
    return Graph(edges=widget_edges)
```

**Edge Cases and Pitfalls:**
*   **Symbolic Perturbation:** Without simplification, noise creates excessive topological zones, leading to oversegmentation and performance issues. Simplification is mandatory for medium to large datasets [S1].
*   **Cells Spanning Multiple Zones:** If a hexahedral cell contains critical points, it spans multiple topological zones. The method must correctly identify which zone is active at the specific sample point's scalar value. Incorrect handling leads to artifacts [S1].
*   **Trilinear Interpolation:** The method relies on the topology of the trilinear interpolant, not a tetrahedral decomposition. Using tetrahedral interpolation for opacity lookup is explicitly avoided as it causes artifacts [S1].

## Subtopics

### [Contour-Based Segmentation for FT Specification](11723-contour-based-segmentation-for-ft-specification.md)
Uses user-drawn 2D contours to isolate volume portions before transfer function specification. It differs by replacing topological tree traversal with direct geometric separation via sketches.

## Operating characteristics
*   **Complexity:** The complexity is dominated by the contour tree construction and simplification. For large datasets, branch decomposition is used to manage the hierarchy, allowing coarse interactions [S1].
*   **Performance:** The method avoids the artifacts associated with tetrahedral interpolation of opacities [S1]. It allows independent transfer functions for individual zones, which can increase the complexity of the rendering pipeline but provides finer control [S1].
*   **Strengths:** Topologically distinct regions are segmented correctly. It handles the trilinear interpolant directly, avoiding tetrahedral artifacts. Branch decomposition helps manage complexity and prevents oversegmentation [S1].
*   **Limitations:** Requires computation of the contour tree, which can be expensive for very large datasets without simplification. The method assumes the scalar field is well-behaved enough for the trilinear interpolant's topology to be meaningful.
*   **When to use:** When the user wants to isolate and style specific topological features (e.g., separate noise from signal, or distinct anatomical structures) based on their topological persistence.
*   **When not to use:** When real-time performance is critical and contour tree computation is too slow, or when the scalar field is too noisy to simplify effectively.

## Background
The method was introduced in "Topology-Controlled Volume Rendering" [S1]. It builds on the concept of topological zones defined by Cox et al. [25] and extends the work of Takeshima et al. [10] by allowing independent transfer functions for individual zones and using trilinear interpolation over hexahedral cells. The contour tree computation uses variations of methods by Pascucci and Cole-McLaughlin [19, 18], and the hierarchical representation uses branch decomposition by Pascucci et al. [35].

## Sources
[S1] Topology-Controlled Volume Rendering, https://doi.org/10.1109/tvcg.2007.47, fetched 2026-10-27
