# Unstructured Grid Conversion from Hypertree

Also called: ToUnstructured

## Overview
This method converts a hypertree grid into an all-hexahedral unstructured grid [S1, unofficial]. It enables the use of hypertree data with visualization algorithms that require standard unstructured mesh inputs, bridging the gap between specialized AMR data structures and general-purpose visualization filters [S1, unofficial]. Unlike its sibling [Isosurface extraction on arbitrary polyhedral grids](951-isosurface-extraction-on-arbitrary-polyhedral-grids.md) which produces surface meshes, this method produces a volumetric hexahedral mesh. The resulting mesh is non-conforming, meaning adjacent cells do not necessarily share faces perfectly, which distinguishes it from conforming mesh generation techniques [S1, unofficial].

## How it works
**Inputs:** A hypertree grid data structure.
**Outputs:** An all-hexahedral unstructured grid.

The conversion process involves traversing the hypertree structure to extract the geometric representation of the cells. The algorithm maps the hierarchical nodes of the hypertree to the vertices and connectivity of a standard unstructured hexahedral mesh.

1.  **Traversal:** Iterate through the hypertree grid nodes. For each active cell, identify its geometric boundaries.
2.  **Vertex Generation:** Generate vertex coordinates for each hexahedral cell based on the hypertree's spatial definition.
3.  **Connectivity Assembly:** Construct the connectivity array (cell-to-vertex mapping) for the resulting unstructured grid. Each hypertree cell becomes a hexahedron in the output mesh.
4.  **Output:** Return the constructed unstructured grid object.

**Edge Cases and Pitfalls:**
*   **Non-Conforming Mesh:** The resulting unstructured mesh is not conforming [S1, unofficial]. Adjacent hexahedra may not share faces perfectly. This is a known limitation of the direct conversion approach [S1, unofficial].
*   **Visualization:** To visualize the output effectively, it is recommended to connect the output to a shrink filter, as the raw hexahedral mesh may appear dense or overlapping due to the non-conforming nature [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_hypertree_to_unstructured(hypertree_grid):
    """
    Converts a hypertree grid to an all-hexahedral unstructured grid.
    
    Args:
        hypertree_grid: The input hypertree grid data structure.
        
    Returns:
        unstructured_grid: An all-hexahedral unstructured grid object.
    """
    # Initialize the output unstructured grid
    unstructured_grid = UnstructuredGrid()
    
    # Iterate through all active cells in the hypertree
    for cell in hypertree_grid.active_cells():
        # Extract the 8 vertices of the hexahedral cell
        # The exact method depends on the hypertree's internal geometry representation
        vertices = cell.get_hexahedral_vertices()
        
        # Add the cell to the unstructured grid
        # Assuming a standard hexahedron has 8 vertices
        cell_id = unstructured_grid.add_cell(
            cell_type=HEXAHEDRON,
            vertex_ids=vertices
        )
        
    return unstructured_grid
```

## Subtopics

### Unstructured View Generation
This variant generates an unstructured view of the hypertree grid from strict nodes [S1, unofficial]. It can produce a complete or partial representation of the mesh [S1, unofficial]. This approach is used to connect hypertree grids with existing non-AMR visualization algorithms [S1, unofficial]. It allows for flexibility in the level of detail and connectivity required by downstream algorithms.

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Performance:** The direct conversion to a non-conforming mesh is considerably faster than methods that produce conforming meshes [S1, unofficial].
**Strengths:** Enables the use of hypertree grids with a wide range of existing visualization filters that expect unstructured mesh inputs [S1, unofficial].
**Limitations:** The output mesh is non-conforming, which may limit its suitability for post-processing tasks requiring perfect connectivity [S1, unofficial].

## Background
Hypertree grids are a specialized data structure for adaptive mesh refinement (AMR) simulations with rectilinear geometry [S1, unofficial]. While they offer optimal storage and traversal capabilities, the ecosystem of visualization filters for hypertree grids is less mature than that for standard unstructured meshes [S1, unofficial]. This method was developed to bridge this gap, allowing researchers to leverage the vast library of existing unstructured mesh visualization tools for hypertree data [S1, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
