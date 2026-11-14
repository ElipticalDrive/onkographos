# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions using a Quadric Error Function (QEF) minimization for ambiguous cases, while using a 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It sets itself apart from standard Marching Cubes by resolving topological ambiguities and vertex placement errors through geometric optimization rather than simple linear interpolation, and differs from QEM-based relocation by solving a full least-squares system for higher accuracy at the cost of increased register usage [S1, unofficial][S2, unofficial].

## How it works
The method processes a voxel grid by splitting ambiguous and non-ambiguous cases into separate processing pipelines. It relies on the precomputed data structures from [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md).

**Inputs:**
*   `occs`: A 3D boolean or integer array representing voxel occupancy or scalar field signs.
*   `GRID`, `GRID2`, `GRID3`: Grid spacing constants ($1, S, S^2$).
*   `ERR`: A small threshold for degenerate edge detection.
*   `QEF_REG`: Regularization parameter for the QEF least-squares solver.
*   `SEP`: Separation constant for boundary constraints.
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Search ranges for the 2D solver.
*   `VERTICAL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Flags controlling search modes.
*   `lctab_2d`, `axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`: Precomputed lookup tables for non-ambiguous cases.
*   `qvntab`, `pfvtab`, `vitogi`, `cstrtab`: Precomputed tables for ambiguous cases (QEF).

**Outputs:**
*   `val_pnts`: An array of vertex positions.
*   `start_idx`: The starting index in `val_pnts` for the current batch.

**Data Structures:**
*   `pidx1`: 1D array of linear voxel indices to process.
*   `pidx3`: 3D coordinates derived from `pidx1`.
*   `val_`: Integer occupancy pattern (0-255) for the current voxel.

**Steps:**

1.  **Filtering**: Split `pidx1` into two groups:
    *   Group 1 (Non-ambiguous): Indices in range $[3 \cdot GRID3, 9 \cdot GRID3)$.
    *   Group 2 (Ambiguous): Indices $\ge 9 \cdot GRID3$ [S1, unofficial].

2.  **Non-Ambiguous Processing (Group 1)**:
    *   Calculate `val_` by summing weighted occupancy bits:
        $$ val\_ = occs[pidx3] + 2 \cdot occs[pidx3 + axis1] + 4 \cdot occs[pidx3 + axis0] + 8 \cdot occs[pidx3 + axis2] $$
        where `axis0`, `axis1`, `axis2` are direction vectors from `ax0_tab`, `ax1_tab` [S1, unofficial].
    *   Apply ambiguity correction: If `amb_chks[pidx3]` is true, set $val\_ = 15 - val\_$ [S1, unofficial].
    *   Compute vertex positions using `_2d_search(pidx3, axdir, axis0, axis1, val_, cases)` [S1, unofficial].
    *   Store results in `val_pnts` at `start_idx` and increment `start_idx` by the count of processed voxels [S1, unofficial].

3.  **Ambiguous Processing (Group 2)**:
    *   Call `_solve_QEF(pidx3, val_, cases)` where `cases` is derived from `pidx1 / GRID3 - 9` [S1, unofficial].
    *   Store results in `val_pnts` at `start_idx` [S1, unofficial].

**Subroutine: `_2d_search`**
1.  Retrieve lookup table result `lcres = lctab_2d[val_, cases + 1]` [S1, unofficial].
2.  Compute intermediate indices `cidx3` and linear index `cidx1` based on `lcres` and direction `axdir` [S1, unofficial].
3.  Retrieve linear vertex `linv = val_pnts[get_idx(cidx1)]` [S1, unofficial].
4.  Compute reference coordinates `rcord` based on `linv` and axes `axis0`, `axis1` [S1, unofficial].
5.  Compute midpoint `spnt = (lcord + rcord) / 2` [S1, unofficial].
6.  Calculate horizontal normal `hnorm` and vertical normal `vnorm` using cross products and occupancy checks against neighbors `occ0..occ3` [S1, unofficial].
7.  Perform vertical search `_2d_linbin` to find `left` [S1, unofficial].
8.  Perform horizontal searches to find `lleft` and `rleft` [S1, unofficial].
9.  Compute result `rslt` using the derived normals and distances, handling degenerate cases (`cndt`) by zeroing the result [S1, unofficial].
10. Return `spnt + rslt` [S1, unofficial].

**Subroutine: `_solve_QEF`**
1.  For each vertex index `i` from 3 to 7:
    *   Find indices `idx` where `qvntab[val_, cases] == i` [S1, unofficial].
    *   Retrieve vertex coordinates `vtcs` from `pfvtab[val0_, cas, :2*i+1]` and edge indices `vitg` from `vitogi[vtcs]` [S1, unofficial].
2.  Compute 3D positions `pnti` using grid offsets:
    $$ pnti = vitg[:,:,3] \cdot GRID3 + (pid3[:,0]+vitg[:,:,0]) \cdot GRID2 + (pid3[:,1]+vitg[:,:,1]) \cdot GRID + (pid3[:,2]+vitg[:,:,2]) $$
    Retrieve global indices `pidx` and pre-computed positions `vert = val_pnts[pidx]` [S1, unofficial].
3.  Add local offsets `vitg[:,:3]` to `vert` [S1, unofficial].
4.  Compute QEF normals `qnrm` using cross products of edge vectors:
    $$ \_1vt = vert[:, 1::2] $$
    $$ qnrm = cross(\_1vt - vert[:, :-1:2], \_1vt - vert[:, 2::2]) $$
    Compute centroid $cntr = mean(\_1vt, dim=1)$ [S1, unofficial].
5.  Normalize `qnrm`, handling zero-length edges with `ERR` threshold [S1, unofficial].
6.  Solve the QEF system using least squares:
    $$ A = cat(qnrm, QEF\_REG \cdot I) $$
    $$ b = cat(sum(\_1vt \cdot qnrm), QEF\_REG \cdot cntr) $$
    Solution $qefr = lstsq(A, b).solution$ [S1, unofficial].
    If all normals are degenerate (`cndt` all true), use $cntr$ instead [S1, unofficial].
7.  Compute ray direction $ctoq = qefr - cpnt$ and length $leng$ [S1, unofficial].
8.  Compute intersection parameter $t$ with constraint planes defined by `cstrtab`. Concatenate boundary parameters $(1-cpnt)/ctoq$ and $-cpnt/ctoq$ with $t$ [S1, unofficial].
9.  Handle NaNs and negative values by setting $t$ to $leng + SEP$, then normalize by $leng$ and clamp to $[0, 1]$ [S1, unofficial].
10. Final vertex is $cpnt + t \cdot ctoq$ [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Dominated by the QEF least-squares solve for ambiguous cases and the 2D search for non-ambiguous cases. The QEF step involves matrix construction and solving, which is more expensive than linear interpolation but constant per vertex.
*   **Memory Complexity**: Requires storage for precomputed tables (`qvntab`, `pfvtab`, etc.) and the vertex array `val_pnts`. The QEF solver requires temporary storage for matrices $A$ and $b$.
*   **Performance**: The QEF approach is computationally heavier than standard Marching Cubes due to the least-squares solve, but provides higher geometric accuracy for ambiguous cases [S1, unofficial][S2, unofficial]. QEM-based variants offer a trade-off with lower register usage and better parallel performance [S2, unofficial].
*   **Strengths**: Produces watertight meshes with accurate vertex placement in ambiguous regions. Handles complex topologies better than standard linear interpolation.
*   **Limitations**: Higher computational cost. Requires careful handling of degenerate cases and boundary constraints. Register usage can be high on GPU hardware for the QEF solver [S2, unofficial].
*   **When to use**: When high geometric accuracy is required, especially in regions with topological ambiguities.
*   **When not to use**: When performance is critical and approximate geometry is acceptable, or when hardware register limits are a bottleneck (consider QEM variant).

## Background
The method originates from the ODC (Octree-based Densification and Compression) project by the KAIST Visual AI Group, which implements a hybrid approach using 2D search for non-ambiguous cases and QEF minimization for ambiguous cases [S1, unofficial]. The Quadric Error Function (QEF) concept was introduced by Garland and Heckbert for mesh simplification, and its application to isosurface extraction was adapted to improve vertex accuracy [S2, unofficial]. The QEM-based variant was proposed to address performance penalties on parallel hardware due to register limits associated with full QEF minimization [S2, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
