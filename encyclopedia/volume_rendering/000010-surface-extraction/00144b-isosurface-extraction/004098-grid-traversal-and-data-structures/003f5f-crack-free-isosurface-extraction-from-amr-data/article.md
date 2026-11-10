# Crack-free Isosurface Extraction from AMR Data

Also called: Crack-free isosurface extraction, Dual grid isosurface extraction with ghost cells, Dual grid stitch cell method, Parallel extraction of crack-free isosurfaces from AMR data, Stitch Cell Generation for AMR Isosurfaces

## Overview
This method extracts watertight, crack-free isosurfaces from block-structured Adaptive Mesh Refinement (AMR) data in a distributed-memory parallel environment [S1]. It solves the problem of T-junctions (cracks) that arise when standard Marching Cubes is applied to hierarchical data, and overcomes the memory and serial processing limitations of previous dual-grid approaches by using ghost cells [S1]. Unlike re-sampling to a vertex-centered format which produces visible cracks, this method produces a continuous isosurface with no discontinuities [S1].

## How it works
The method operates on block-structured AMR data, typically produced by codes such as BoxLib, Chombo, SAM-RAI, and Enzo [S1]. The input is a hierarchy of axis-aligned rectilinear grids (boxes) at multiple levels, where finer levels replace coarser levels. Data is cell-centered [S1]. The output is a triangle mesh representing the isosurface at a specified threshold [S1].

### Data Structures

**AMR Box Structure**
Each box is defined by its integer indices in a global reference frame.
*   `id`: Integer identifier for the box.
*   `level`: Hierarchy level index.
*   `extent`: Tuple `(min_i, min_j, min_k, max_i, max_j, max_k)` defining the box boundaries in global cell indices.
*   `data`: Array of scalar values, cell-centered. Dimensions correspond to the box extent.

**Dual Grid Cell**
The algorithm operates on a dual grid defined by the cell centers of the AMR boxes. A dual cell is defined by four vertices (in 2D) or eight vertices (in 3D) corresponding to the corners of the primal cell.
*   `origin`: Global indices `(i, j, k)` of the primal cell.
*   `vertices`: List of global indices of the surrounding primal cells.

**Ghost Cell Layer**
Each AMR box is extended with a one-cell wide layer of ghost cells [S1].
*   `ghost_data`: Scalar values for the ghost layer.
*   `source_type`: Enum indicating if the value comes from an adjacent box (`BOX`) or the parent level (`PARENT`) [S1].
*   `source_id`: Integer identifier of the neighboring box if `source_type` is `BOX`, or `-1` if `PARENT` [S1].

**Case Table**
A static table storing tessellation data for stitch cells.
*   `num_subcases`: Integer count of sub-cases for this entry.
*   `check_axes`: List of axes (`i`, `j`, `k`) along which to check for identical parent level vertices.
*   `tessellation`: List of geometric primitives (pyramids `P`, wedges `W`, tetrahedra `T`, hexahedra `H`, irregular cells `I`) [S1].

### Parameters
*   `isosurface_threshold`: Scalar value defining the isosurface [S1].
*   `refinement_ratio`: Integer specifying how many child level cells a parent level cell contains along each axis. Typically 2 [S1].
*   `ghost_layer_width`: Width of the ghost cell layer. Fixed at 1 cell [S1].

### Algorithm Steps

1.  **Dual Grid Conversion**: Convert the block-structured AMR data from cell-centered to a dual grid representation defined by the cell centers of the AMR boxes. This avoids T-junctions caused by re-sampling to a vertex-centered representation [S1].

2.  **Ghost Cell Initialization**: Extend each AMR grid (box) with a one-cell wide layer of ghost cells. Fill these ghost cells with values from adjacent grids or the next coarser level. This ensures all data required to construct stitch cells are available locally for each grid [S1].
    *   If a ghost cell overlaps with an adjacent box, copy the value from that box.
    *   If a ghost cell overlaps with the parent level (coarser grid), copy the value from the parent cell. Note that the same coarse cell value is used to fill all ghost cells overlapping it [S1].

3.  **Stitch Cell Generation**: Iterate over all dual cells that contain at least one ghost cell or the original box [S1].
    *   Compute a case number analogous to Marching Cubes by setting bits in a 4-bit integer corresponding to refined vertices. Determine if a vertex is refined using previously computed neighboring box information [S1].
    *   For cases requiring sub-case determination (e.g., Case 2), check along the axes stored in the case table. Use the modulo refinement ratio method to determine if parent level vertices are identical.
    *   Look up the appropriate tessellation (primitives) from the Case Table based on the case number and sub-case [S1].
    *   Generate the geometric primitives (vertices and faces) for the stitch cell.

4.  **Duplicate Handling**: To avoid creating the same stitch cell for multiple grids, assign the stitch cell to the grid with the lowest integer identifier. Discard any stitch cells that contain a vertex of a same-level patch with a lower integer identifier [S1].

5.  **Result Filtering**: Discard the results generated from the ghost cells before rendering. Only the results from the regular (non-ghost) cells are kept, as the ghost cells merely duplicate regular cell data to ensure boundary consistency [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface_amr(boxes, isovalue, refinement_ratio):
    """
    Extracts crack-free isosurfaces from AMR data.
    
    Args:
        boxes: List of Box objects, each containing id, level, extent, and data.
        isovalue: Scalar threshold for isosurface extraction.
        refinement_ratio: Integer, number of child cells per parent cell along an axis.
        
    Returns:
        List of triangles (faces) forming the isosurface.
    """
    all_triangles = []
    
    # Step 1: Initialize Ghost Cells for each box
    for box in boxes:
        box.ghost_data = initialize_ghost_cells(box, boxes, refinement_ratio)
        
    # Step 2: Process each box to generate stitch cells and regular isosurface parts
    for box in boxes:
        # Get dual cells that involve ghost cells or the box itself
        relevant_dual_cells = get_relevant_dual_cells(box)
        
        box_triangles = []
        
        for dual_cell in relevant_dual_cells:
            # Compute case number based on refined vertices
            case_number = compute_case_number(dual_cell, box)
            
            # Determine sub-case if necessary
            sub_case = 0
            if case_number in cases_with_subcases:
                # Check axes specified in Case Table for identical parent vertices
                sub_case = determine_sub_case(dual_cell, box, refinement_ratio)
            
            # Get tessellation from Case Table
            tessellation = case_table[case_number][sub_case]
            
            # Generate primitives
            primitives = generate_primitives(tessellation, dual_cell, box)
            
            # Duplicate handling: discard if vertex belongs to a lower-ID box
            valid_primitives = []
            for prim in primitives:
                if is_valid_stitch_cell(prim, box, boxes):
                    valid_primitives.append(prim)
            
            box_triangles.extend(valid_primitives)
            
        # Step 5: Discard ghost cell results (already handled by valid_primitives check)
        # Only keep triangles from regular cells (implicitly handled by processing only 
        # relevant dual cells and filtering duplicates)
        all_triangles.extend(box_triangles)
        
    return all_triangles

def initialize_ghost_cells(box, boxes, refinement_ratio):
    """
    Fills the one-cell wide ghost layer around a box.
    """
    ghost_data = {}
    for axis in ['i', 'j', 'k']:
        # Determine ghost cells on both sides of the box
        ghost_cells = get_ghost_cells_for_box(box, axis)
        
        for gc in ghost_cells:
            # Check if overlapping with another box
            overlapping_box = find_overlapping_box(gc, boxes)
            
            if overlapping_box is not None:
                # Copy value from adjacent box
                ghost_data[gc] = overlapping_box.data[gc.local_index]
                ghost_data[gc].source_type = 'BOX'
                ghost_data[gc].source_id = overlapping_box.id
            else:
                # Copy value from parent level
                parent_cell = get_parent_cell(gc, box.level, refinement_ratio)
                ghost_data[gc] = parent_cell.value
                ghost_data[gc].source_type = 'PARENT'
                ghost_data[gc].source_id = -1
                
    return ghost_data

def determine_sub_case(dual_cell, box, refinement_ratio):
    """
    Uses modulo refinement ratio to determine if parent level vertices are identical.
    """
    # If (index + 1) % refinement_ratio == 0, parent level vertices are different.
    # If (index + 1) % refinement_ratio != 0, parent level vertices are identical.
    
    # Check along axes specified in Case Table
    for axis in case_table[dual_cell.case_number].check_axes:
        index = dual_cell.origin[axis]
        if (index + 1) % refinement_ratio != 0:
            # Vertices are identical, this affects the sub-case
            return 1 # Example sub-case logic
    return 0

def is_valid_stitch_cell(primitive, current_box, all_boxes):
    """
    Discards stitch cells that contain a vertex of a same-level patch with a lower identifier.
    """
    for vertex in primitive.vertices:
        # Check if vertex belongs to a same-level box with lower ID
        if vertex.belongs_to_same_level_box() and vertex.box_id < current_box.id:
            return False
    return True
```

### Edge Cases and Pitfalls
*   **T-junctions**: Cracks in the isosurface caused by re-sampling cell-centered AMR data to a vertex-centered representation. Solved by using dual grids (cell centers) instead [S1].
*   **Identical Parent Vertices**: When two parent level vertices map to the same parent level grid cell, a stitch cell must be generated connecting to that single vertex. This occurs when the modulo check indicates identical vertices [S1].
*   **Duplicate Stitch Cells**: Multiple grids might try to create the same stitch cell. This is handled by assigning the stitch cell to the grid with the lowest integer identifier and discarding cells containing a vertex of a same-level patch with a lower identifier [S1].
*   **Ghost Data Generation Bottleneck**: Runtime increases with core count due to increased inter-node communication, not just data volume. Communication time increases linearly with data transferred between nodes [S1].

## Operating characteristics
*   **Time Complexity**: Stitch cell generation scales well and consistently up to the largest tested number of tasks, composing the smallest runtime among pipeline phases [S1]. Ghost data generation runtime increases with core count due to inter-node communication [S1].
*   **Memory Complexity**: The method allows processing each AMR grid independently, avoiding the memory bottleneck of previous methods that required the entire dataset in a single processor's memory [S1].
*   **Measured Performance**:
    *   **Hydrogen Flame**: 2,581 boxes, 3 levels, 48,531,968 grid cells, 8.1GB data. Isosurface ~2.2 million triangles [S1].
    *   **Methane Flame**: 7,747 boxes, 3 levels, 1,525,420,032 grid cells, 592GB data. Isosurface ~29.7 million triangles [S1].
    *   **Hardware**: Cray X6 "Hopper", 6,384 nodes, 153,216 cores, AMD Magny-Cours 2.1GHz, 24 cores/node, 32GB/64GB RAM/node [S1].
*   **Strengths**: Produces continuous, crack-free isosurfaces; parallelizable; handles large-scale data; simplifies stitch cell generation with case tables [S1].
*   **Limitations**: Ghost data generation can become a bottleneck due to inter-node communication at high core counts [S1]. Requires AMR data to have at least one layer of grid cells between child level and parent level boundary (satisfied by BoxLib/Chombo) [S1].

## Background
The method was introduced by Weber, Childs, and Meredith in 2012 [S1]. It builds on prior work utilizing dual grids and filling resulting gaps with stitch cells [S1]. Previous dual-grid approaches generated stitch cells serially and required the entire dataset to reside in the memory of a single processor [S1]. This method addresses these limitations by using ghost cells for parallelization and a case-table-based approach for simplified implementation [S1]. It is designed for block-structured AMR simulation codes such as BoxLib, Chombo, SAM-RAI, and Enzo [S1].

## Sources
[S1] Efficient parallel extraction of crack-free isosurfaces from adaptive mesh refinement (AMR) data, https://doi.org/10.1109/ldav.2012.6378973, fetched 2026-11-01
