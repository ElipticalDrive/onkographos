# Ghost Cell Stitching for Node-Centered AMR

Also called: Ghost Cell Generation for AMR, Ghost Cell Stitching, Ghost cell generation, Ghost data generation

## Overview
Ghost Cell Stitching is a method for handling data continuity between different Adaptive Mesh Refinement (AMR) levels in node-centered grids by introducing ghost cells at level boundaries [S1, unofficial]. It solves the problem of rendering artifacts (such as cracks or "ghosting") that occur when isosurface extraction or volume rendering requires access to neighboring cells that reside on different processing elements or different refinement levels [S2]. Unlike methods that redistribute the entire dataset, this approach uses simulation-provided abutment information to selectively exchange only the necessary boundary data, ensuring that stitch cells can be generated consistently across coarse-fine interfaces [S2]. It builds on [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) [G].

## How it works
The method generates ghost cells to provide surrounding data for stitch cell generation, ensuring that the values at the vertices of stitch cells are consistent across AMR level boundaries [S2]. It relies on the fact that for node-centered AMR, both sides of a level boundary form structured node-centered grids that naturally lend to trilinear interpolation [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   AMR dataset consisting of boxes at various refinement levels [S2].
    *   `BoxAbutment` data structure: For each box $B$, this lists the boxes $B^*_0, B^*_1, \dots, B^*_{n-1}$ that $B$ abuts with and the locations of those abutments. This includes abutment for boxes at the same refinement level and nesting for boxes at different refinement levels [S2].
    *   `BoxToProcessingElement` map: A map indicating which processing element (PE) each box resides on [S2].
    *   Grid data values for each box [S2].
*   **Outputs**:
    *   Extended boxes where boundary cells are duplicated as ghost cells [S1, unofficial].
    *   `Ghost Flag Array`: An array specifying for each grid cell whether it is a ghost cell. If flagged, the type is also specified (boundary ghost or invalidation ghost) [S2].

### Data Structures
*   **BoxAbutment[B]**: A list of tuples or objects describing abutting boxes. Each entry contains the reference to the abutting box $B^*$ and the specific location/offset of the abutment region on box $B$ [S2].
*   **Ghost Flag Array**: A boolean or integer array aligned with the grid cells. It indicates if a cell is a ghost. If true, it distinguishes between:
    *   *Boundary Ghost*: Cells added to support stitching at domain or level boundaries [S2].
    *   *Invalidation Ghost*: Cells marked invalid because a finer grid provides a more accurate representation [S2].
*   **Messages**: Data structures used during the exchange phase, containing destination box ID, the data values, and the location of that data on the destination patch [S2].

### Parameters
*   **Ghost Cell Count**: A single layer of ghost cells around a box is always sufficient to determine appropriate stitch cell vertex values [S2].
*   **Level Dependency**: When generating ghost data for a box in level $i$, only data in the same level $i$ and the parent level $i-1$ are considered. Data from finer levels ($> i$) is ignored during this specific generation step [S2].

### Steps
The ghost cell generation process involves four phases: Initialize, Pack, Exchange, and Unpack [S2].

1.  **Initialize**:
    *   Each processing element (PE) identifies which boxes it has loaded [S2].
    *   PEs exchange information to create the `BoxToProcessingElement` map, stating which PE each box resides on [S2].

2.  **Pack**:
    *   Every PE iterates over the boxes $B$ it owns [S2].
    *   For each box $B$, determine its abutting patches $B_i$ using `BoxAbutment[B]` [S2].
    *   For each abutting patch $B_i$, identify the one layer of cells in the abutment region [S2].
    *   Convert these cells into a message [S2].
    *   Append this message to the overall message buffer designated for the PE identified by `BoxToProcessingElement[B_i]` [S2].

3.  **Exchange**:
    *   Each PE sends its accumulated messages to the other PEs [S2].
    *   This is typically implemented using an MPI `Alltoallv` command, allowing the MPI implementation to optimize communication based on the interconnect [S2].

4.  **Unpack**:
    *   Every PE receives messages from other PEs [S2].
    *   For each incoming message, the PE extends its local boxes with ghost cells [S2].
    *   The data for these ghost cells is populated from the incoming messages [S2].

### Ghost Cell Value Assignment Rules
When filling the ghost cells during the Unpack phase (or if generating locally without exchange), the following rules determine the values [S2]:

1.  **Domain Boundaries**: If a ghost cell coincides with the boundary of the domain, there is no need to connect to other grids. No ghost data is needed or generated for these edges [S2].
2.  **Adjacent Box Values**: If a ghost cell coincides with the cell of an adjacent box (in the same level), the ghost cell must be filled with the value from that adjacent box [S2].
3.  **Coarse Parent Values**: If no neighboring box overlapping the ghost cell exists in the same level, the stitch cell (and thus the ghost cell) must be filled with the value from the cell it refines in the parent level ($i-1$). Since a level is always completely contained in its parent level, such a cell always exists. This ensures that several cells in the fine box (corresponding to the refinement ratio) are filled with the coarse grid value, simplifying look-up for connecting coarse grid values in stitch cell generation [S2].
4.  **Invalidation**: If a cell is flagged as an invalidation ghost (due to finer grid refinement), it is marked in the `Ghost Flag Array`. Visualization algorithms use this flag to blank out these cells, operating on the original rectilinear grids [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_ghost_cells(boxes, box_abutment, box_to_pe, local_pe_id):
    """
    Generates ghost cells for AMR data using the Pack-Exchange-Unpack pattern.
    
    Args:
        boxes: Dict mapping box_id -> Box object. Box object has:
               - data: 3D array of values
               - ghost_flag_array: array aligned with data
        box_abutment: Dict mapping box_id -> List of AbutmentInfo.
                      AbutmentInfo has:
                      - neighbor_box_id: int
                      - abutment_region_indices: list of local cell indices
                      - target_indices: list of indices in the neighbor's ghost layer
        box_to_pe: Dict mapping box_id -> int (PE ID)
        local_pe_id: int, the ID of the current processing element.
        
    Returns:
        Updated boxes with ghost cells added and flagged.
    """
    
    # Phase 1: Initialize
    # box_to_pe is populated via collective communication (e.g., MPI_Allgather)
    # to map every box_id to its owning PE.
    
    # Phase 2: Pack
    # messages[dest_pe_id] stores lists of data to send to each destination PE
    messages = {pe_id: [] for pe_id in set(box_to_pe.values())}
    
    for box_id, box in boxes.items():
        # Only process boxes owned by this PE
        if box_to_pe[box_id] != local_pe_id:
            continue
            
        # Get abutting patches for this box
        abutments = box_abutment.get(box_id, [])
        for abut in abutments:
            neighbor_box_id = abut.neighbor_box_id
            dest_pe_id = box_to_pe[neighbor_box_id]
            
            # Identify the one layer of cells in the abutment region
            region_indices = abut.abutment_region_indices
            values = [box.data[i] for i in region_indices]
            
            msg = {
                'source_box_id': box_id,
                'values': values,
                'target_box_id': neighbor_box_id,
                'target_indices': abut.target_indices
            }
            messages[dest_pe_id].append(msg)
            
    # Phase 3: Exchange
    # In MPI, this is MPI_Alltoallv.
    # We simulate the collective exchange by gathering messages destined for local_pe_id.
    received_messages = []
    for dest_pe, msg_list in messages.items():
        if dest_pe == local_pe_id:
            received_messages.extend(msg_list)
            
    # Phase 4: Unpack
    for msg in received_messages:
        target_box_id = msg['target_box_id']
        target_indices = msg['target_indices']
        values = msg['values']
        
        if target_box_id in boxes:
            target_box = boxes[target_box_id]
            # Extend box data structure to accommodate ghost cells if necessary
            # Set values in ghost cells from incoming message
            for idx, val in zip(target_indices, values):
                target_box.data[idx] = val
            # Mark these cells as ghost cells in the Ghost Flag Array
            for idx in target_indices:
                target_box.ghost_flag_array[idx] = 'boundary_ghost'
                
    return boxes
```

### Edge Cases and Pitfalls
*   **Domain Boundaries**: If a ghost cell coincides with the boundary of the domain, there is no need to connect to other grids. The algorithm should not attempt to fetch data from non-existent neighbors [S2].
*   **Adjacent Box Values**: If a ghost cell coincides with the cell of an adjacent box (in the same level), the ghost cell must be filled with the value from the adjacent box. This ensures continuity across box boundaries within the same level [S2].
*   **Coarse Parent Values**: If no neighboring box overlapping the ghost cell exists in the same level, the value must be taken from the parent level ($i-1$). This is guaranteed to exist because a level is always completely contained in its parent level [S2].
*   **Level Dependency**: When generating ghost data for a box in level $i$, only data from level $i$ and level $i-1$ are considered. Data from finer levels ($> i$) is ignored because it will be handled when generating stitch cells for the finer boxes themselves [S2].
*   **Precision**: The method relies on the structured nature of node-centered grids. For cell-centered AMR, stitching requires unstructured mesh geometry (tets or hexes), making this method inapplicable without significant modification [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The complexity is dominated by the communication phase. The Pack and Unpack phases are linear with respect to the number of boundary cells. The Exchange phase uses MPI `Alltoallv`, which is efficient but requires collective communication [S2].
*   **Memory Complexity**: Requires storage for the `BoxAbutment` structure and the `Ghost Flag Array`. The ghost cells themselves add a small overhead (one layer of cells per box boundary) [S2].
*   **Measured Performance**: This approach incurs lower costs than methods that redistribute the entire dataset (like D3) because it only exchanges necessary boundary data [S2].
*   **Strengths**:
    *   Efficient for in-core processing.
    *   Leverages existing simulation abutment information.
    *   Ensures crack-free isosurfaces by providing consistent stitch cell vertex values [S2].
*   **Limitations**:
    *   Requires collective communication, which can be a bottleneck in large-scale parallel systems.
    *   Cannot work in an out-of-core setting due to the requirement for collective communication and the `BoxAbutment` data structure [S2].
    *   Specific to node-centered AMR; cell-centered AMR requires different handling [S1, unofficial].

## Background
*   **Origin**: The method is based on previous work using dual grids and procedurally generated stitch cells [S2]. It was developed to address artifacts in parallel processing boundaries in AMR isosurface extraction [S2].
*   **History**: The approach was described in the context of efficient parallel extraction of crack-free isosurfaces from AMR data [S2]. It contrasts with earlier methods like D3, which redistributed the entire dataset, and Isenburg et al., which worked in an out-of-core setting for block-decomposed rectilinear grids [S2].
*   **Usage**: It is used in visualization tools like VisIt to handle ghost data at boundaries and due to invalidation by finer grids [S2].

## Sources
[S1] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
[S2] Efficient parallel extraction of crack-free isosurfaces from adaptive mesh refinement (AMR) data, https://doi.org/10.1109/ldav.2012.6378973, fetched 2026-11-01
