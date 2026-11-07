# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions using a Quadric Error Function (QEF) minimization for ambiguous cases, while using a 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It builds on [Isosurface extraction](30-isosurface-extraction.md) by replacing the standard linear interpolation of classic Marching Cubes with a constrained optimization step for topologically ambiguous configurations, thereby reducing geometric artifacts while maintaining the topological correctness of the base algorithm.

## How it works
The method operates on a regular 3D grid of occupancy values. It classifies grid cells based on their index to determine the optimization strategy: cells with indices in the range $[3 \cdot G^3, 9 \cdot G^3)$ are processed using a fast 2D search, while cells with indices $\ge 9 \cdot G^3$ are processed using QEF minimization [S1, unofficial].

### Data Structures
*   **Grid Dimensions**: $G$ is the grid size in each dimension. Total cells $N = G^3$.
*   **Occupancy Array**: `occs` of shape $(G, G, G)$, containing binary values (0 or 1) representing voxel occupancy.
*   **Vertex Buffer**: `val_pnts`, a dynamic array storing computed vertex positions (3D floats).
*   **Lookup Tables**:
    *   `amb_chks`: Boolean array indicating ambiguous configurations.
    *   `axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`: Precomputed tables mapping cell indices to axis directions, basis vectors, and case numbers.
    *   `qvntab`, `pfvtab`, `vitogi`, `cstrtab`: Tables defining vertex indices, coordinates, edge indices, and constraint planes for QEF solving.

### Algorithm Steps

1.  **Classification**:
    Filter the set of active cell indices `sel_pidx1` into two groups:
    *   Group 1 (Fast): `pidx1` where $3 \cdot G^3 \le \text{pidx1} < 9 \cdot G^3$.
    *   Group 2 (QEF): `pidx1` where $\text{pidx1} \ge 9 \cdot G^3$.

2.  **Processing Group 1 (2D Search)**:
    For each cell index in Group 1:
    a.  Retrieve precomputed `axdir`, `axis0`, `axis1`, and `cases` from lookup tables using `csidx = floor(pidx1 / G^3) - 3`.
    b.  Compute the 8-neighbor occupancy pattern `val_`:
        $$
        \text{val\_} = \text{occs}[p] + 2 \cdot \text{occs}[p + \text{axis1}] + 4 \cdot \text{occs}[p + \text{axis0}] + 8 \cdot \text{occs}[p + \text{axis2}]
        $$
        where $p$ is the 3D coordinate of the cell corner, and $\text{axis2} = \text{axis0} + \text{axis1}$.
    c.  **Ambiguity Correction**: If `amb_chks[p]` is true, set $\text{val\_} = 15 - \text{val\_}$.
    d.  Compute vertex positions using `_2d_search(p, axdir, axis0, axis1, val_, cases)`.
    e.  Store results in `val_pnts` and update the start index.

3.  **Processing Group 2 (QEF Optimization)**:
    For each cell index in Group 2:
    a.  Compute `val_` and apply ambiguity correction as in Step 2b-c.
    b.  Call `_solve_QEF(p, val_, cases)` to compute optimized vertex positions.
    c.  Store results in `val_pnts`.

4.  **QEF Solving (`_solve_QEF`)**:
    For each vertex index $i$ associated with the cell configuration:
    a.  Retrieve vertex coordinates `vtcs` and edge indices `vitg` from lookup tables.
    b.  Compute 3D positions `pnti` using grid offsets and global indices.
    c.  Retrieve pre-computed vertex positions `vert` from `val_pnts`.
    d.  Compute QEF normals `qnrm` using cross products of edge vectors:
        $$
        \text{qnrm} = \text{cross}(\text{vert}_{1::2} - \text{vert}_{:-1:2}, \text{vert}_{1::2} - \text{vert}_{2::2})
        $$
    e.  Compute centroid `cntr` of the vertex ring.
    f.  Normalize `qnrm`, handling zero-length edges with a threshold `ERR`.
    g.  Solve the QEF system using least squares:
        $$
        A = \text{concat}(\text{qnrm}, \lambda \cdot I), \quad b = \text{concat}(\sum (\text{vert} \cdot \text{qnrm}), \lambda \cdot \text{cntr})
        $$
        where $\lambda$ is `QEF_REG`.
    h.  If all normals are degenerate, use `cntr` as the solution `qefr`.
    i.  Compute ray direction `ctoq = qefr - cpnt` and length `leng`.
    j.  Compute intersection parameter `t` with constraint planes defined by `cstrtab`.
    k.  Handle NaNs and negative values by setting `t` to `leng + SEP`, then normalize by `leng` and clamp to $[0, 1]$.
    l.  Final vertex is `cpnt + t * ctoq`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def solve_qef_vertex_optimization(occs, G, GRID3, GRID2, GRID, ERR, QEF_REG, SEP, 
                                  VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                                  HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH,
                                  axd_tab, ax0_tab, ax1_tab, cas_tab, amb_chks, qvntab, pfvtab, vitogi, cstrtab):
    """
    Main function for Marching Cubes with QEF Vertex Optimization.
    """
    val_pnts = []  # List of vertices
    start_idx = 0
    
    # Identify active cells (simplified: assume all cells are active for demonstration)
    # In practice, sel_pidx1 would be derived from occupancy changes
    sel_pidx1 = torch.arange(G**3)
    
    # Split into groups
    mask_fast = (sel_pidx1 >= 3 * G**3) & (sel_pidx1 < 9 * G**3)
    mask_qef = sel_pidx1 >= 9 * G**3
    
    pidx1_fast = sel_pidx1[mask_fast]
    pidx1_qef = sel_pidx1[mask_qef]
    
    # Process Fast Group
    if pidx1_fast.numel() > 0:
        count_idx = pidx1_fast.shape[0]
        csidx = torch.div(pidx1_fast, G**3, rounding_mode='floor') - 3
        axdir = axd_tab[csidx]
        axis0 = ax0_tab[csidx]
        axis1 = ax1_tab[csidx]
        cases = cas_tab[csidx]
        axis2 = axis0 + axis1
        
        pidx3 = pidx1_to_pidx3(pidx1_fast)
        
        # Compute occupancy pattern
        val_ = (occs[pidx3[:, 0], pidx3[:, 1], pidx3[:, 2]] +
                2 * occs[pidx3[:, 0] + axis1[:, 0], pidx3[:, 1] + axis1[:, 1], pidx3[:, 2] + axis1[:, 2]] +
                4 * occs[pidx3[:, 0] + axis0[:, 0], pidx3[:, 1] + axis0[:, 1], pidx3[:, 2] + axis0[:, 2]] +
                8 * occs[pidx3[:, 0] + axis2[:, 0], pidx3[:, 1] + axis2[:, 1], pidx3[:, 2] + axis2[:, 2]])
        
        # Ambiguity correction
        tmp = amb_chks[pidx3]
        val_ = torch.where(tmp, 15 - val_, val_)
        
        # Compute vertices
        new_verts = _2d_search(pidx3, axdir, axis0, axis1, val_, cases,
                               VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                               HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH,
                               axd_tab, ax0_tab, ax1_tab, cas_tab, amb_chks, qvntab, pfvtab, vitogi, cstrtab)
        
        val_pnts.extend(new_verts)
        start_idx += count_idx
        
    # Process QEF Group
    if pidx1_qef.numel() > 0:
        csidx = torch.div(pidx1_qef, G**3, rounding_mode='floor') - 9
        axdir = axd_tab[csidx]
        axis0 = ax0_tab[csidx]
        axis1 = ax1_tab[csidx]
        cases = cas_tab[csidx]
        axis2 = axis0 + axis1
        
        pidx3 = pidx1_to_pidx3(pidx1_qef)
        
        val_ = (occs[pidx3[:, 0], pidx3[:, 1], pidx3[:, 2]] +
                2 * occs[pidx3[:, 0] + axis1[:, 0], pidx3[:, 1] + axis1[:, 1], pidx3[:, 2] + axis1[:, 2]] +
                4 * occs[pidx3[:, 0] + axis0[:, 0], pidx3[:, 1] + axis0[:, 1], pidx3[:, 2] + axis0[:, 2]] +
                8 * occs[pidx3[:, 0] + axis2[:, 0], pidx3[:, 1] + axis2[:, 1], pidx3[:, 2] + axis2[:, 2]])
        
        tmp = amb_chks[pidx3]
        val_ = torch.where(tmp, 15 - val_, val_)
        
        new_verts = _solve_QEF(pidx3, val_, cases, axd_tab, ax0_tab, ax1_tab, cas_tab, 
                               qvntab, pfvtab, vitogi, cstrtab, ERR, QEF_REG, SEP)
        
        val_pnts.extend(new_verts)
        
    return val_pnts

def _2d_search(pidx3, axdir, axis0, axis1, val_, cases, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
               HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH, axd_tab, ax0_tab, ax1_tab, cas_tab, amb_chks, qvntab, pfvtab, vitogi, cstrtab):
    """
    Performs 2D linear/binomial search for non-ambiguous cases.
    """
    # Implementation details from [S1, unofficial]
    # This function computes vertex positions using 2D search heuristics
    # for cells in the fast group.
    # Note: The exact implementation involves complex tensor operations
    # as described in the source code snippet.
    # For brevity, we assume this function returns a list of 3D points.
    pass

def _solve_QEF(pidx3, val_, cases, axd_tab, ax0_tab, ax1_tab, cas_tab, qvntab, pfvtab, vitogi, cstrtab, ERR, QEF_REG, SEP):
    """
    Solves the QEF for ambiguous cases.
    """
    # Implementation details from [S1, unofficial]
    # This function computes vertex positions using QEF minimization
    # for cells in the QEF group.
    # Note: The exact implementation involves complex tensor operations
    # as described in the source code snippet.
    # For brevity, we assume this function returns a list of 3D points.
    pass

def pidx1_to_pidx3(pidx1):
    """
    Converts 1D index to 3D coordinates.
    """
    z = pidx1 % G
    y = (pidx1 // G) % G
    x = pidx1 // (G * G)
    return torch.stack([x, y, z], dim=1)
```

### Parameters
*   `ERR`: Threshold for handling degenerate edges/normals.
*   `QEF_REG`: Regularization parameter for QEF least squares.
*   `SEP`: Separation parameter for handling NaNs/negative values.
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Ranges for linear/binomial search.
*   `VERTICAL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Flags for search type.

### Edge Cases and Pitfalls
*   **Degenerate Normals**: If all QEF normals are degenerate (zero length), the centroid is used as the solution [S1, unofficial].
*   **NaNs and Negative Values**: Handled by setting the intersection parameter `t` to `leng + SEP` before normalization and clamping [S1, unofficial].
*   **Ambiguity Correction**: Critical for topological correctness; failure to apply can result in holes or artifacts [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The method introduces a least-squares solve for ambiguous cases, which is more expensive than linear interpolation but less expensive than full dual contouring for every vertex. The fast 2D search group maintains near-linear complexity for non-ambiguous cases.
*   **Memory Complexity**: Requires additional lookup tables for QEF constraints and precomputed vertex positions.
*   **Performance**: QEM-based variants offer better performance on parallel hardware due to lower register usage [S2, unofficial].
*   **Strengths**: Produces watertight meshes with reduced geometric artifacts in ambiguous regions.
*   **Limitations**: More complex implementation than standard Marching Cubes. Performance depends on the proportion of ambiguous cases.

## Background
The method originates from the ODC (Octree-based Dual Contouring) project by KAIST Visual AI Group [S1, unofficial]. It combines the topological robustness of Marching Cubes with the geometric accuracy of QEF minimization, addressing the limitations of linear interpolation in ambiguous configurations. The QEM-based variant was developed to optimize performance on parallel hardware for terrain visualization [S2, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
