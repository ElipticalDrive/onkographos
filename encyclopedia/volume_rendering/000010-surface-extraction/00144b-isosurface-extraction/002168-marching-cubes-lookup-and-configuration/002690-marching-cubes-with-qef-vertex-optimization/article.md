# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions using a Quadric Error Function (QEF) minimization for ambiguous cases, while using a 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It builds on [Isosurface extraction](30-isosurface-extraction.md) by replacing the standard linear interpolation of vertex positions with a constrained optimization that respects the local geometry of the isosurface, thereby reducing artifacts in complex topologies [S1, unofficial].

## How it works
The method operates on a regular 3D grid of voxels. It classifies cells based on their index to determine the optimization strategy: cells with indices in the range $[3 \cdot G^3, 9 \cdot G^3)$ use a fast 2D search, while cells with indices $\ge 9 \cdot G^3$ use QEF minimization [S1, unofficial].

### Data Structures
*   **Grid Dimensions**: $G$ is the grid size in each dimension. Total cells $N = G^3$.
*   **Occupancy Array**: `occs` of shape $(G, G, G)$, containing binary values (0 or 1) indicating voxel occupancy.
*   **Ambiguity Check Array**: `amb_chks` of shape $(G, G, G)$, boolean flags indicating if a cell has an ambiguous configuration requiring correction.
*   **Lookup Tables**:
    *   `axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`: Precomputed tables mapping cell indices to axis directions and case numbers.
    *   `lctab_2d`: Lookup table for 2D search results.
    *   `qvntab`, `pfvtab`, `vitogi`: Tables used for QEF vertex coordinate and edge index retrieval.
    *   `cstrtab`: Constraint plane parameters for ray intersection.

### Algorithm Steps

1.  **Filter and Classify Cells**:
    Iterate over all active cell indices `pidx1`. Split them into two groups:
    *   Group A (Fast): $3 \cdot G^3 \le \text{pidx1} < 9 \cdot G^3$.
    *   Group B (QEF): $\text{pidx1} \ge 9 \cdot G^3$.

2.  **Process Group A (2D Search)**:
    For each cell in Group A:
    a.  Compute the 8-neighbor occupancy value `val_` by summing weighted occupancy bits:
        $$ \text{val\_} = \text{occs}[p_0, p_1, p_2] + 2 \cdot \text{occs}[p_0 + \Delta_1, p_1 + \Delta_2, p_2 + \Delta_3] + 4 \cdot \text{occs}[p_0 + \Delta_4, p_1 + \Delta_5, p_2 + \Delta_6] + 8 \cdot \text{occs}[p_0 + \Delta_7, p_1 + \Delta_8, p_2 + \Delta_9] $$
        *(Note: The exact bit weights and neighbor offsets depend on the specific grid traversal order and axis definitions `axis0`, `axis1`, `axis2` derived from `axd_tab`)* [S1, unofficial].
    b.  Apply ambiguity correction: If `amb_chks[pidx3]` is true, set $\text{val\_} = 15 - \text{val\_}$ [S1, unofficial].
    c.  Compute vertex positions using `_2d_search(pidx3, axdir, axis0, axis1, val_, cases)` [S1, unofficial].
        *   The 2D search involves finding a starting point `spnt` and a normal `vnorm`.
        *   It performs a linear/binomial search along the normal direction to find the intersection.
        *   It refines the position using horizontal searches along edge vectors `hnorm` [S1, unofficial].
    d.  Store results in `val_pnts` at `start_idx`, then increment `start_idx` by the count of processed cells [S1, unofficial].

3.  **Process Group B (QEF Optimization)**:
    For each cell in Group B:
    a.  Call `_solve_QEF(pidx3, val_, cases)` where `val_` is the integer occupancy pattern and `cases` is derived from $\text{pidx1} / G^3 - 9$ [S1, unofficial].
    b.  Inside `_solve_QEF`:
        i.  For each vertex index $i$ from 3 to 7, find indices `idx` where `qvntab[val_, cases] == i` [S1, unofficial].
        ii. Retrieve vertex coordinates `vtcs` from `pfvtab[val0_, cas, :2*i+1]` and edge indices `vitg` from `vitogi[vtcs]` [S1, unofficial].
        iii. Compute 3D positions `pnti` for the vertices using grid offsets:
            $$ \text{pnti} = \text{vitg}[:,:,3] \cdot G^3 + (\text{pid3}[:,0]+\text{vitg}[:,:,0]) \cdot G^2 + (\text{pid3}[:,1]+\text{vitg}[:,:,1]) \cdot G + (\text{pid3}[:,2]+\text{vitg}[:,:,2]) $$
            Get global indices `pidx` and retrieve pre-computed vertex positions `vert = val_pnts[pidx]`. Add local offsets `vitg[:,:3]` to `vert` [S1, unofficial].
        iv. Compute QEF normals `qnrm` using cross products of edge vectors:
            $$ \text{qnrm} = \text{cross}(\text{vert}[:, 1::2] - \text{vert}[:, :-1:2], \text{vert}[:, 1::2] - \text{vert}[:, 2::2]) $$
            Compute centroid `cntr = mean(vert[:, 1::2], dim=1)` [S1, unofficial].
            Normalize `qnrm` handling zero-length edges with `ERR` threshold [S1, unofficial].
        v.  Solve the QEF system using least squares:
            $$ A = \text{cat}(\text{qnrm}, \text{QEF\_REG} \cdot I) $$
            $$ b = \text{cat}(\sum(\text{vert}[:, 1::2] \cdot \text{qnrm}), \text{QEF\_REG} \cdot \text{cntr}) $$
            Solution $\text{qefr} = \text{lstsq}(A, b).\text{solution}$. If all normals are degenerate (`cndt` all true), use `cntr` instead [S1, unofficial].
        vi. Compute ray direction $\text{ctoq} = \text{qefr} - \text{cpnt}$ and length `leng`.
        vii. Compute intersection parameter $t$ with constraint planes defined by `cstrtab`. Concatenate boundary parameters $(1-\text{cpnt})/\text{ctoq}$ and $-\text{cpnt}/\text{ctoq}$ with $t$. Handle NaNs and negative values by setting $t$ to $\text{leng} + \text{SEP}$, then normalize by `leng` and clamp to $[0, 1]$. Final vertex is $\text{cpnt} + t \cdot \text{ctoq}$ [S1, unofficial].

4.  **Output**:
    The resulting `val_pnts` array contains the optimized vertex positions for all cells. These vertices are connected according to the Marching Cubes topology tables to form the final mesh.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_qef(occs, amb_chks, G, GRID3, GRID2, GRID, ERR, QEF_REG, SEP, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH, HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH):
    """
    Performs Marching Cubes with QEF Vertex Optimization.
    
    Args:
        occs: 3D array of occupancy values (0 or 1).
        amb_chks: 3D array of boolean ambiguity flags.
        G: Grid size.
        GRID3, GRID2, GRID: Precomputed powers of G (G^3, G^2, G).
        ERR: Epsilon for numerical stability.
        QEF_REG: Regularization factor for QEF.
        SEP: Separation constant for ray intersection.
        VERTICAL/HORIZNTL_*: Parameters for 2D search.
        
    Returns:
        val_pnts: Array of vertex positions.
    """
    # Initialize lookup tables (assumed precomputed)
    axd_tab, ax0_tab, ax1_tab, cas_tab, lctab_2d, qvntab, pfvtab, vitogi, cstrtab = get_lookup_tables()
    
    # Get all active cell indices (simplified for demonstration)
    # In practice, this involves iterating over the grid and identifying cells intersected by the isosurface
    all_pidx1 = get_active_cell_indices(occs, G)
    
    # Split into groups
    mask_a = (all_pidx1 >= 3 * GRID3) & (all_pidx1 < 9 * GRID3)
    mask_b = all_pidx1 >= 9 * GRID3
    
    pidx1_a = all_pidx1[mask_a]
    pidx1_b = all_pidx1[mask_b]
    
    val_pnts = [] # List to store vertices
    
    # Process Group A: Fast 2D Search
    if len(pidx1_a) > 0:
        count_idx = len(pidx1_a)
        csidx = (pidx1_a // GRID3) - 3
        axdir = axd_tab[csidx]
        axis0 = ax0_tab[csidx]
        axis1 = ax1_tab[csidx]
        cases = cas_tab[csidx]
        axis2 = axis0 + axis1
        
        pidx3 = pidx1_to_pidx3(pidx1_a)
        
        # Compute occupancy value
        val_ = (occs[pidx3[:, 0], pidx3[:, 1], pidx3[:, 2]] +
                2 * occs[pidx3[:, 0] + axis1[:, 0], pidx3[:, 1] + axis1[:, 1], pidx3[:, 2] + axis1[:, 2]] +
                4 * occs[pidx3[:, 0] + axis0[:, 0], pidx3[:, 1] + axis0[:, 1], pidx3[:, 2] + axis0[:, 2]] +
                8 * occs[pidx3[:, 0] + axis2[:, 0], pidx3[:, 1] + axis2[:, 1], pidx3[:, 2] + axis2[:, 2]])
        
        # Ambiguity correction
        amb_mask = amb_chks[pidx3[:, 0], pidx3[:, 1], pidx3[:, 2]]
        val_[amb_mask] = 15 - val_[amb_mask]
        
        # Compute vertices using 2D search
        vertices_a = _2d_search(pidx3, axdir, axis0, axis1, val_, cases, lctab_2d, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH, HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH, ERR)
        val_pnts.extend(vertices_a)
        
    # Process Group B: QEF Optimization
    if len(pidx1_b) > 0:
        csidx = (pidx1_b // GRID3) - 9
        cases = cas_tab[csidx] # Assuming cases are stored in cas_tab for these indices too
        
        pidx3 = pidx1_to_pidx3(pidx1_b)
        
        # Compute occupancy value (same logic as Group A, but for different indices)
        # Note: The exact axis directions for Group B might differ or be derived differently
        # For simplicity, assuming similar structure
        axis0 = ax0_tab[csidx]
        axis1 = ax1_tab[csidx]
        axis2 = axis0 + axis1
        
        val_ = (occs[pidx3[:, 0], pidx3[:, 1], pidx3[:, 2]] +
                2 * occs[pidx3[:, 0] + axis1[:, 0], pidx3[:, 1] + axis1[:, 1], pidx3[:, 2] + axis1[:, 2]] +
                4 * occs[pidx3[:, 0] + axis0[:, 0], pidx3[:, 1] + axis0[:, 1], pidx3[:, 2] + axis0[:, 2]] +
                8 * occs[pidx3[:, 0] + axis2[:, 0], pidx3[:, 1] + axis2[:, 1], pidx3[:, 2] + axis2[:, 2]])
        
        # Ambiguity correction
        amb_mask = amb_chks[pidx3[:, 0], pidx3[:, 1], pidx3[:, 2]]
        val_[amb_mask] = 15 - val_[amb_mask]
        
        # Compute vertices using QEF
        vertices_b = _solve_QEF(pidx3, val_, cases, qvntab, pfvtab, vitogi, cstrtab, QEF_REG, ERR, SEP, GRID3, GRID2, GRID)
        val_pnts.extend(vertices_b)
        
    return val_pnts

def _2d_search(pidx3, axdir, axis0, axis1, val_, cases, lctab_2d, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH, HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH, ERR):
    """
    Performs 2D linear/binomial search for vertex positions.
    """
    # Placeholder for the complex 2D search logic described in [S1, unofficial]
    # This involves finding spnt, vnorm, and refining with horizontal searches
    # The exact implementation depends on the specific lookup tables and search parameters
    pass

def _solve_QEF(pidx3, val_, cases, qvntab, pfvtab, vitogi, cstrtab, QEF_REG, ERR, SEP, GRID3, GRID2, GRID):
    """
    Solves the QEF for vertex positions.
    """
    vertices = []
    for i in range(3, 8): # Vertex indices 3 to 7
        idx = qvntab[val_, cases] == i
        if not idx.any():
            continue
            
        vtcs = pfvtab[val_, cases, :2*i+1][idx]
        vitg = vitogi[vtcs][idx]
        
        # Compute 3D positions
        pid3 = pidx3[idx]
        pnti = (vitg[:,:,3] * GRID3 + 
                (pid3[:,0] + vitg[:,:,0]) * GRID2 + 
                (pid3[:,1] + vitg[:,:,1]) * GRID + 
                (pid3[:,2] + vitg[:,:,2]))
        
        # Get global indices and pre-computed positions
        # Note: This assumes val_pnts is accessible or passed in
        # For simplicity, we assume we can retrieve them
        # In practice, this might require a global lookup or passing val_pnts as an argument
        vert = get_precomputed_vertices(pnti) # Placeholder
        
        # Add local offsets
        vert = vert + vitg[:,:3]
        
        # Compute QEF normals
        _1vt = vert[:, 1::2]
        qnrm = cross(_1vt - vert[:, :-1:2], _1vt - vert[:, 2::2])
        cntr = mean(_1vt, dim=1)
        
        # Normalize normals
        leng = norm(qnrm, dim=1, keepdim=True)
        cndt = leng < ERR
        leng[cndt] = 1.0
        qnrm = qnrm / leng
        
        # Solve QEF
        A = cat([qnrm, QEF_REG * eye(qnrm.shape[1])], dim=0)
        b = cat([sum(_1vt * qnrm, dim=1, keepdim=True), QEF_REG * cntr], dim=0)
        qefr = lstsq(A, b).solution
        
        # Handle degenerate cases
        if all(cndt):
            qefr = cntr
            
        # Ray intersection
        cpnt = get_cell_center(pidx3[idx]) # Placeholder
        ctoq = qefr - cpnt
        leng = norm(ctoq, dim=1, keepdim=True)
        
        # Compute t
        # This part is complex and depends on cstrtab
        # Placeholder for the actual intersection logic
        t = compute_intersection_t(ctoq, cpnt, cstrtab, leng, SEP, ERR)
        
        # Final vertex
        final_vert = cpnt + t * ctoq
        vertices.append(final_vert)
        
    return vertices
```

### Parameters
*   `ERR`: Epsilon for numerical stability, typically a small value like $10^{-6}$ [S1, unofficial].
*   `QEF_REG`: Regularization factor for QEF, typically a small value like $10^{-4}$ [S1, unofficial].
*   `SEP`: Separation constant for ray intersection, typically a small value like $10^{-3}$ [S1, unofficial].
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Ranges for the 2D search [S1, unofficial].
*   `VERTICAL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Flags to enable/disable linear or binary search steps [S1, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Normals**: If all QEF normals are degenerate (zero length), the centroid is used instead [S1, unofficial].
*   **NaNs and Negative Values**: In ray intersection, NaNs and negative values for $t$ are handled by setting $t$ to $\text{leng} + \text{SEP}$ [S1, unofficial].
*   **Ambiguity Correction**: The ambiguity correction step is crucial for watertightness in ambiguous cases [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The method involves iterating over all cells and performing either a fast 2D search or a QEF optimization. The QEF optimization involves solving a least squares problem, which is more expensive than linear interpolation but still efficient for moderate grid sizes. The 2D search is faster but requires additional lookups and computations. Overall, the time complexity is $O(N)$ where $N$ is the number of cells, but with a higher constant factor for QEF cells [S1, unofficial].
*   **Memory Complexity**: The method requires storing occupancy arrays, ambiguity check arrays, and lookup tables. The size of these arrays is proportional to the grid size $G^3$. The QEF optimization also requires temporary storage for normals, centroids, and least squares solutions. Overall, the memory complexity is $O(G^3)$ [S1, unofficial].
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: The method scales linearly with the number of cells. However, the QEF optimization may become a bottleneck for very large grids due to the computational cost of solving the least squares problem [S1, unofficial].
*   **Strengths**: Produces watertight meshes with reduced artifacts in complex topologies compared to standard Marching Cubes [S1, unofficial]. The 2D search provides a fast alternative for non-ambiguous cases [S1, unofficial].
*   **Limitations**: More complex to implement than standard Marching Cubes. The QEF optimization may be slower than linear interpolation [S1, unofficial].
*   **When to use**: When high-quality watertight meshes are required, especially for complex topologies where standard Marching Cubes may produce artifacts [S1, unofficial].
*   **When not to use**: When performance is critical and the grid size is very large, or when the topological complexity is low and standard Marching Cubes is sufficient [S1, unofficial].

## Background
The method originates from the ODC (Octree-based Dual Contouring) project by the KAIST Visual AI Group [S1, unofficial]. It combines the robustness of QEF minimization with the efficiency of 2D search to handle ambiguous cases in Marching Cubes. The QEM-based vertex relocation variant is described in a paper on generating smooth high-quality isosurfaces for interactive modeling and visualization of complex terrains [S2, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
