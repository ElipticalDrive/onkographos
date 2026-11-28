# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions using a Quadric Error Function (QEF) minimization for ambiguous cases, while using a 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It builds on [Isosurface extraction](30-isosurface-extraction.md) by replacing the standard linear interpolation of vertex positions with an optimization step that minimizes geometric error, thereby reducing the "sliding vertex" artifacts common in standard Marching Cubes [S1, unofficial].

## How it works
The method operates on a regular 3D grid of voxels. It classifies cells into two groups based on their index to optimize performance: "fast" cells (indices in range $[3 \cdot G^3, 9 \cdot G^3)$) and "ambiguous" cells (indices $\ge 9 \cdot G^3$), where $G$ is the grid dimension [S1, unofficial].

### Data Structures
*   **`occs`**: A 3D array of binary occupancy values (0 or 1) at grid vertices.
*   **`val_pnts`**: A dynamic list or array storing computed vertex positions.
*   **`amb_chks`**: A boolean array indicating which cells have topological ambiguity requiring correction.
*   **`qvntab`**: A lookup table mapping occupancy patterns and cases to vertex indices.
*   **`pfvtab`**: A table storing pre-computed vertex coordinates and edge indices for specific cases.
*   **`vitogi`**: A table mapping vertex indices to edge indices.
*   **`cstrtab`**: A table defining constraint planes for ray intersection.
*   **`axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`**: Lookup tables for axis directions and case identifiers derived from cell indices.

### Algorithm Steps

1.  **Filter and Classify**:
    Identify active cell indices `pidx1`. Split them into:
    *   `pidx_fast`: indices where $3 \cdot G^3 \le \text{pidx1} < 9 \cdot G^3$.
    *   `pidx_slow`: indices where $\text{pidx1} \ge 9 \cdot G^3$.

2.  **Process Fast Cells (Non-Ambiguous)**:
    For each cell in `pidx_fast`:
    a.  Compute the 8-neighbor occupancy value `val_` by summing weighted occupancy bits:
        $$ \text{val\_} = \text{occs}[pidx3] + 2 \cdot \text{occs}[pidx3 + \text{axis1}] + 4 \cdot \text{occs}[pidx3 + \text{axis0}] + 8 \cdot \text{occs}[pidx3 + \text{axis2}] $$
        where `axis0`, `axis1`, `axis2` are direction vectors for the cell's local axes.
    b.  Apply ambiguity correction: If `amb_chks[pidx3]` is true, set $\text{val\_} = 15 - \text{val\_}$.
    c.  Compute vertex positions using a 2D search function `_2d_search(pidx3, axdir, axis0, axis1, val_, cases)` [S1, unofficial].
    d.  Store results in `val_pnts` at `start_idx` and increment `start_idx` by the count of fast cells.

3.  **Process Slow Cells (Ambiguous/QEF)**:
    For each cell in `pidx_slow`:
    a.  Determine the case identifier `cases` from `pidx1 / G^3 - 9`.
    b.  Call `_solve_QEF(pidx3, val_, cases)` to compute optimized vertex positions [S1, unofficial].
    c.  Store results in `val_pnts` at `start_idx`.

4.  **QEF Solving (`_solve_QEF`)**:
    For each vertex index $i$ from 3 to 7:
    a.  Find indices `idx` where `qvntab[val_, cases] == i`.
    b.  Retrieve vertex coordinates `vtcs` from `pfvtab[val0_, cas, :2*i+1]` and edge indices `vitg` from `vitogi[vtcs]`.
    c.  Compute 3D positions `pnti` using grid offsets:
        $$ \text{pnti} = \text{vitg}[:,:,3] \cdot G^3 + (\text{pid3}[:,0]+\text{vitg}[:,:,0]) \cdot G^2 + (\text{pid3}[:,1]+\text{vitg}[:,:,1]) \cdot G + (\text{pid3}[:,2]+\text{vitg}[:,:,2]) $$
    d.  Get global indices `pidx` and retrieve pre-computed vertex positions `vert = val_pnts[pidx]`.
    e.  Add local offsets `vitg[:,:3]` to `vert`.
    f.  Compute QEF normals `qnrm` using cross products of edge vectors:
        $$ \_1vt = \text{vert}[:, 1::2] $$
        $$ \text{qnrm} = \text{cross}(\_1vt - \text{vert}[:, :-1:2], \_1vt - \text{vert}[:, 2::2]) $$
    g.  Compute centroid `cntr = mean(_1vt, dim=1)`.
    h.  Normalize `qnrm`, handling zero-length edges with a threshold `ERR` [S1, unofficial].
    i.  Solve the QEF system using least squares:
        $$ A = \text{cat}(\text{qnrm}, \text{QEF\_REG} \cdot I) $$
        $$ b = \text{cat}(\sum(\_1vt \cdot \text{qnrm}), \text{QEF\_REG} \cdot \text{cntr}) $$
        $$ \text{qefr} = \text{lstsq}(A, b).\text{solution} $$
        If all normals are degenerate (`cndt` all true), use `cntr` instead [S1, unofficial].
    j.  Compute ray direction `ctoq = qefr - cpnt` and length `leng`.
    k.  Compute intersection parameter `t` with constraint planes defined by `cstrtab`. Concatenate boundary parameters $(1-\text{cpnt})/\text{ctoq}$ and $-\text{cpnt}/\text{ctoq}$ with `t`.
    l.  Handle NaNs and negative values by setting `t` to `leng + SEP`, then normalize by `leng` and clamp to $[0, 1]$.
    m.  Final vertex is $\text{cpnt} + t \cdot \text{ctoq}$ [S1, unofficial].

5.  **2D Search (`_2d_search`)**:
    For fast cells, this function performs a 2D linear/binomial search to determine vertex positions [S1, unofficial]. It calculates a midpoint `spnt`, determines a normal `vnorm` by checking occupancy changes in neighboring voxels, and solves for the intersection along the normal and horizontal directions using linear search ranges `VERTICAL_RANGE`, `HORIZNTL_RANGE` and binary search flags [S1, unofficial].

### Parameters
*   `ERR`: Threshold for zero-length edge detection.
*   `QEF_REG`: Regularization parameter for the QEF least squares solver.
*   `SEP`: Separation constant for handling degenerate ray intersections.
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Search ranges for the 2D linear search.
*   `VERTICAL_LINEAR_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Flags controlling the search strategy.

### Edge Cases and Pitfalls
*   **Degenerate Normals**: If all QEF normals are zero-length, the solver falls back to the centroid [S1, unofficial].
*   **Ray Intersection NaNs**: If the ray direction `ctoq` is zero or parallel to constraint planes, `t` is handled by clamping and normalization [S1, unofficial].
*   **Ambiguity Correction**: The method explicitly corrects for topological ambiguities by inverting the occupancy value `val_` when `amb_chks` is true [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The method splits processing into two paths. The "fast" path uses 2D search, which is $O(1)$ per cell but involves multiple lookups and arithmetic. The "slow" path uses QEF minimization, which involves solving a least squares system, making it more expensive per cell. The overall complexity is linear with respect to the number of active cells, $O(N)$, but with a higher constant factor for ambiguous regions.
*   **Memory Complexity**: Requires storage for `val_pnts`, lookup tables (`qvntab`, `pfvtab`, etc.), and intermediate arrays for normals and centroids. Memory usage scales with the number of vertices and the size of the lookup tables.
*   **Performance**: The split strategy allows for optimized handling of non-ambiguous cases, which are typically more frequent, while reserving the expensive QEF optimization for ambiguous cases [S1, unofficial]. QEM-based variants offer better performance on parallel hardware by reducing register pressure [S2, unofficial].
*   **Strengths**: Produces watertight meshes with reduced sliding vertex artifacts compared to standard Marching Cubes [S1, unofficial]. Handles topological ambiguities correctly [S1, unofficial].
*   **Limitations**: More complex to implement than standard Marching Cubes. The QEF optimization is computationally intensive for ambiguous cases.
*   **When to use**: When high-quality surface reconstruction is required, especially in regions with topological ambiguity, and computational resources allow for the additional cost.
*   **When not to use**: When real-time performance is critical and the visual improvement over standard Marching Cubes is not worth the computational cost, or when using hardware with severe register constraints (consider QEM variant [S2, unofficial]).

## Background
The method originates from the Open Data Cube (ODC) project, specifically the `KAIST-Visual-AI-Group/ODC` repository [S1, unofficial]. It was developed to provide efficient and accurate surface extraction for large-scale voxel data. The QEF approach is inspired by the work of Garland and Heckbert on Quadric Error Metrics for mesh simplification [S2, unofficial], adapted here for vertex position optimization in isosurface extraction.

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
