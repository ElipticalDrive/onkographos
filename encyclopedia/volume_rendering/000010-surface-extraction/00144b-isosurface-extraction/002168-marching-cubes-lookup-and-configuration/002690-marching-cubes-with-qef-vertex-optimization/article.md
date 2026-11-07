# Marching Cubes with QEF Vertex Optimization

Also called: MC-QEF, Optimized Marching Cubes

## Overview
Marching Cubes with QEF Vertex Optimization is a surface extraction method that computes vertex positions using a Quadric Error Function (QEF) minimization for ambiguous cases, while using a 2D linear/binomial search for non-ambiguous cases, to produce a watertight mesh from a voxel grid [S1, unofficial]. It builds on [Isosurface extraction](30-isosurface-extraction.md) by replacing the standard linear interpolation of classic Marching Cubes with a geometric optimization step that minimizes distance to supporting planes, thereby reducing artifacts in ambiguous topological configurations.

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid of occupancy values `occs` (binary or scalar), an isovalue threshold (implicit in occupancy), and precomputed lookup tables for topology (`lctab_2d`, `axd_tab`, `cas_tab`, etc.).
*   **Output:** A list of vertex positions `val_pnts` and associated topology indices.

**Data Structures:**
*   `occs`: 3D array of occupancy bits.
*   `GRID`, `GRID2`, `GRID3`: Grid dimensions or stride constants.
*   `ERR`: Small threshold for degenerate normal detection.
*   `QEF_REG`: Regularization constant for QEF least-squares solver.
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Search ranges for 2D optimization.
*   `VERTICAL_LINEAR_SEARCH`, `VERTICAL_BINARY_SEARCH`, `HORIZNTL_LINEAR_SEARCH`, `HORIZNTL_BINARY_SEARCH`: Flags controlling search strategy.
*   `qvntab`, `pfvtab`, `vitogi`, `cstrtab`: Precomputed tables for vertex quantization, face vertex indices, edge indices, and constraint planes.

**Steps:**

1.  **Filter and Classify Cells:**
    Filter voxel indices `pidx1` into two groups based on their value relative to `GRID3`:
    *   Group 1: Indices in range `[3*GRID3, 9*GRID3)`. These are processed using a fast 2D search.
    *   Group 2: Indices `>= 9*GRID3`. These are processed using QEF optimization.
    Initialize `start_idx` to 0.

2.  **Process Group 1 (Fast 2D Search):**
    For each index in Group 1:
    a.  Compute the 8-neighbor occupancy value `val_` by summing weighted occupancy bits:
        $$val_ = occs[pidx3] + 2 \cdot occs[pidx3 + axis1] + 4 \cdot occs[pidx3 + axis0] + 8 \cdot occs[pidx3 + axis2]$$
        where `pidx3` is the 3D coordinate of the cell, and `axis0`, `axis1` are direction vectors from lookup tables.
    b.  Apply ambiguity correction: If `tmp = amb_chks[pidx3]` is true, set `val_ = 15 - val_`.
    c.  Compute vertex positions using `_2d_search(pidx3, axdir, axis0, axis1, val_, cases)` and store in `val_pnts` at `start_idx`.
    d.  Increment `start_idx` by the count of processed indices.

3.  **Process Group 2 (QEF Optimization):**
    For each index in Group 2:
    a.  Call `_solve_QEF(pidx3, val_, cases)` where `val_` is the integer occupancy pattern and `cases` is derived from `pidx1 / GRID3 - 9`.
    b.  Store results in `val_pnts` at `start_idx`.

4.  **QEF Solver Details (`_solve_QEF`):**
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
        $$A = cat(qnrm, QEF_REG \cdot eye)$$
        $$b = cat(sum(\_1vt \cdot qnrm), QEF_REG \cdot cntr)$$
        Solution `qefr = lstsq(A, b).solution`.
    j.  If all normals are degenerate (`cndt` all true), use `cntr` instead.
    k.  Compute ray direction `ctoq = qefr - cpnt` and length `leng`.
    l.  Compute intersection parameter `t` with constraint planes defined by `cstrtab`. Concatenate boundary parameters `(1-cpnt)/ctoq` and `-cpnt/ctoq` with `t`.
    m.  Handle NaNs and negative values by setting `t` to `leng + SEP`, then normalize by `leng` and clamp to `[0, 1]`.
    n.  Final vertex is `cpnt + t \cdot ctoq`.

5.  **2D Search Details (`_2d_search`):**
    a.  Compute initial point `spnt` and normal `vnorm` based on occupancy differences.
    b.  Perform vertical search using `_2d_linbin` to find `left`.
    c.  Perform horizontal searches to find `lleft` and `rleft`.
    d.  Compute result `rslt` using the formula:
        $$rslt = \left( \frac{0.5 \cdot (rh + lh)}{denm} \cdot hnorm + \frac{vl}{denm} \cdot vnorm \right) \cdot leng$$
        where `denm = rh - lh`.
    e.  Clamp result if degenerate or invalid.
    f.  Return `spnt + rslt`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_qef_vertex_optimization(occs, GRID, GRID2, GRID3, GRID3_, ERR, QEF_REG, 
                                  VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                                  HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH,
                                  lctab_2d, axd_tab, cas_tab, amb_chks, qvntab, pfvtab, vitogi, cstrtab):
    # Filter indices
    pidx1 = get_all_surface_indices(occs) # Placeholder for actual index generation
    group1_mask = (3 * GRID3_ <= pidx1) & (pidx1 < 9 * GRID3_)
    group2_mask = pidx1 >= 9 * GRID3_
    
    pidx1_g1 = pidx1[group1_mask]
    pidx1_g2 = pidx1[group2_mask]
    
    val_pnts = []
    start_idx = 0
    
    # Process Group 1
    if len(pidx1_g1) > 0:
        count_idx = len(pidx1_g1)
        csidx = floor_div(pidx1_g1, GRID3_) - 3
        axdir = axd_tab[csidx]
        axis0 = ax0_tab[csidx]
        axis1 = ax1_tab[csidx]
        cases = cas_tab[csidx]
        axis2 = axis0 + axis1
        
        pidx3 = pidx1_to_pidx3(pidx1_g1)
        
        # Compute occupancy value
        val_ = (occs[pidx3[:, 0], pidx3[:, 1], pidx3[:, 2]] +
                2 * occs[pidx3[:, 0] + axis1[:, 0], pidx3[:, 1] + axis1[:, 1], pidx3[:, 2] + axis1[:, 2]] +
                4 * occs[pidx3[:, 0] + axis0[:, 0], pidx3[:, 1] + axis0[:, 1], pidx3[:, 2] + axis0[:, 2]] +
                8 * occs[pidx3[:, 0] + axis2[:, 0], pidx3[:, 1] + axis2[:, 1], pidx3[:, 2] + axis2[:, 2]])
        
        # Ambiguity correction
        amb = amb_chks[pidx3]
        val_ = where(amb, 15 - val_, val_)
        
        # 2D Search
        vertices_g1 = _2d_search(pidx3, axdir, axis0, axis1, val_, cases,
                                 lctab_2d, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
                                 HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH, ERR)
        
        val_pnts.extend(vertices_g1)
        start_idx += count_idx

    # Process Group 2
    if len(pidx1_g2) > 0:
        csidx = floor_div(pidx1_g2, GRID3_) - 9
        cases = cas_tab[csidx] # Assuming cases are derived differently for group 2
        # Note: val_ for group 2 needs to be computed similarly or retrieved
        # For simplicity, assuming val_ is passed or computed internally in _solve_QEF
        vertices_g2 = _solve_QEF(pidx1_g2, cases, occs, GRID, GRID2, GRID3, ERR, QEF_REG,
                                 qvntab, pfvtab, vitogi, cstrtab, val_pnts)
        val_pnts.extend(vertices_g2)

    return val_pnts

def _2d_search(pidx3, axdir, axis0, axis1, val_, cases, lctab_2d, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH,
               HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH, ERR):
    lcres = lctab_2d[val_, cases + 1]
    cidx3 = pidx3.clone()
    arng = range(len(pidx3)) # Placeholder for actual range
    
    # Update cidx3 based on lcres
    cidx3[arng, axdir[:, 0]] += lcres[:, 0]
    cidx3[arng, axdir[:, 1]] += lcres[:, 1]
    
    edir = axdir[arng, lcres[:, 2]]
    cidx1 = (2 - edir) * GRID3_ + cidx3[:, 0] * GRID2_ + cidx3[:, 1] * GRID_ + cidx3[:, 2]
    
    linv = val_pnts[get_idx(cidx1)] # Placeholder for actual index retrieval
    rcord = where(lcres[:, 2:] == 0, linv, axis0 * lcres[:, 0: 1]) + \
            where(lcres[:, 2:] == 1, linv, axis1 * lcres[:, 1: 2])
    
    # Compute normal and midpoint
    chk2 = pidx3.clone()
    occ2 = occs[chk2[:, 0], chk2[:, 1], chk2[:, 2]].unsqueeze(1)
    chk2 = (chk2 - pidx3).to(float64)
    
    chk3 = pidx3.clone()
    chk3[arng, edir] += 1
    occ3 = occs[chk3[:, 0], chk3[:, 1], chk3[:, 2]].unsqueeze(1)
    chk3 = (chk3 - pidx3).to(float64)
    
    spnt = (lcord + rcord) * 0.5
    locc = get_occv(pnts + spnt / GRID).unsqueeze(1) # Placeholder for actual occupancy retrieval
    
    hnorm = lcord - rcord
    leng = norm(hnorm, dim=1, keepdim=True)
    cndt = leng < ERR
    leng[cndt] = 1.0
    hnorm = hnorm / leng
    
    vnorm = cross(cross(axis0, axis1, dim=1), hnorm, dim=1)
    
    # Adjust normal based on occupancy
    tmp = sum((chk0 - spnt) * vnorm, dim=1, keepdim=True)
    vnorm = where((abs(tmp) > ERR) & bitwise_xor(tmp > 0, locc != occ0), -vnorm, vnorm)
    
    tmp = sum((chk1 - spnt) * vnorm, dim=1, keepdim=True)
    vnorm = where((abs(tmp) > ERR) & bitwise_xor(tmp > 0, locc != occ1), -vnorm, vnorm)
    
    tmp = sum((chk2 - spnt) * vnorm, dim=1, keepdim=True)
    vnorm = where((abs(tmp) > ERR) & bitwise_xor(tmp > 0, locc != occ2), -vnorm, vnorm)
    
    tmp = sum((chk3 - spnt) * vnorm, dim=1, keepdim=True)
    vnorm = where((abs(tmp) > ERR) & bitwise_xor(tmp > 0, locc != occ3), -vnorm, vnorm)
    
    # Vertical search
    left = _2d_linbin(pnts, spnt, locc, vnorm, VERTICAL_RANGE, VERTICAL_LINEAR_SEARCH, VERTICAL_BINARY_SEARCH)
    vl = sum((left - spnt) * vnorm, dim=1, keepdim=True)
    cndt |= abs(vl) < ERR
    
    # Horizontal search
    lleft = _2d_linbin(pnts, left, locc, hnorm, HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH) - lcord
    lh = sum(lleft * hnorm, dim=1, keepdim=True)
    rleft = _2d_linbin(pnts, left, locc, -hnorm, HORIZNTL_RANGE, HORIZNTL_LINEAR_SEARCH, HORIZNTL_BINARY_SEARCH) - rcord
    rh = sum(rleft * hnorm, dim=1, keepdim=True)
    
    denm = rh - lh
    cndt |= abs(denm) < ERR
    
    rslt = ((0.5 * (rh + lh)) * hnorm + vl * vnorm) * (leng / denm)
    
    cndt |= sum(rslt * vnorm, dim=1, keepdim=True) < 0
    rslt = where(cndt, zeros_like(rslt), rslt)
    
    return spnt + rslt

def _solve_QEF(pidx1_g2, cases, occs, GRID, GRID2, GRID3, ERR, QEF_REG, qvntab, pfvtab, vitogi, cstrtab, val_pnts):
    vertices = []
    for i in range(3, 8):
        idx = where(qvntab[val_, cases] == i)
        vtcs = pfvtab[val0_, cas, :2*i+1]
        vitg = vitogi[vtcs]
        
        pnti = vitg[:,:,3] * GRID3 + (pid3[:,0] + vitg[:,:,0]) * GRID2 + (pid3[:,1] + vitg[:,:,1]) * GRID + (pid3[:,2] + vitg[:,:,2])
        pidx = get_global_indices(pnti)
        vert = val_pnts[pidx]
        vert += vitg[:,:3]
        
        _1vt = vert[:, 1::2]
        qnrm = cross(_1vt - vert[:, :-1:2], _1vt - vert[:, 2::2])
        cntr = mean(_1vt, dim=1)
        
        # Normalize qnrm
        leng = norm(qnrm, dim=1, keepdim=True)
        cndt = leng < ERR
        leng[cndt] = 1.0
        qnrm = qnrm / leng
        
        # Solve QEF
        A = cat(qnrm, QEF_REG * eye)
        b = cat(sum(_1vt * qnrm), QEF_REG * cntr)
        qefr = lstsq(A, b).solution
        
        if all(cndt):
            qefr = cntr
            
        ctoq = qefr - cpnt
        leng = norm(ctoq, dim=1, keepdim=True)
        
        # Compute t
        t = compute_intersection_t(ctoq, cstrtab)
        
        # Handle NaNs and negatives
        t = where(isnan(t) | (t < 0), leng + SEP, t)
        t = t / leng
        t = clamp(t, 0, 1)
        
        final_vertex = cpnt + t * ctoq
        vertices.append(final_vertex)
    return vertices
```

**Parameters with Typical Values:**
*   `ERR`: Small threshold, e.g., $10^{-6}$ [S1, unofficial].
*   `QEF_REG`: Regularization constant, e.g., $10^{-3}$ [S1, unofficial].
*   `SEP`: Small constant for handling degenerate cases, e.g., $10^{-6}$ [S1, unofficial].
*   `VERTICAL_RANGE`, `HORIZNTL_RANGE`: Search ranges, typically related to grid resolution [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Normals:** If edge lengths are below `ERR`, normals are normalized to 1.0 to avoid division by zero [S1, unofficial].
*   **Ambiguity Correction:** If `amb_chks[pidx3]` is true, the occupancy value is inverted to resolve topological ambiguities [S1, unofficial].
*   **NaN Handling:** NaNs in intersection parameters are replaced with `leng + SEP` to ensure valid vertex positions [S1, unofficial].

## Subtopics

### QEM-based Vertex Relocation
A vertex relocation method used in the Mesh Generation phase to move vertices to the isosurface, using the Quadric Error Metric (QEM) instead of full QEF minimization to reduce computational complexity and register usage on parallel hardware [S2, unofficial]. Unlike pseudoinverse QEF minimization which is complex and penalizes performance on parallel hardware due to register limits, QEM minimization is stable with supporting planes and produces accurate results sufficient for volume preservation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method involves a linear scan of the grid cells, with constant-time lookups for topology and a constant-time least-squares solve for QEF. The 2D search involves binary/linear searches, which are logarithmic or linear in the search range. Overall, it is $O(N)$ where $N$ is the number of surface cells.
*   **Memory Complexity:** Requires storage for the occupancy grid, lookup tables, and the output mesh. The lookup tables are constant size. Memory usage is $O(N)$ for the output vertices.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of surface cells. Parallelization is straightforward as each cell is processed independently.
*   **Strengths:** Produces watertight meshes, handles ambiguous cases effectively, and reduces artifacts compared to standard Marching Cubes.
*   **Limitations:** More complex than standard Marching Cubes due to the QEF optimization step. The 2D search can be computationally expensive for large search ranges.
*   **When to Use:** When high-quality, watertight meshes are required from voxel data, especially in ambiguous topological configurations.
*   **When Not to Use:** When computational resources are extremely limited and approximate meshes are acceptable.

## Background
The method originates from the ODC (Octree-based Dual Contouring) project by KAIST Visual AI Group [S1, unofficial]. It combines the robustness of Marching Cubes with the geometric accuracy of QEF minimization, similar to Dual Contouring but adapted for structured grids. The QEM-based variant is discussed in the context of terrain visualization [S2, unofficial].

## Sources
[S1] KAIST-Visual-AI-Group/ODC, https://github.com/KAIST-Visual-AI-Group/ODC, fetched 2026-10-26 (unofficial)
[S2] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
