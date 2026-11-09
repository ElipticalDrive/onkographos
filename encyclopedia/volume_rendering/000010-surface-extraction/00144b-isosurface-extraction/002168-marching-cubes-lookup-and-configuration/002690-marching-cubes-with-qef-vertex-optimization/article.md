# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions using Quadric Error Function (QEF) minimization for ambiguous topological cases, while employing a 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It distinguishes itself from standard Marching Cubes by resolving topological ambiguities through geometric optimization rather than lookup tables, and from Dual Contouring by retaining the structured grid traversal of Marching Cubes while optimizing vertex placement [S1, unofficial].

## How it works
The method operates on a regular 3D grid where each voxel has an occupancy state (binary or scalar thresholded). The algorithm traverses the grid, identifies surface intersections, and computes vertex positions.

**Inputs:**
- `occs`: 3D array of occupancy bits (0 or 1) for grid vertices.
- `GRID`, `GRID2`, `GRID3`: Grid spacing constants ($1, S, S^2$).
- `ERR`: Numerical threshold for degenerate checks.
- `QEF_REG`: Regularization weight for QEF least squares.
- `SEP`: Separation constant for boundary constraints.
- `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Search ranges for 2D search.
- `VERTICAL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Flags controlling search modes.
- Lookup tables: `lctab_2d`, `axd_tab`, `ax0_tab`, `ax1_tab`, `cas_tab`, `qvntab`, `pfvtab`, `vitogi`, `cstrtab`.

**Outputs:**
- `val_pnts`: Array of computed vertex positions.
- Mesh connectivity (implied by the traversal and vertex storage).

**Data Structures:**
- `pidx1`: 1D array of voxel indices to process.
- `pidx3`: 3D coordinates corresponding to `pidx1`.
- `val_`: Integer occupancy pattern (0-15) for a cell.
- `cases`: Ambiguity case identifier.

**Steps:**

1.  **Filter and Initialize:**
    Filter voxel indices `pidx1` into two groups:
    - Group 1 (Fast): Indices in range $[3 \cdot GRID3, 9 \cdot GRID3)$.
    - Group 2 (QEF): Indices $\ge 9 \cdot GRID3$.
    Initialize `start_idx` to 0.

2.  **Process Group 1 (Non-Ambiguous/Fast Path):**
    For each index in Group 1:
    a.  Compute the 8-neighbor occupancy value `val_` by summing weighted occupancy bits:
        $$val_ = occs[pidx3] + 2 \cdot occs[pidx3 + axis1] + 4 \cdot occs[pidx3 + axis0] + 8 \cdot occs[pidx3 + axis2]$$
        where `axis0`, `axis1`, `axis2` are offset vectors for the specific cell orientation.
    b.  Apply ambiguity correction: If `tmp = amb_chks[pidx3]` is true, set `val_ = 15 - val_`.
    c.  Compute vertex positions using `_2d_search(pidx3, axdir, axis0, axis1, val_, cases)` and store in `val_pnts` at `start_idx`.
    d.  Increment `start_idx` by the count of processed vertices.

3.  **Process Group 2 (Ambiguous/QEF Path):**
    For each index in Group 2:
    a.  Call `_solve_QEF(pidx3, val_, cases)` where `val_` is the integer occupancy pattern and `cases` is derived from `pidx1 / GRID3 - 9`.
    b.  Store results in `val_pnts` at `start_idx`.

4.  **QEF Solving (`_solve_QEF`):**
    a.  For each vertex index `i` from 3 to 7, find indices `idx` where `qvntab[val_, cases] == i`.
    b.  Retrieve vertex coordinates `vtcs` from `pfvtab[val0_, cas, :2*i+1]` and edge indices `vitg` from `vitogi[vtcs]`.
    c.  Compute 3D positions `pnti` for the vertices using grid offsets:
        $$pnti = vitg[:,:,3] \cdot GRID3 + (pid3[:,0] + vitg[:,:,0]) \cdot GRID2 + (pid3[:,1] + vitg[:,:,1]) \cdot GRID + (pid3[:,2] + vitg[:,:,2])$$
    d.  Get global indices `pidx` and retrieve pre-computed vertex positions `vert = val_pnts[pidx]`.
    e.  Add local offsets `vitg[:,:3]` to `vert`.
    f.  Compute QEF normals `qnrm` using cross products of edge vectors:
        $$\_1vt = vert[:, 1::2]$$
        $$qnrm = cross(\_1vt - vert[:, :-1:2], \_1vt - vert[:, 2::2])$$
    g.  Compute centroid `cntr = mean(\_1vt, dim=1)`.
    h.  Normalize `qnrm` handling zero-length edges with `ERR` threshold.
    i.  Solve the QEF system using least squares:
        $$A = cat(qnrm, QEF\_REG \cdot eye)$$
        $$b = cat(sum(\_1vt \cdot qnrm), QEF\_REG \cdot cntr)$$
        Solution `qefr = lstsq(A, b).solution`.
    j.  If all normals are degenerate (`cndt` all true), use `cntr` instead.
    k.  Compute ray direction `ctoq = qefr - cpnt` and length `leng`.
    l.  Compute intersection parameter `t` with constraint planes defined by `cstrtab`. Concatenate boundary parameters $(1-cpnt)/ctoq$ and $-cpnt/ctoq$ with `t`.
    m.  Handle NaNs and negative values by setting `t` to $leng + SEP$, then normalize by `leng` and clamp to $[0, 1]$.
    n.  Final vertex is $cpnt + t \cdot ctoq$.

5.  **2D Search (`_2d_search`):**
    a.  Compute 8-neighbor occupancy value `val_` as in Step 2a.
    b.  Apply ambiguity correction as in Step 2b.
    c.  Perform 2D linear/binomial search to determine vertex position relative to the cell edges.
    d.  Refine position using normal vectors derived from edge differences and occupancy checks.
    e.  Return final vertex position.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def march_cubes_qef(occs, GRID, GRID2, GRID3, ERR, QEF_REG, SEP, 
                    VERTICAL_RANGE, HORIZNTL_RANGE, 
                    VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH, 
                    HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH,
                    lctab_2d, axd_tab, ax0_tab, ax1_tab, cas_tab, 
                    qvntab, pfvtab, vitogi, cstrtab, amb_chks):
    """
    Marching Cubes with QEF Vertex Optimization.
    """
    val_pnts = [] # List to store vertex positions
    start_idx = 0
    
    # Filter indices
    pidx1_all = get_all_voxel_indices() # Implementation detail: iterate grid
    pidx1_fast = [p for p in pidx1_all if 3 * GRID3 <= p < 9 * GRID3]
    pidx1_qef = [p for p in pidx1_all if p >= 9 * GRID3]
    
    # Process Fast Group
    for pidx1 in pidx1_fast:
        pidx3 = pidx1_to_pidx3(pidx1)
        csidx = pidx1 // GRID3 - 3
        axdir = axd_tab[csidx]
        axis0 = ax0_tab[csidx]
        axis1 = ax1_tab[csidx]
        cases = cas_tab[csidx]
        axis2 = axis0 + axis1
        
        # Compute occupancy pattern
        val_ = (occs[pidx3[0], pidx3[1], pidx3[2]] +
                2 * occs[pidx3[0] + axis1[0], pidx3[1] + axis1[1], pidx3[2] + axis1[2]] +
                4 * occs[pidx3[0] + axis0[0], pidx3[1] + axis0[1], pidx3[2] + axis0[2]] +
                8 * occs[pidx3[0] + axis2[0], pidx3[1] + axis2[1], pidx3[2] + axis2[2]])
        
        # Ambiguity correction
        if amb_chks[pidx1]:
            val_ = 15 - val_
            
        # 2D Search
        vertex = _2d_search(pidx3, axdir, axis0, axis1, val_, cases,
                            lctab_2d, VERTICAL_RANGE, HORIZNTL_RANGE,
                            VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                            HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH,
                            ERR)
        val_pnts.append(vertex)
        start_idx += 1
        
    # Process QEF Group
    for pidx1 in pidx1_qef:
        pidx3 = pidx1_to_pidx3(pidx1)
        csidx = pidx1 // GRID3 - 9
        cases = cas_tab[csidx] # Note: cases derived differently for QEF group
        
        # Compute occupancy pattern (simplified, assumes standard orientation)
        # In practice, axis offsets might be needed or precomputed
        val_ = compute_occupancy_pattern(pidx3, occs) 
        
        vertex = _solve_QEF(pidx3, val_, cases, qvntab, pfvtab, vitogi, 
                            cstrtab, QEF_REG, ERR, SEP, val_pnts)
        val_pnts.append(vertex)
        start_idx += 1
        
    return val_pnts

def _2d_search(pidx3, axdir, axis0, axis1, val_, cases, lctab_2d,
               VERTICAL_RANGE, HORIZNTL_RANGE, VERTICAL_LINEAR_SEARCH, 
               VERTICAL_BINARY_SEARCH, HORIZNTL_LINEAR_SEARCH, 
               HORIZNTL_BINARY_SEARCH, ERR):
    """
    Perform 2D linear/binomial search for non-ambiguous cases.
    """
    # Lookup local cell resolution
    lcres = lctab_2d[val_, cases + 1]
    
    # Determine edge directions and indices
    arng = range(len(lcres))
    edir = axdir[arng, lcres[:, 2]]
    
    # Compute base positions
    cidx3 = pidx3.clone()
    cidx3[arng, axdir[:, 0]] += lcres[:, 0]
    cidx3[arng, axdir[:, 1]] += lcres[:, 1]
    
    # Compute linear/binary search results
    # This part involves complex logic from S1, simplified here for structure
    # Actual implementation requires full S1 logic for _2d_linbin
    
    # Compute midpoint and normals
    spnt = (lcord + rcord) * 0.5
    locc = get_occv(spnt)
    
    hnorm = lcord - rcord
    leng = norm(hnorm)
    cndt = leng < ERR
    if cndt: leng = 1.0
    hnorm = hnorm / leng
    
    vnorm = cross(cross(axis0, axis1), hnorm)
    
    # Adjust normals based on occupancy
    # ... (see S1 for detailed occupancy checks)
    
    # Perform search
    left = _2d_linbin(spnt, locc, vnorm, VERTICAL_RANGE, 
                      VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH)
    
    # ... (rest of S1 logic)
    
    return spnt + rslt

def _solve_QEF(pidx3, val_, cases, qvntab, pfvtab, vitogi, cstrtab, 
               QEF_REG, ERR, SEP, val_pnts):
    """
    Solve QEF for ambiguous cases.
    """
    # Retrieve vertex indices and edge info
    # ... (see S1 for qvntab lookup)
    
    # Compute positions and normals
    # ... (see S1 for pnti, vert, qnrm, cntr)
    
    # Solve Least Squares
    A = concatenate([qnrm, QEF_REG * eye], axis=1)
    b = concatenate([sum(_1vt * qnrm), QEF_REG * cntr], axis=1)
    qefr = lstsq(A, b)
    
    # Handle degeneracy
    if all(cndt):
        qefr = cntr
        
    # Compute final vertex with constraints
    ctoq = qefr - cpnt
    leng = norm(ctoq)
    
    # Compute t with constraints
    # ... (see S1 for cstrtab and boundary handling)
    
    t = clamp(normalize(t), 0, 1)
    return cpnt + t * ctoq
```

**Edge Cases and Pitfalls:**
- **Degenerate Normals:** If all edge vectors are zero-length (degenerate cell), the QEF system is singular. The method falls back to the centroid `cntr` [S1, unofficial].
- **Numerical Precision:** Use `ERR` threshold to handle near-zero lengths and dot products to avoid division by zero or NaNs [S1, unofficial].
- **Boundary Conditions:** Constraint planes `cstrtab` must be correctly defined to handle mesh boundaries and ensure watertightness [S1, unofficial].
- **Ambiguity Correction:** The ambiguity correction `val_ = 15 - val_` is critical for correct topology in ambiguous cases [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of ambiguous cases. Non-ambiguous cases use fast 2D search ($O(1)$ per vertex). Ambiguous cases use QEF least squares ($O(V)$ where $V$ is the number of vertices in the local patch, typically small constant). Overall complexity is linear with respect to the number of surface vertices.
- **Memory Complexity:** Requires storage for occupancy grid, lookup tables, and vertex positions. QEF method requires additional memory for normal and centroid computations per ambiguous vertex.
- **Performance:** QEF method is more computationally expensive than standard Marching Cubes due to least squares solving, but provides higher quality meshes for ambiguous cases [S1, unofficial]. QEM-based variant offers a trade-off between quality and performance on parallel hardware [S2, unofficial].
- **Strengths:** Handles topological ambiguities correctly, produces watertight meshes, higher quality than standard MC for ambiguous cases.
- **Limitations:** More complex implementation, higher computational cost for ambiguous cases, sensitive to numerical precision.
- **When to Use:** When high-quality surface reconstruction is required, especially in regions with topological ambiguities.
- **When Not to Use:** When computational resources are severely limited and approximate geometry is acceptable.

## Background
The method originates from the ODC (Octree-based Direct Contouring) project by KAIST Visual AI Group [S1, unofficial]. It combines the structured grid traversal of Marching Cubes with the geometric optimization of QEF, a technique popularized by Garland and Heckbert for mesh simplification [S2, unofficial]. The QEM-based variant was proposed to address performance issues on parallel hardware [S2, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
