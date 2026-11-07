# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions using Quadric Error Function (QEF) minimization for ambiguous cases, while using a 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) by replacing the standard linear interpolation of vertex positions with geometric optimization, specifically targeting the topological ambiguities inherent in the standard lookup table [S1, unofficial].

## How it works
The method processes voxel indices by splitting them into two groups based on their position in the grid to optimize performance: a "fast" group for non-ambiguous cases and a "slow" group for ambiguous cases requiring QEF optimization [S1, unofficial].

**Inputs and Outputs**
*   **Input**: A scalar field defined on a 3D grid of size $G_x \times G_y \times G_z$. Let $GRID = G_x$, $GRID2 = G_x G_y$, $GRID3 = G_x G_y G_z$. The input includes precomputed occupancy bits `occs` (binary values indicating if a vertex is inside/outside the isosurface), ambiguity check flags `amb_chks`, and lookup tables for axis directions `axd_tab`, axes `ax0_tab`, `ax1_tab`, and cases `cas_tab` derived from the parent's configuration [S1, unofficial].
*   **Output**: A list of vertex positions `val_pnts` and associated triangle indices forming the mesh.

**Data Structures**
*   `occs`: 3D array of integers (0 or 1) representing the binary occupancy of grid vertices.
*   `amb_chks`: 3D array of booleans indicating if a voxel configuration is one of the ambiguous cases (46, 47, 98, 99, 210, 213, 214, 215) [S1, unofficial].
*   `val_pnts`: Array of 3D vectors storing computed vertex positions.
*   `qvntab`, `pfvtab`, `vitogi`: Precomputed tables mapping case indices to QEF constraints (vertex indices, coordinates, edge indices) [S1, unofficial].

**Parameters**
*   `GRID`, `GRID2`, `GRID3`: Grid dimensions and strides.
*   `ERR`: Threshold for degenerate edge detection (e.g., $10^{-6}$) [S1, unofficial].
*   `QEF_REG`: Regularization weight for the QEF least-squares solver [S1, unofficial].
*   `SEP`: Separation constant for boundary handling [S1, unofficial].
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Search ranges for the 2D optimization [S1, unofficial].
*   `VERTICAL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Flags controlling search strategy [S1, unofficial].

**Algorithm Steps**

1.  **Filter Indices**: Separate voxel indices `pidx1` into two groups:
    *   Fast group: indices in range $[3 \cdot GRID3, 9 \cdot GRID3)$.
    *   QEF group: indices $\ge 9 \cdot GRID3$ [S1, unofficial].
    Initialize `start_idx` to 0.

2.  **Process Fast Group (Non-Ambiguous)**:
    For each index in the fast group:
    a.  Compute the 8-neighbor occupancy value `val_` by summing weighted occupancy bits:
        $$ val\_ = occs[pidx3] + 2 \cdot occs[pidx3 + axis1] + 4 \cdot occs[pidx3 + axis0] + 8 \cdot occs[pidx3 + axis2] $$
        where `axis0` and `axis1` are direction vectors from `ax0_tab` and `ax1_tab` [S1, unofficial].
    b.  Apply ambiguity correction: If `amb_chks[pidx3]` is true, set $val\_ = 15 - val\_$ [S1, unofficial].
    c.  Compute vertex positions using `_2d_search(pidx3, axdir, axis0, axis1, val_, cases)` [S1, unofficial].
    d.  Store results in `val_pnts` at `start_idx` and increment `start_idx` by the count of processed indices [S1, unofficial].

3.  **Process QEF Group (Ambiguous)**:
    For each index in the QEF group:
    a.  Call `_solve_QEF(pidx3, val_, cases)` where `cases` is derived from $pidx1 / GRID3 - 9$ [S1, unofficial].
    b.  Store results in `val_pnts` at `start_idx` [S1, unofficial].

**Detailed Subroutines**

**`_2d_search`**:
1.  Compute local coordinates `lcord` and `rcord` using the 2D lookup table `lctab_2d` [S1, unofficial].
2.  Calculate the midpoint `spnt = (lcord + rcord) / 2` [S1, unofficial].
3.  Compute the half-normal `hnorm` as the normalized difference between `lcord` and `rcord` [S1, unofficial].
4.  Compute the vertex normal `vnorm` by crossing the grid axes with `hnorm` [S1, unofficial].
5.  Adjust `vnorm` sign based on occupancy differences at the 8 neighbors to ensure it points towards the isosurface [S1, unofficial].
6.  Perform a vertical search along `vnorm` to find `left` [S1, unofficial].
7.  Perform horizontal searches along `hnorm` and `-hnorm` from `left` to find `lleft` and `rleft` [S1, unofficial].
8.  Compute the final offset `rslt` using the formula:
    $$ rslt = \frac{0.5 \cdot (rh + lh) \cdot hnorm + vl \cdot vnorm}{rh - lh} \cdot leng $$
    where $rh = \sum rleft \cdot hnorm$, $lh = \sum lleft \cdot hnorm$, and $vl = \sum (left - spnt) \cdot vnorm$ [S1, unofficial].
9.  If the result is degenerate or points in the wrong direction, return zero offset [S1, unofficial].
10. Return $spnt + rslt$ [S1, unofficial].

**`_solve_QEF`**:
1.  For each vertex index $i$ from 3 to 7:
    a.  Find indices `idx` where `qvntab[val_, cases] == i` [S1, unofficial].
    b.  Retrieve vertex coordinates `vtcs` from `pfvtab[val0_, cas, :2*i+1]` and edge indices `vitg` from `vitogi[vtcs]` [S1, unofficial].
    c.  Compute 3D positions `pnti` using grid offsets:
        $$ pnti = vitg[:,:,3] \cdot GRID3 + (pid3[:,0] + vitg[:,:,0]) \cdot GRID2 + (pid3[:,1] + vitg[:,:,1]) \cdot GRID + (pid3[:,2] + vitg[:,:,2]) $$
        Get global indices `pidx` and retrieve pre-computed vertex positions `vert = val_pnts[pidx]` [S1, unofficial].
    d.  Add local offsets `vitg[:,:3]` to `vert` [S1, unofficial].
2.  Compute QEF normals `qnrm` using cross products of edge vectors:
    $$ \_1vt = vert[:, 1::2] $$
    $$ qnrm = cross(\_1vt - vert[:, :-1:2], \_1vt - vert[:, 2::2]) $$
    Compute centroid $cntr = mean(\_1vt, dim=1)$ [S1, unofficial].
3.  Normalize `qnrm`, handling zero-length edges with `ERR` threshold [S1, unofficial].
4.  Solve the QEF system using least squares:
    $$ A = cat(qnrm, QEF\_REG \cdot I) $$
    $$ b = cat(\sum(\_1vt \cdot qnrm), QEF\_REG \cdot cntr) $$
    Solution $qefr = lstsq(A, b).solution$ [S1, unofficial].
    If all normals are degenerate, use $cntr$ instead [S1, unofficial].
5.  Compute ray direction $ctoq = qefr - cpnt$ and length $leng$ [S1, unofficial].
6.  Compute intersection parameter $t$ with constraint planes defined by `cstrtab`. Concatenate boundary parameters $(1-cpnt)/ctoq$ and $-cpnt/ctoq$ with $t$ [S1, unofficial].
7.  Handle NaNs and negative values by setting $t$ to $leng + SEP$, then normalize by $leng$ and clamp to $[0, 1]$ [S1, unofficial].
8.  Final vertex is $cpnt + t \cdot ctoq$ [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
The method splits computation into a fast path (2D search) and a slow path (QEF least squares). The fast path avoids the overhead of matrix inversion, making it suitable for non-ambiguous regions. The QEF path is computationally heavier due to the least-squares solve but ensures topological correctness in ambiguous cases [S1, unofficial]. QEM-based variants offer a trade-off, reducing register pressure and complexity at the cost of some geometric accuracy, which is acceptable for volume-preserving applications [S2, unofficial].

## Background
The method originates from the Open Data Cube (ODC) project, which implemented a GPU-accelerated Marching Cubes variant that uses QEF for ambiguous cases and 2D search for others [S1, unofficial]. The use of QEF for vertex optimization in surface extraction is a general technique, with QEM being a related metric used for mesh simplification and vertex relocation [S2, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
