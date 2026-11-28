# Meta-Cell Edge Octree Connection

Also called: Edge-Octree, Meta-Cell Boundary Processing

## Overview
This method connects mesh components generated from individual meta-cells by processing boundary edges that were skipped during the initial extraction phase [S1, unofficial]. It solves the problem of hierarchical disconnection between adjacent cells in an adaptive octree built from meta-cells, specifically addressing edges shared by multiple meta-cells that could not be processed due to data locality constraints [S1, unofficial]. It differs from its parent method, [Edge-Octree Construction for Meta-Cell Adjacency](4701-edge-octree-construction-for-meta-cell-adjacency.md), by performing the actual connectivity resolution using a partial octree structure, whereas the parent method focuses on the structural construction of the adjacency graph [S1, unofficial].

## How it works
**Inputs:**
- The set of meta-cells extracted in the previous stage.
- For each meta-cell, the list of cells that have at least one face on the external face of the meta-cell, and all cells adjacent to these boundary cells [S1, unofficial].
- The Morton codes of the meta-cells involved in the adjacency [S1, unofficial].

**Outputs:**
- A connected mesh component for the edges shared between adjacent meta-cells [S1, unofficial].

**Data Structures:**
- **Edge-Octree:** A partial octree containing only cells adjacent to faces of meta-cells that are adjacent to each other via a meta-edge [S1, unofficial].
- **Morton Code Mapping:** A lookup table mapping edge orientations and neighbor offsets to 3-bit Morton codes for adjacency queries [S1, unofficial].

**Steps:**
1. **Identify Unprocessed Edges:** Identify edges belonging to three other cells. Each such edge belongs to faces shared between four meta-cells [S1, unofficial].
2. **Query Adjacency:** Use Morton codes to query the adjacency of the meta-cells sharing these edges. The specific neighbor offsets depend on the reference axis of the meta-cell and the orientation of the edge (X, Y, or Z parallel) [S1, unofficial].
3. **Construct Edge-Octree:** Create a partial octree structure that includes only the cells necessary to resolve the connectivity of these shared edges [S1, unofficial].
4. **Process Boundary Edges:** Calculate the dual vertices for these edges. This step requires Prewitt operator values from voxels outside the current meta-cell, which are now available through the edge-octree structure [S1, unofficial].
5. **Connect Mesh Components:** Integrate the processed boundary edges into the mesh components created for the individual meta-cells [S1, unofficial].

**Morton Code Layout for Edge Adjacency:**
The adjacency is determined by the reference axis of the meta-cell and the parallel axis of the edge. The table below defines the 3-bit Morton codes for the neighbors relative to a base cell $B(i,j,k)$ [S1, unofficial]:

| Edge Orientation | Neighbor Offset | Morton Code |
| :--- | :--- | :--- |
| **X-axis parallel** | $B(i, j, k)$ | 000 |
| | $B(i, j+1, k)$ | 010 |
| | $B(i, j, k+1)$ | 100 |
| | $B(i, j+1, k+1)$ | 110 |
| **Y-axis parallel** | $B(i+1, j, k)$ | 000 |
| | $B(i+1, j, k+1)$ | 001 |
| | $B(i, j, k+1)$ | 100 |
| | $B(i+1, j, k+1)$ | 101 |
| **Z-axis parallel** | $B(i+1, j, k)$ | 000 |
| | $B(i+1, j, k+1)$ | 001 |
| | $B(i, j+1, k)$ | 010 |
| | $B(i+1, j+1, k)$ | 011 |

*Note: The source text contains a slight ambiguity in the Y-axis parallel table entry for the last neighbor, listing $B(i+1, j, k+1)$ twice with different codes. The table above reflects the explicit codes provided in the source [S1, unofficial].*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def connect_meta_cell_edges(meta_cells, edge_octree_builder):
    """
    Connects mesh components for edges shared between adjacent meta-cells.
    """
    # Step 1 & 2: Identify edges and query adjacency
    for meta_cell in meta_cells:
        # Identify edges belonging to three other cells (shared by 4 meta-cells)
        boundary_edges = identify_shared_edges(meta_cell)
        
        for edge in boundary_edges:
            # Determine reference axis and edge orientation
            ref_axis = get_reference_axis(meta_cell)
            edge_axis = get_edge_orientation(edge)
            
            # Query Morton codes for the 4 adjacent meta-cells
            # Using the layout defined in the Morton Code Layout section
            neighbor_codes = query_morton_codes(ref_axis, edge_axis, meta_cell.morton_code)
            
            # Step 3: Construct Edge-Octree for this local neighborhood
            # The edge-octree contains only cells adjacent to faces of meta-cells
            # adjacent via a meta-edge
            local_octree = edge_octree_builder.build(neighbor_codes)
            
            # Step 4: Process boundary edges
            # Calculate dual vertices using Prewitt operator values from voxels
            # outside the current meta-cell, now available in local_octree
            dual_vertices = calculate_dual_vertices(edge, local_octree)
            
            # Step 5: Connect mesh components
            connect_mesh_components(meta_cell.mesh, dual_vertices)

def identify_shared_edges(meta_cell):
    """
    Identifies edges that belong to three other cells and are shared by four meta-cells.
    """
    # Implementation detail: Iterate over the boundary faces of the meta-cell
    # and identify edges that are not internal to the meta-cell
    # but are shared with adjacent meta-cells.
    # [G] This requires checking the boundary faces of the meta-cell
    # and verifying if the edge is shared by exactly 4 meta-cells.
    pass

def query_morton_codes(ref_axis, edge_axis, base_morton_code):
    """
    Returns the Morton codes of the 4 adjacent meta-cells based on the
    reference axis and edge orientation.
    """
    # Decode base_morton_code to (i, j, k)
    i, j, k = decode_morton(base_morton_code)
    
    offsets = []
    if edge_axis == 'X':
        offsets = [
            (0, 0, 0), (0, 1, 0), (0, 0, 1), (0, 1, 1)
        ]
    elif edge_axis == 'Y':
        offsets = [
            (1, 0, 0), (1, 0, 1), (0, 0, 1), (1, 0, 1) # Note: Source ambiguity
        ]
    elif edge_axis == 'Z':
        offsets = [
            (1, 0, 0), (1, 0, 1), (0, 1, 0), (1, 1, 0)
        ]
    
    neighbor_codes = []
    for dx, dy, dz in offsets:
        # [G] Encode (i+dx, j+dy, k+dz) to Morton code
        neighbor_codes.append(encode_morton(i + dx, j + dy, k + dz))
        
    return neighbor_codes
```

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Boundary Edges:** Boundary edges of meta-cells are not processed in the initial stage because calculating dual vertices requires Prewitt operator values from voxels outside the meta-cell [S1, unofficial]. These edges must be processed in this final connection stage [S1, unofficial].
- **Data Locality:** Meta-cells do not have shared access to data from neighboring meta-cells during the initial extraction, necessitating this separate connection step [S1, unofficial].
- **Ambiguity in Morton Codes:** The source text contains a slight ambiguity in the Y-axis parallel Morton code table, listing the same neighbor offset with different codes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Allows for parallel processing of meta-cells by deferring the handling of inter-meta-cell boundaries to a dedicated connection stage [S1, unofficial].
- **Limitations:** Requires additional data structures (edge-octree) and processing steps to resolve connectivity [S1, unofficial].
- **When to use:** When extracting isosurfaces from massive binary volumes using meta-cells and requiring a connected mesh across meta-cell boundaries [S1, unofficial].
- **When not to use:** When meta-cells are not used or when a single global octree is sufficient [S1, unofficial].

## Background
This method is part of a pipeline for extracting isosurfaces from massive binary volumes, specifically domes of salt [S1, unofficial]. It was developed by Lobello et al. in 2014 [S1, unofficial]. The method addresses the challenge of processing large volumes by dividing them into meta-cells, which are processed individually, and then connecting the resulting mesh components [S1, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
