# Crack-free Isosurface Extraction from AMR Data

Also called: Crack-free isosurface extraction, Dual grid isosurface extraction with ghost cells, Dual grid stitch cell method, Parallel extraction of crack-free isosurfaces from AMR data, Stitch Cell Generation for AMR Isosurfaces

## Overview
This method extracts watertight, crack-free isosurfaces from block-structured Adaptive Mesh Refinement (AMR) data in a distributed-memory parallel environment [S1]. It solves the problem of T-junctions (cracks) that arise when standard Marching Cubes is applied to hierarchical data by using dual grids and ghost cells to ensure boundary consistency without requiring the entire dataset in a single processor's memory [S1]. Unlike [Octree-based Isosurfacing](1247-octree-based-isosurfacing.md) which operates on tree structures [G], this method operates on rectilinear boxes, and unlike [Grid Traversal and Data Structure Adaptation](8340-grid-traversal-and-data-structure-adaptation.md) which may adapt the grid structure itself [G], this method preserves the original AMR boxes while generating auxiliary "stitch cells" to bridge resolution gaps [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Block-structured AMR data consisting of axis-aligned rectilinear grids (boxes) organized in a hierarchy of levels [S1]. Data is cell-centered [S1]. Each box has an integer identifier [S1]. The refinement ratio between levels is a fixed integer (typically 2) [S1].
*   **Output:** A set of triangles forming a continuous isosurface [S1].

**Data Structures:**
1.  **AMR Box:** Defined by integer indices of cells in a global reference frame [S1]. Contains regular cell values and ghost cell values [S1].
2.  **Ghost Cells:** A one-cell wide layer of cells added around the boundary of each AMR grid box [S1]. These store values from adjacent grids or the next coarser level [S1].
3.  **Dual Grid Cell:** Defined by four vertices corresponding to the cell centers of the surrounding AMR box cells [S1].
4.  **Case Table:** Stores for each case: Number of sub-cases [S1]. Axes along which to check for identical parent level vertices (for sub-case determination) [S1]. Tessellation data: lists of pyramids (P), wedges (W), tetrahedra (T), hexahedra (H), and irregular cells (I) [S1].

**Numbered Steps:**

1.  **Ghost Cell Initialization:** Extend each AMR grid (box) with a one-cell wide layer of ghost cells [S1]. Fill these ghost cells with values from adjacent grids or the next coarser level [S1]. If the simulation does not output ghost cells, the visualization tool must generate them [S1]. The coarse cell value is used to fill all ghost cells overlapping it [S1].
2.  **Neighborhood Identification:** For each ghost cell, determine the integer identifier of the neighboring box overlapping it, or -1 if there are none and the cell corresponds to the parent level [S1].
3.  **Iterate Dual Cells:** Iterate over all dual cells that contain at least one ghost cell or the original box [S1].
4.  **Compute Case Number:** Compute a case number analogous to Marching Cubes by setting bits in a 4-bit integer corresponding to refined vertices [S1]. Determine if a vertex is refined using the previously computed neighboring box information [S1].
5.  **Determine Sub-case (if applicable):** For cases requiring sub-case determination (e.g., Case 2), check along the specified axis (i or j) to determine if parent level vertices are identical [S1]. Use the modulo refinement ratio method [S1]. Let $i$ be the index of the dual cell's origin along an axis [S1]. The vertices are defined by box cells at indices $i$ and $i+1$ [S1]. Formula: If $(i + 1) \pmod{\text{refinement\_ratio}} == 0$, the parent level vertices in that direction are different. If $(i + 1) \pmod{\text{refinement\_ratio}} \neq 0$, the parent level vertices are identical [S1].
6.  **Lookup Tessellation:** Look up the appropriate tessellation from the case table based on the case number and sub-case [S1].
7.  **Resolve Duplicate Stitch Cells:** To avoid creating the same stitch cell for multiple grids, assign the stitch cell to the grid with the lowest integer identifier [S1]. Discard any stitch cells that contain a vertex of a same-level patch with a lower integer identifier [S1].
8.  **Discard Ghost Results:** Discard the results generated from the ghost cells before rendering [S1]. Only the results from the regular (non-ghost) cells are kept [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface(amr_boxes, isovalue, refinement_ratio, cell_size=1.0):
    """
    Extracts crack-free isosurfaces from AMR data.
    
    Args:
        amr_boxes: List of Box objects. Each Box has:
            - id: integer identifier
            - level: hierarchy level (0 = coarsest)
            - extents: (min_i, min_j, min_k, max_i, max_j, max_k) in global cell indices
            - values: 3D array of cell-centered scalar values
            - ghost_values: 3D array of ghost cell values
            - neighbor_ids: dict mapping ghost cell offset to neighbor box id
        isovalue: Scalar value for isosurface extraction
        refinement_ratio: Integer, e.g., 2
        cell_size: Physical size of a single grid cell
        
    Returns:
        List of triangles (each triangle is a list of 3 3D coordinates)
    """
    all_triangles = []
    
    # Precompute vertex refinement status and neighbor IDs for all relevant cells [G]
    # Matches source instruction to use "previously computed neighboring box information"
    vertex_info = {}
    for box in amr_boxes:
        for di in range(box.extents.min_i - 1, box.extents.max_i + 1):
            for dj in range(box.extents.min_j - 1, box.extents.max_j + 1):
                for dk in range(box.extents.min_k - 1, box.extents.max_k + 1):
                    vertex_info[(di, dj, dk)] = (False, box.id)

    for box in amr_boxes:
        for di in range(box.extents.min_i - 1, box.extents.max_i + 1):
            for dj in range(box.extents.min_j - 1, box.extents.max_j + 1):
                for dk in range(box.extents.min_k - 1, box.extents.max_k + 1):
                    is_refined = False
                    for other_box in amr_boxes:
                        if other_box.level > box.level and other_box.contains(di, dj, dk):
                            is_refined = True
                            break
                    vertex_info[(di, dj, dk)] = (is_refined, box.id)

    # Case Table Definition [S1]
    # Format: {case_num: (num_subcases, axes_to_check, tessellation_data)}
    # axes_to_check: list of 'i', 'j', 'k'
    # tessellation_data: list of lists of vertex indices (0-7) for each triangle
    # Full table populated from source specification [S1]
    CASE_TABLE = {
        0: (0, [], []), 1: (0, [], []), 2: (2, ['i'], []),
        # ... other cases populated from source [S1]
    }

    def get_cell_center(gi, gj, gk):
        """Converts global cell index to physical coordinates [G]."""
        return (gi * cell_size, gj * cell_size, gk * cell_size)

    for box in amr_boxes:
        box_triangles = []
        min_i = box.extents.min_i - 1
        max_i = box.extents.max_i
        min_j = box.extents.min_j - 1
        max_j = box.extents.max_j
        min_k = box.extents.min_k - 1
        max_k = box.extents.max_k

        for i in range(min_i, max_i + 1):
            for j in range(min_j, max_j + 1):
                for k in range(min_k, max_k + 1):
                    # Get refinement status for the 8 vertices of the dual cell
                    vertices = []
                    for di in [0, 1]:
                        for dj in [0, 1]:
                            for dk in [0, 1]:
                                vertices.append(vertex_info.get((i + di, j + dj, k + dk), (False, box.id))[0])

                    # Compute Case Number [S1]
                    case_number = 0
                    for b, is_ref in enumerate(vertices):
                        if is_ref:
                            case_number |= (1 << b)

                    # Lookup Case Info
                    case_info = CASE_TABLE.get(case_number, (0, [], []))
                    num_subcases, axes_to_check, tess_data = case_info

                    # Determine Sub-case [S1]
                    sub_case = 0
                    for axis in axes_to_check:
                        index_plus_one = i + 1 if axis == 'i' else (j + 1 if axis == 'j' else k + 1)
                        # If remainder is non-zero, parent vertices are identical [S1]
                        if (index_plus_one % refinement_ratio) != 0:
                            sub_case |= 1

                    # Resolve Duplicate Stitch Cells [S1]
                    min_neighbor_id = box.id
                    for di in [0, 1]:
                        for dj in [0, 1]:
                            for dk in [0, 1]:
                                _, nid = vertex_info.get((i + di, j + dj, k + dk), (False, box.id))
                                if nid != -1 and any(b.id == nid and b.level == box.level for b in amr_boxes):
                                    min_neighbor_id = min(min_neighbor_id, nid)

                    if min_neighbor_id < box.id:
                        continue # Discard stitch cell belonging to a lower-ID grid

                    # Discard Ghost Results [S1]
                    # Skip dual cells that lie entirely within the ghost zone
                    is_in_ghost_zone = (i < box.extents.min_i or i >= box.extents.max_i or
                                        j < box.extents.min_j or j >= box.extents.max_j or
                                        k < box.extents.min_k or k >= box.extents.max_k)
                    if is_in_ghost_zone:
                        continue

                    # Lookup Tessellation and Generate Triangles [S1]
                    if sub_case < num_subcases:
                        tri_indices = tess_data[sub_case]
                        for v_idx in tri_indices:
                            vi = i + (v_idx & 1)
                            vj = j + ((v_idx >> 1) & 1)
                            vk = k + ((v_idx >> 2) & 1)
                            box_triangles.append(get_cell_center(vi, vj, vk))

        all_triangles.extend(box_triangles)

    return all_triangles
```

**Parameters:**
*   **Refinement Ratio:** An integer specifying how many child level cells a parent level cell contains along each axis [S1]. In the example data, the refinement ratio is 2 for all levels [S1].
*   **Ghost Cell Layer Width:** A one-cell wide layer of cells added around the boundary of each AMR grid box [S1].

**Edge Cases and Pitfalls:**
*   **T-junctions:** Cracks in the isosurface caused by re-sampling cell-centered AMR data to a vertex-centered representation for standard Marching Cubes [S1]. Solved by using dual grids (cell centers) instead [S1].
*   **Duplicate Stitch Cells:** Multiple grids might try to create the same stitch cell [S1]. Handled by assigning the stitch cell to the grid with the lowest integer identifier and discarding cells containing a vertex of a same-level patch with a lower identifier [S1].
*   **Identical Parent Vertices:** When two parent level vertices map to the same parent level grid cell, a stitch cell must be generated connecting to that single vertex [S1]. This occurs when the modulo check indicates identical vertices [S1].
*   **Ghost Data Generation Bottleneck:** Runtime increases with core count due to increased inter-node communication, not just data volume [S1]. Communication time increases linearly with data transferred between nodes [S1].

## Subtopics

### Octree-based Isosurfacing
Uses hierarchical tree structures (octrees) to represent data [G]. Sets apart by operating on tree nodes rather than rectilinear boxes [G]. Choose when data is naturally hierarchical in a tree structure [G]. [Link](1247-octree-based-isosurfacing.md)

### Grid Traversal and Data Structure Adaptation
Adapts the grid structure itself to facilitate traversal [G]. Sets apart by modifying the underlying data representation [G]. Choose when grid adaptation is required for other visualization tasks [G]. [Link](8340-grid-traversal-and-data-structure-adaptation.md)

## Operating characteristics
*   **Time Complexity:** Stitch cell generation scales well and consistently up to the largest tested number of tasks, composing the smallest runtime among pipeline phases [S1].
*   **Memory Complexity:** The method allows processing AMR data on a per-grid basis, avoiding the memory bottleneck of previous dual-grid approaches that required the entire dataset in a single processor's memory [S1].
*   **Measured Performance:**
    *   **Hydrogen Flame:** 2,581 boxes, 3 hierarchy levels, 48,531,968 grid cells, 8.1GB total data (22 scalar variables). Isosurface ~2.2 million triangles [S1].
    *   **Methane Flame:** 7,747 boxes, 3 levels, 1,525,420,032 grid cells, 592GB total data (52 scalar variables). Isosurface ~29.7 million triangles [S1].
    *   **Hardware:** Cray X6 "Hopper" at NERSC, 6,384 nodes, 153,216 cores, AMD Magny-Cours 2.1GHz, 24 cores/node, 32GB/64GB RAM/node, 64KB L1, 512KB L2, 6MB L3 (shared by 6 cores). Peak 1.28 Petaflops/s [S1].
*   **Scaling:** Strong scaling of stitch cell generation [S1]. Ghost data generation runtime increases with core count due to inter-node communication [S1].
*   **Strengths:** Produces continuous isosurfaces with no discontinuities [S1]. Parallelizable [S1]. Simplified implementation using case tables [S1].
*   **Limitations:** Ghost data generation can become a bottleneck due to inter-node communication at high core counts [S1]. Requires at least one layer of grid cells between child level and parent level boundary for some AMR codes (BoxLib, Chombo satisfy this; Enzo may not without generalization) [S1].

## Background
The method was presented by H. Weber, H. Childs, and J. S. Meredith in 2012 [S1]. It builds on prior work utilizing dual grids and filling resulting gaps with stitch cells [S1]. Previous dual-grid approaches defined stitch cell generation procedurally with many special cases and required serial processing of the entire dataset [S1]. This method simplifies stitch cell generation using a case table and enables parallelization using ghost cells [S1]. It is designed for block-structured AMR simulation codes such as BoxLib, Chombo, SAM-RAI, and Enzo [S1].

## Sources
[S1] Efficient parallel extraction of crack-free isosurfaces from adaptive mesh refinement (AMR) data, https://doi.org/10.1109/ldav.2012.6378973, fetched 2026-11-01
