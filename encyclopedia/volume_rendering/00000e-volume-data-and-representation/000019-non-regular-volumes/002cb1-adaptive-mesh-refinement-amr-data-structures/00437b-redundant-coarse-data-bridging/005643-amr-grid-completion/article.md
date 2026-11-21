# AMR Grid Completion

Also called: AMR Generation from Wavelets, Adaptive Wavelet Grid to AMR Conversion, Completing Wavelet Based Grid to AMR, Wavelet to AMR conversion

## Overview
AMR Grid Completion is a preprocessing technique that converts an adaptive wavelet grid into a traditional node-centered Adaptive Mesh Refinement (AMR) grid by adding supplementary nodes, allowing the use of existing AMR visualization software [S1][S2, unofficial]. Unlike direct summation in wavelet space, this approach deals with actual function values on a traditional node-centered AMR grid [S1][S2, unofficial]. It is chosen when one needs to leverage established AMR rendering pipelines (e.g., ParaView, VisIt) rather than implementing custom wavelet-based renderers [S1][S2, unofficial].

## How it works
**Inputs:**
*   An adaptive wavelet grid representing a scalar field.
*   Scaling function $\phi(\xi)$ and wavelet function $\psi(\xi)$ definitions.
*   Cutoff limits for the basis functions: $\xi_0^\phi = 1$ and $\xi_0^\psi = 2$ [S1][S2, unofficial].

**Outputs:**
*   A node-centered AMR grid with actual function values at all nodes.
*   An unstructured tetrahedral mesh VTK file containing the AMR grid data [S1][S2, unofficial].

**Data Structures:**
*   **Wavelet Grid:** Defined by node locations and coefficients. Nodes are located at specific coordinates within the domain.
*   **AMR Grid:** A hierarchical set of cells (cubes in 3D, squares in 2D) with nodes at vertices. Cells are refined based on the "bisection" rules derived from wavelet node locations.
*   **VTK File:** Stores the unstructured mesh. Each cubic cell of the AMR grid is decomposed into six tetrahedra for rendering [S2, unofficial].

**Steps:**

1.  **Initialize AMR Grid:** Start with the coarsest level of the domain.
2.  **Induce Bisections:** Analyze the locations of wavelet nodes to determine cell refinement:
    *   If a wavelet node is located **inside** a cell, induce a "bisection" of that cell up to the level of that node [S1][S2, unofficial].
    *   If a wavelet node is located at a cell's **face or edge**, induce a "bisection" of all adjacent cells [S1][S2, unofficial].
    *   *Note:* The term "bisection" here refers to the refinement process required to accommodate the node's position in the hierarchical grid structure [S1][S2, unofficial].
3.  **Construct Node-Centered AMR Grid:** Create the final node-centered AMR grid structure based on the induced bisections. This involves adding supplementary nodes where necessary to maintain the node-centered topology [S1][S2, unofficial].
4.  **Inverse Wavelet Transform:** Perform an inverse wavelet transform on all nodes of the obtained AMR grid to compute the actual function values at these nodes [S1][S2, unofficial].
    *   This step converts the wavelet coefficients into physical values at the AMR grid nodes [S1][S2, unofficial].
5.  **Export to VTK:** Write the obtained AMR grid data in the form of an unstructured (tetrahedral) mesh VTK file [S1][S2, unofficial].
    *   Each cubic cell is decomposed into six tetrahedra [S2, unofficial].
    *   The resulting file is readable by ParaView, VisIt, and VTK [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def amr_grid_completion(wavelet_grid, scaling_func, wavelet_func, cutoff_phi=1, cutoff_psi=2):
    """
    Converts an adaptive wavelet grid to a node-centered AMR grid.
    
    Args:
        wavelet_grid: List of wavelet nodes with coordinates and coefficients.
        scaling_func: Function phi(xi).
        wavelet_func: Function psi(xi).
        cutoff_phi: Cutoff limit for scaling function (default 1).
        cutoff_psi: Cutoff limit for wavelet function (default 2).
        
    Returns:
        amr_grid: Node-centered AMR grid with function values.
        vtk_data: VTK file data structure.
    """
    # Step 1: Initialize empty AMR grid structure
    amr_grid = initialize_amr_grid()
    
    # Step 2: Induce bisections based on wavelet node locations
    for node in wavelet_grid:
        cell = find_containing_cell(amr_grid, node.coordinates)
        if cell is None:
            continue
            
        if is_inside_cell(node.coordinates, cell):
            # Wavelet node inside a cell induces bisection of that cell
            refine_cell(amr_grid, cell, node.level)
        elif is_on_face_or_edge(node.coordinates, cell):
            # Wavelet node at face/edge induces bisection of all adjacent cells
            adjacent_cells = get_adjacent_cells(cell)
            for adj_cell in adjacent_cells:
                refine_cell(amr_grid, adj_cell, node.level)
                
    # Step 3: Add supplementary nodes to ensure node-centered topology
    add_supplementary_nodes(amr_grid)
    
    # Step 4: Perform inverse wavelet transform to get function values
    for node in amr_grid.nodes:
        node.value = inverse_wavelet_transform(node.coordinates, wavelet_grid, 
                                               scaling_func, wavelet_func, 
                                               cutoff_phi, cutoff_psi)
                                               
    # Step 5: Export to VTK format (tetrahedral mesh)
    vtk_data = convert_to_vtk(amr_grid)
    
    return amr_grid, vtk_data

def is_inside_cell(coordinates, cell):
    """Check if coordinates are strictly inside the cell boundaries."""
    # Implementation depends on cell geometry
    return cell.min_x < coordinates.x < cell.max_x and \
           cell.min_y < coordinates.y < cell.max_y and \
           cell.min_z < coordinates.z < cell.max_z

def is_on_face_or_edge(coordinates, cell):
    """Check if coordinates are on the face or edge of the cell."""
    # Implementation depends on cell geometry
    # Check if any coordinate is within epsilon of cell boundary
    return (abs(coordinates.x - cell.min_x) < epsilon or 
            abs(coordinates.x - cell.max_x) < epsilon or
            abs(coordinates.y - cell.min_y) < epsilon or 
            abs(coordinates.y - cell.max_y) < epsilon or
            abs(coordinates.z - cell.min_z) < epsilon or 
            abs(coordinates.z - cell.max_z) < epsilon)

def refine_cell(amr_grid, cell, level):
    """Refine a cell up to the specified level."""
    # Implementation of cell refinement logic
    # This may involve recursive subdivision
    pass

def add_supplementary_nodes(amr_grid):
    """Add nodes to ensure node-centered topology."""
    # Implementation to add missing nodes at vertices
    pass

def inverse_wavelet_transform(coordinates, wavelet_grid, scaling_func, wavelet_func, cutoff_phi, cutoff_psi):
    """
    Compute function value at coordinates using inverse wavelet transform.
    """
    value = 0.0
    for node in wavelet_grid:
        # Calculate contribution based on distance and basis functions
        # Note: Exact formula not provided in sources
        # Typically involves summing scaling and wavelet contributions
        pass
    return value

def convert_to_vtk(amr_grid):
    """Convert AMR grid to VTK unstructured mesh format."""
    # Decompose each cubic cell into six tetrahedra
    # Write to VTK format
    pass
```

**Parameters:**
*   Cutoff limits: $\xi_0^\phi = 1$ for scaling function, $\xi_0^\psi = 2$ for wavelet function [S1][S2, unofficial].
*   Typical values for other parameters are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The inverse wavelet transform may require careful handling of numerical precision, especially for nodes far from the original wavelet support [G].
*   **Boundaries:** Wavelet nodes at cell boundaries (faces/edges) require special handling to ensure all adjacent cells are refined correctly [S1][S2, unofficial].
*   **Degenerate Input:** If the wavelet grid is empty or has no nodes, the AMR grid remains at the coarsest level [G].

## Subtopics

### Direct Summation
Computes values directly in wavelet space without converting to AMR. Faster preprocessing but requires custom rendering code [S1][S2, unofficial].

### AMR Volume Rendering
The rendering step performed on the completed AMR grid, typically using VTK-based tools [S1][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time depends on the complexity of the wavelet grid and may take significant time [S1][S2, unofficial]. For time-dependent problems, preprocessing time is expected to be of the same order of magnitude as the computational time for a single time step [S1][S2, unofficial].
*   **Memory Complexity:** Requires additional storage for supplementary nodes [S1][S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Allows use of existing AMR visualization software (ParaView, VisIt) [S1][S2, unofficial]. Deals with actual function values on a traditional node-centered AMR grid [S1][S2, unofficial].
*   **Limitations:** Requires additional memory and significant preprocessing time [S1][S2, unofficial].
*   **When to use:** When existing AMR rendering pipelines are preferred over custom wavelet-based renderers [S1][S2, unofficial].
*   **When not to use:** When preprocessing time and memory overhead are critical constraints [S1][S2, unofficial].

## Background
*   **Origin:** The method is described in the paper "Volumetric Rendering on Wavelet-Based Adaptive Grid" [S1][S2, unofficial].
*   **History:** The approach is presented as an alternative to direct summation in wavelet space for volume rendering of wavelet-based data [S1][S2, unofficial].
*   **Usage:** Used for volume rendering of wavelet-based data using standard AMR visualization tools [S1][S2, unofficial].

## Sources
[S1] Volumetric Rendering on Wavelet-Based Adaptive Grid, https://doi.org/10.3390/fluids7070245, fetched 2026-10-26
[S2] Volumetric Rendering on Wavelet-Based Adaptive Grid - MDPI, https://www.mdpi.com/2311-5521/7/7/245, fetched 2026-10-28 (unofficial)
