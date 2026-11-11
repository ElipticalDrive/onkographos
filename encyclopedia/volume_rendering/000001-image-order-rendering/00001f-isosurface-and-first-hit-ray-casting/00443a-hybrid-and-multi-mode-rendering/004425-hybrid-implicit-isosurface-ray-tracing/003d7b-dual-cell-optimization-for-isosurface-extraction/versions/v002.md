# Dual Cell Optimization for Isosurface Extraction

    Also called: Optimization 1, Using Dual Cells at Same-level Regions

    ## Overview
    This method optimizes the hybrid implicit isosurface extraction process by replacing octants with dual cells in non-boundary (same-level) regions, significantly reducing the number of primitives traversed during ray tracing [S1]. It achieves an approximate 4× speedup in extraction time by leveraging the fact that an octant of a cell remains an octant of the corresponding dual cell, ensuring the final isosurface geometry is unchanged when eight octants are merged into a single dual cell [S1]. It builds on [Hybrid Implicit Isosurface Ray Tracing](11176-hybrid-implicit-isosurface-ray-tracing.md) and is distinct from [Active Cell Filtering](9505-active-cell-filtering.md), which filters input cells before primitive generation [S1].

    ## How it works
    **Inputs and Outputs:**
    - Input: A mesh of cells $C_i$ from TB-AMR (Tree-Based Adaptive Mesh Refinement) data, and a target isovalue [S1].
    - Output: A list of active cubic primitives (either dual cells $D_{ij}$ or octants $O_{ij}$) [S1].

    **Data Structures:**
    - **Cell $C_i$**: The input volume element stored in an octree [S1].
    - **Octant $O_{ij}$**: One of eight sub-cubes of $C_i$, indexed by $j \in \{0, \dots, 7\}$ [S1]. Vertices require reconstruction via the GTI method if at a boundary [S1].
    - **Dual Cell $D_{ij}$**: A cell enclosed by data points of known values [S1]. Its vertices are data points, requiring no reconstruction kernel [S1]. The center of $C_i$ is a vertex of $D_{ij}$ [S1].

    **Algorithm Steps:**
    1. Iterate through each input cell $C_i$ in the input mesh [S1].
    2. For each of the 8 octants $j$ of $C_i$:
       a. Construct the candidate dual cell $D_{ij}$ [S1].
       b. Determine if $D_{ij}$ lies in a same-level region [S1].
       c. If $D_{ij}$ is in a same-level region:
          i. Check if the isovalue lies within the value range of $D_{ij}$ [S1].
          ii. If true, and if $j=7$ (to avoid duplicate insertion for shared dual cells), insert $D_{ij}$ into `res` [S1].
       d. If $D_{ij}$ is not in a same-level region (i.e., $C_i$ is at a level boundary):
          i. Initialize the octant $O_{ij}$ [S1].
          ii. Compute vertex values for $O_{ij}$ using the GTI method [S1].
          iii. Check if the isovalue lies within the value range of $O_{ij}$ [S1].
          iv. If true, insert $O_{ij}$ into `res` [S1].

    **Pseudocode:**
    ```python
    def extract_cubic_primitives(input_cells, isovalue):
        res = []
        seen_dual_cells = set()
        for Ci in input_cells:
            for j in range(8):
                Dij = find_dual_cell(Ci, j)
                in_same_level = is_dual_cell_in_same_level(Dij)
                if in_same_level:
                    # Use a unique identifier to prevent duplicates in AMR grids
                    # where j=7 indexing may not be globally consistent [G]
                    if Dij.id not in seen_dual_cells and isovalue in Dij.value_range():
                        seen_dual_cells.add(Dij.id)
                        res.append(Dij)
                else:
                    Oij = init_octant(Ci, j)
                    gti_method(Oij)
                    if isovalue in Oij.value_range():
                        res.append(Oij)
        return res

    def find_dual_cell(Ci, j):
        # Returns the dual cell associated with octant j of cell Ci.
        # The dual cell corresponds to a vertex shared by up to 8 cells.
        # Its bounds are defined by the centers of the surrounding cells.
        vertex_coords = get_vertex_coords(Ci, j)
        neighbor_cells = get_neighbor_cells_at_vertex(vertex_coords)
        vertex_values = [cell.value for cell in neighbor_cells if cell is not None]
        return DualCell(id=vertex_coords, bounds=compute_bounds(neighbor_cells), vertex_values=vertex_values)

    def is_dual_cell_in_same_level(Dij):
        # Checks if all 8 cells sharing the dual cell's vertex are at the same AMR level.
        neighbor_cells = get_neighbor_cells_at_vertex(Dij.id)
        levels = [cell.level for cell in neighbor_cells if cell is not None]
        return len(set(levels)) == 1

    def init_octant(Ci, j):
        # Creates an octant of cell Ci at index j.
        # Bounds are half the cell dimensions from the center.
        half_dims = compute_half_dimensions(Ci)
        offset = get_octant_offset(j)
        return Octant(id=(Ci.id, j), bounds=compute_octant_bounds(Ci.center, half_dims, offset), vertex_values=[])

    def gti_method(Oij):
        # Computes vertex values for the octant using Grid Trilinear Interpolation.
        # Interpolates from the parent cell's center value and neighbor values.
        Oij.vertex_values = compute_trilinear_vertex_values(Oij.id)

    def get_vertex_coords(Ci, j):
        # Standard octant-to-vertex mapping [G]
        return tuple(Ci.center[i] + (-1 if j & (1 << i) else 1) * Ci.half_size for i in range(3))

    def get_neighbor_cells_at_vertex(vertex_coords):
        # Queries the octree for cells adjacent to the vertex [G]
        return [find_cell_at_offset(vertex_coords, offset) for offset in octant_offsets()]

    def compute_bounds(neighbor_cells):
        # Computes axis-aligned bounding box from neighbor centers [G]
        mins = [min(c.center[i] for c in neighbor_cells if c) for i in range(3)]
        maxs = [max(c.center[i] for c in neighbor_cells if c) for i in range(3)]
        return (tuple(mins), tuple(maxs))

    def compute_octant_bounds(center, half_dims, offset):
        # Computes bounds for a specific octant [G]
        mins = tuple(center[i] + offset[i] * half_dims[i] for i in range(3))
        maxs = tuple(center[i] - offset[i] * half_dims[i] for i in range(3))
        return (mins, maxs)

    def compute_trilinear_vertex_values(vertex_coords):
        # Standard trilinear interpolation from surrounding cell centers [G]
        return
    ```

    **Parameters and Edge Cases:**
    - **TB-AMR vs BS-AMR**: For BS-AMR (Brick-Based), cells are stored in uniform grids, making same-level checks trivial [S1]. For TB-AMR, cells are stored in an octree, and neighboring cell levels are not readily available [S1]. This requires checking if a dual cell lies in a same-level region during extraction, complicating the implementation [S1].
    - **Duplicate Insertion**: The condition `j == 7` prevents inserting the same dual cell multiple times when it is shared by adjacent parent cells [S1]. In AMR, this relies on consistent octant indexing; a vertex-coordinate lookup set is used in practice to guarantee manifold correctness [G].

    ## Operating characteristics
    - **Performance**: Achieves an approximate 4× speedup in extraction time compared to using octants exclusively [S1].
    - **Complexity**: Reduces the number of primitives traversed by ray tracing, improving rendering performance [S1].
    - **Limitations**: The optimization is specific to same-level regions. At level boundaries, octants must still be used with GTI reconstruction, which is more costly [S1].

    ## Background
    This optimization was introduced by Wang et al. in the context of CPU ray tracing of Tree-Based Adaptive Mesh Refinement (TB-AMR) data [S1]. While similar optimizations were applied to BS-AMR data in previous work [WWW∗19], the TB-AMR implementation required additional complexity due to the octree data layout [S1]. The method is part of the hybrid implicit isosurface rendering pipeline integrated into OSPRay [S1].

    ## Sources
    [S1] CPU Ray Tracing of Tree‐Based Adaptive Mesh Refinement Data, https://publications.sci.utah.edu/publications/Wan2020a/cgf.13958.pdf, fetched 2026-10-27
