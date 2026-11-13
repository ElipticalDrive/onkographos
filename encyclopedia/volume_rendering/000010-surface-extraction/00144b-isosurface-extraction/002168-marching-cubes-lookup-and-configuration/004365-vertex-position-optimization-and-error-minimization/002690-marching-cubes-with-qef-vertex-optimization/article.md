# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions by minimizing a Quadric Error Function (QEF) for ambiguous cube configurations, while employing a fast 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) for topology determination and applies the shared principles of [Vertex Position Optimization and Error Minimization](10991-vertex-position-optimization-and-error-minimization.md) to reduce geometric error $E$ between extracted vertices and the scalar field $f(\mathbf{x})$ [S1, unofficial].

## How it works
The method processes voxel indices by categorizing them into two groups based on their configuration complexity: a fast group for non-ambiguous cases and a slow group for ambiguous cases requiring QEF minimization [S1, unofficial].

### Inputs and Outputs
*   **Input**: A voxel grid of occupancy bits `occs`, grid dimensions `GRID`, `GRID2`, `GRID3` (where $GRID2 = GRID \times GRID$, $GRID3 = GRID \times GRID \times GRID$), and pre-computed lookup tables (`lctab_2d`, `axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`, `qvntab`, `pfvtab`, `vitogi`, `cstrtab`, `amb_chks`) [S1, unofficial].
*   **Output**: A list of optimized vertex positions `val_pnts` and associated triangle indices [S1, unofficial].

### Data Structures
*   `occs`: 3D array of integer occupancy bits (0 or 1) [S1, unofficial].
*   `pidx1`: 1D array of linear voxel indices to process [S1, unofficial].
*   `val_pnts`: 2D array of vertex positions, indexed by global vertex ID [S1, unofficial].
*   `GRID`, `GRID2`, `GRID3`: Integer constants representing grid strides [S1, unofficial].
*   `ERR`: Floating-point threshold for degenerate edge detection [S1, unofficial].
*   `QEF_REG`: Regularization parameter for the QEF least-squares solver [S1, unofficial].
*   `SEP`: Separation constant for boundary handling [S1, unofficial].

### Algorithm Steps
1.  **Filter and Initialize**:
    Split `pidx1` into two groups:
    *   Group 1 (Fast): Indices in range $[3 \cdot GRID3, 9 \cdot GRID3)$ [S1, unofficial].
    *   Group 2 (QEF): Indices $\ge 9 \cdot GRID3$ [S1, unofficial].
    Initialize `start_idx` to 0 [S1, unofficial].

2.  **Process Group 1 (Non-Ambiguous)**:
    For each index `pidx1` in Group 1:
    a.  Compute the 8-neighbor occupancy value `val_`:
        $$ val\_ = occs[pidx3] + 2 \cdot occs[pidx3 + axis1] + 4 \cdot occs[pidx3 + axis0] + 8 \cdot occs[pidx3 + axis2] $$
        where `pidx3` is the 3D coordinate of the voxel, and `axis0`, `axis1` are direction vectors from lookup tables [S1, unofficial].
    b.  Apply ambiguity correction: If `amb_chks[pidx3]` is true, set `val\_ = 15 - val\_` [S1, unofficial].
    c.  Compute vertex positions using `_2d_search(pidx3, axdir, axis0, axis1, val_, cases)` [S1, unofficial].
    d.  Store results in `val_pnts` at `start_idx` and increment `start_idx` by the count of processed vertices [S1, unofficial].

3.  **Process Group 2 (Ambiguous/QEF)**:
    For each index `pidx1` in Group 2:
    a.  Derive the case index: $cases = pidx1 / GRID3 - 9$ [S1, unofficial].
    b.  Compute `val_` (occupancy pattern) similarly to Step 2 [S1, unofficial].
    c.  Call `_solve_QEF(pidx3, val_, cases)` to compute optimized vertex positions [S1, unofficial].
    d.  Store results in `val_pnts` at `start_idx` [S1, unofficial].

4.  **`_solve_QEF` Implementation**:
    For each vertex index $i$ from 3 to 7:
    a.  Find indices `idx` where `qvntab[val_, cases] == i` [S1, unofficial].
    b.  Retrieve vertex coordinates `vtcs` from `pfvtab[val0_, cas, :2*i+1]` and edge indices `vitg` from `vitogi[vtcs]` [S1, unofficial].
    c.  Compute 3D positions `pnti` using grid offsets:
        $$ pnti = vitg[:,:,3] \cdot GRID3 + (pid3[:,0] + vitg[:,:,0]) \cdot GRID2 + (pid3[:,1] + vitg[:,:,1]) \cdot GRID + (pid3[:,2] + vitg[:,:,2]) $$
    d.  Get global indices `pidx` and retrieve pre-computed vertex positions `vert = val_pnts[pidx]` [S1, unofficial].
    e.  Add local offsets `vitg[:,:3]` to `vert` [S1, unofficial].

5.  **QEF Normal and Centroid Calculation**:
    a.  Compute QEF normals `qnrm` using cross products of edge vectors:
        $$ \_1vt = vert[:, 1::2] $$
        $$ qnrm = \text{cross}(\_1vt - vert[:, :-1:2], \_1vt - vert[:, 2::2]) $$
    b.  Compute centroid `cntr = mean(\_1vt, dim=1)` [S1, unofficial].
    c.  Normalize `qnrm`. Handle zero-length edges by checking if length $< ERR$; if so, set length to 1.0 before normalization [S1, unofficial].

6.  **Solve QEF System**:
    a.  Construct matrix $A$ and vector $b$:
        $$ A = \text{cat}(qnrm, QEF\_REG \cdot I) $$
        $$ b = \text{cat}(\sum(\_1vt \cdot qnrm), QEF\_REG \cdot cntr) $$
    b.  Solve least squares: $qefr = \text{lstsq}(A, b).\text{solution}$ [S1, unofficial].
    c.  If all normals are degenerate (`cndt` all true), use `cntr` instead of `qefr` [S1, unofficial].

7.  **Boundary Constraint and Final Position**:
    a.  Compute ray direction $ctoq = qefr - cpnt$ and length $leng$ [S1, unofficial].
    b.  Compute intersection parameter $t$ with constraint planes defined by `cstrtab` [S1, unofficial].
    c.  Concatenate boundary parameters $(1-cpnt)/ctoq$ and $-cpnt/ctoq$ with $t$ [S1, unofficial].
    d.  Handle NaNs and negative values: if invalid, set $t = leng + SEP$ [S1, unofficial].
    e.  Normalize by $leng$ and clamp $t$ to $[0, 1]$ [S1, unofficial].
    f.  Final vertex: $v = cpnt + t \cdot ctoq$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_qef_vertex_optimization(occs, pidx1, GRID, GRID2, GRID3, 
                                  lctab_2d, axd_tab, ax0_tab, ax1_tab, cas_tab,
                                  qvntab, pfvtab, vitogi, cstrtab, 
                                  amb_chks, ERR, QEF_REG, SEP):
    """
    Computes optimized vertex positions for Marching Cubes using QEF for ambiguous cases.
    """
    val_pnts = {} # Dictionary mapping global vertex index to position vector
    start_idx = 0
    
    # Split indices
    fast_mask = (pidx1 >= 3 * GRID3) & (pidx1 < 9 * GRID3)
    slow_mask = pidx1 >= 9 * GRID3
    
    pidx1_fast = pidx1[fast_mask]
    pidx1_slow = pidx1[slow_mask]
    
    # Process Fast Group
    if len(pidx1_fast) > 0:
        pidx3_fast = pidx1_fast // (GRID2 * GRID) # linear_to_3d helper
        csidx_fast = (pidx1_fast // GRID3) - 3
        axdir_fast = axd_tab[csidx_fast]
        axis0_fast = ax0_tab[csidx_fast]
        axis1_fast = ax1_tab[csidx_fast]
        cases_fast = cas_tab[csidx_fast]
        axis2_fast = axis0_fast + axis1_fast
        
        # Compute occupancy values
        occs_fast = occs[pidx3_fast[:, 0], pidx3_fast[:, 1], pidx3_fast[:, 2]]
        occs_a1 = occs[pidx3_fast[:, 0] + axis1_fast[:, 0], pidx3_fast[:, 1] + axis1_fast[:, 1], pidx3_fast[:, 2] + axis1_fast[:, 2]]
        occs_a0 = occs[pidx3_fast[:, 0] + axis0_fast[:, 0], pidx3_fast[:, 1] + axis0_fast[:, 1], pidx3_fast[:, 2] + axis0_fast[:, 2]]
        occs_a2 = occs[pidx3_fast[:, 0] + axis2_fast[:, 0], pidx3_fast[:, 1] + axis2_fast[:, 1], pidx3_fast[:, 2] + axis2_fast[:, 2]]
        
        val_fast = occs_fast + 2 * occs_a1 + 4 * occs_a0 + 8 * occs_a2
        
        # Ambiguity correction
        amb_mask = amb_chks[pidx3_fast[:, 0], pidx3_fast[:, 1], pidx3_fast[:, 2]]
        val_fast[amb_mask] = 15 - val_fast[amb_mask]
        
        # Compute positions
        # _2d_search is not fully detailed in sources; implements 2D linear/binomial search [G]
        positions_fast = _2d_search(pidx3_fast, axdir_fast, axis0_fast, axis1_fast, val_fast, cases_fast)
        
        # Store in val_pnts
        for i, pidx in enumerate(pidx1_fast):
            for v in positions_fast[i]:
                val_pnts[v.global_idx] = v.position
            start_idx += len(positions_fast[i])

    # Process Slow Group (QEF)
    if len(pidx1_slow) > 0:
        pidx3_slow = pidx1_slow // (GRID2 * GRID)
        cases_slow = (pidx1_slow // GRID3) - 9
        
        for i, pidx in enumerate(pidx1_slow):
            pidx3 = pidx3_slow[i]
            # Compute val_ similarly to fast group
            val_ = ... # Compute val_ for this voxel [G]
            case_idx = cases_slow[i]
            
            optimized_verts = _solve_QEF(pidx3, val_, case_idx, occs, GRID, GRID2, GRID3, 
                                         qvntab, pfvtab, vitogi, cstrtab, val_pnts, ERR, QEF_REG, SEP)
            
            for v in optimized_verts:
                val_pnts[v.global_idx] = v.position
            start_idx += len(optimized_verts)

    return val_pnts

def _solve_QEF(pidx3, val_, case_idx, occs, GRID, GRID2, GRID3, 
               qvntab, pfvtab, vitogi, cstrtab, val_pnts, ERR, QEF_REG, SEP):
    """
    Solves QEF for a specific ambiguous cube configuration.
    """
    vertices = []
    
    # Iterate over vertex indices 3 to 7 (ambiguous vertices)
    for i in range(3, 8):
        # Find which edges correspond to vertex i
        edge_indices = [k for k, v in enumerate(qvntab[val_, case_idx]) if v == i]
        
        if len(edge_indices) == 0:
            continue
            
        # Retrieve pre-computed vertex coordinates and edge indices
        vtcs = pfvtab[val_, case_idx, :2*i+1] 
        vitg = vitogi[vtcs] 
        
        # Compute 3D positions
        pnti = vitg[:, 3] * GRID3 + (pidx3[0] + vitg[:, 0]) * GRID2 + \
               (pidx3[1] + vitg[:, 1]) * GRID + (pidx3[2] + vitg[:, 2])
        
        # Get global indices and pre-computed positions
        pidx_global = ... # Map pnti to global vertex indices [G]
        vert = [val_pnts[idx] for idx in pidx_global]
        
        # Add local offsets
        vert = [vert[k] + vitg[k, :3] for k in range(len(vert))]
        
        # Compute QEF normals
        _1vt = [vert[k][1::2] for k in range(len(vert))] 
        edge_vecs_1 = [_1vt[k] - vert[k][:-1:2] for k in range(len(vert))]
        edge_vecs_2 = [_1vt[k] - vert[k][2::2] for k in range(len(vert))]
        qnrm = [cross(e1, e2) for e1, e2 in zip(edge_vecs_1, edge_vecs_2)]
        
        # Normalize normals
        lengths = [norm(n) for n in qnrm]
        degenerate = [l < ERR for l in lengths]
        qnrm = [n / (1.0 if d else l) for n, l, d in zip(qnrm, lengths, degenerate)]
        
        # Compute centroid
        cntr = mean(_1vt, axis=0)
        
        # Solve QEF
        A = cat([qnrm, QEF_REG * eye(3)])
        b = cat([sum(_1vt * qnrm, axis=0), QEF_REG * cntr])
        
        qefr = lstsq(A, b).solution
        
        # Handle degenerate case
        if all(degenerate):
            qefr = cntr
            
        # Boundary constraints
        cpnt = ... # Center point or reference point [G]
        ctoq = qefr - cpnt
        leng = norm(ctoq)
        
        # Compute t with constraints
        # compute_intersection_t handles cstrtab, NaNs, negatives, clamping [G]
        t = compute_intersection_t(ctoq, cpnt, cstrtab, leng, SEP, ERR)
        
        # Final vertex
        final_v = cpnt + t * ctoq
        
        vertices.append(final_v)
        
    return vertices
```

### Parameters
*   `ERR`: Threshold for degenerate edges. Typical value: Not given in the sources.
*   `QEF_REG`: Regularization weight for QEF. Typical value: Not given in the sources.
*   `SEP`: Separation constant for boundary handling. Typical value: Not given in the sources.
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Search ranges for 2D search. Typical value: Not given in the sources.
*   `VERTICAL_LINEAR_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Boolean flags for search type. Typical value: Not given in the sources.

### Edge Cases and Pitfalls
*   **Degenerate Normals**: If all edge vectors are zero-length (length $< ERR$), the QEF system is singular. The method falls back to using the centroid `cntr` [S1, unofficial].
*   **NaNs and Negative Values**: Intersection parameters $t$ that are NaN or negative are handled by setting $t = leng + SEP$ before clamping to $[0, 1]$ [S1, unofficial].
*   **Ambiguity Correction**: For certain configurations, the occupancy value `val_` must be inverted ($15 - val\_$) to resolve topological ambiguities [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The method splits processing into two groups. The fast group uses 2D linear/binomial search, which is $O(1)$ per vertex. The slow group uses QEF least-squares solution, which is $O(1)$ per vertex but with a higher constant factor due to matrix operations. Overall complexity is linear with respect to the number of voxels $N$, i.e., $O(N)$.
*   **Memory Complexity**: Requires storage for `val_pnts` (one position per vertex), lookup tables, and intermediate arrays for QEF solving. Memory usage is $O(N)$.
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Produces watertight meshes with high geometric accuracy for ambiguous cases. The hybrid approach (fast search for non-ambiguous, QEF for ambiguous) balances speed and quality.
*   **Limitations**: QEF solving is computationally expensive compared to linear interpolation. Register usage can be high on parallel hardware if not optimized (as noted in QEM variant [S2, unofficial]).
*   **When to Use**: When high geometric fidelity is required, especially in regions with topological ambiguities.
*   **When Not to Use**: When performance is critical and approximate geometry is acceptable; consider standard Marching Cubes or QEM-based variants.

## Background
The method is designed for efficient surface extraction from voxel grids, particularly in contexts where topological ambiguity is common [S1, unofficial]. The QEM-based variant is discussed in the context of preserving the volume of terrain data [S2, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
