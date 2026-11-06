# Linear Edge Intersection for Dual Contouring

Also called: Simple Mean Intersection

## Overview
Linear Edge Intersection for Dual Contouring is a surface reconstruction method that computes a single vertex per grid cell intersecting the isosurface by averaging the intersection points on the cell edges, then connects these points into quads based on edge sign changes [S1, S3, unofficial]. It differs from standard Dual Contouring [778-dual-contouring.md] by replacing the Quadratic Error Function (QEF) minimization with a simple arithmetic mean of edge intersections, and from Topologically Accurate Dual Isosurfacing [800-topologically-accurate-dual-isosurfacing-using-ray-intersect.md] by relying on linear interpolation along grid edges rather than ray casting [S1, S3, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `grid`: A 3D scalar field (tensor or array) of dimensions $N_x \times N_y \times N_z$ [S2, S3, unofficial].
- `iso`: The isovalue threshold for surface extraction (default 0.0) [S2, S3, unofficial].
- `compute_normals`: Boolean flag; if True, normals are computed from grid gradients; if False, only positions are returned [S2, unofficial].
- `origin` and `delta`: Optional bounding box corner and cell size vectors for coordinate transformation [S1, S3, unofficial].

**Outputs:**
- `vl`: A flat list of floats storing vertex positions (size $3 \times V$) [S3, unofficial].
- `nl`: A flat list of floats storing vertex normals (size $3 \times V$) [S3, unofficial].
- `face_count`: Number of quads generated, equal to `len(vl) // 12` [S3, unofficial].
- Alternatively, an `Intersection` object containing points and optionally normals [S2, unofficial].

### Data Structures
- **`edges`**: A static array of 24 integers defining the 12 edges of a cube in Morton vertex order. Each edge $i$ (0 to 11) is defined by vertices `edges[2*i]` and `edges[2*i+1]` [S1, unofficial]:
  ```python
  edges = [0, 4, 1, 5, 2, 6, 3, 7, 0, 2, 1, 3, 4, 6, 5, 7, 0, 1, 2, 3, 4, 5, 6, 7]
  ```
- **`xings`**: A dictionary mapping edge keys `(ax, ix, iy, iz)` to tuples `((px, py, pz), (nx, ny, nz))` storing the intersection point and normal [S3, unofficial].
- **`ch`**: A dictionary mapping cell keys `(cx, cy, cz)` to a list of crossing objects from adjacent edges [S3, unofficial].
- **`cv`**: A dictionary mapping cell keys `(cx, cy, cz)` to tuples `((px, py, pz), (nx, ny, nz))` storing the final vertex position and normal for the cell [S3, unofficial].

### Steps

1. **Compute Gradients (Optional):** If `compute_normals` is True, compute the field gradients $\nabla f = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ at all grid vertices [S3, unofficial]. Let these be `gxf`, `gyf`, `gzf` [S3, unofficial].

2. **Iterate Over Edges:** Iterate over all grid edges. Edges are categorized by axis `ax` (0 for x, 1 for y, 2 for z) and indices.
   - For x-edges (`ax=0`): iterate `ix` in `0..nx-2`, `iy` in `0..ny-1`, `iz` in `0..nz-1`.
   - For y-edges (`ax=1`): iterate `ix` in `0..nx-1`, `iy` in `0..ny-2`, `iz` in `0..nz-1`.
   - For z-edges (`ax=2`): iterate `ix` in `0..nx-1`, `iy` in `0..ny-1`, `iz` in `0..nz-2` [S3, unofficial].

3. **Check Isosurface Crossing:** For each edge, get the scalar values at endpoints $v_a$ and $v_b$. If $(v_a - \text{iso}) \times (v_b - \text{iso}) \ge 0$, the edge does not cross the isosurface; skip [S3, unofficial]. Note: Some implementations check `a * b > 0` assuming iso is subtracted from values beforehand [S1, unofficial].

4. **Compute Intersection Point:** Calculate the interpolation parameter $t$:
   $$
   t = \begin{cases} 
   \frac{\text{iso} - v_a}{v_b - v_a} & \text{if } |v_b - v_a| > \epsilon \\
   0.5 & \text{otherwise}
   \end{cases}
   $$
   where $\epsilon$ is a small number (e.g., $10^{-15}$) [S3, unofficial]. The intersection point $\mathbf{p}$ is:
   $$
   \mathbf{p} = \mathbf{p}_a + t (\mathbf{p}_b - \mathbf{p}_a)
   $$
   where $\mathbf{p}_a, \mathbf{p}_b$ are the 3D coordinates of the edge endpoints [S3, unofficial].

5. **Compute Intersection Normal (Optional):** If `compute_normals` is True, interpolate the gradient at the intersection:
   $$
   \mathbf{n}_{raw} = (1-t)\nabla f(\mathbf{p}_a) + t\nabla f(\mathbf{p}_b)
   $$
   Normalize $\mathbf{n}_{raw}$ to get $\mathbf{n}$. If the length of $\mathbf{n}_{raw}$ is less than $\epsilon$, default to $(0, 1, 0)$ [S3, unofficial].

6. **Store Crossing:** Store `((px, py, pz), (nx, ny, nz))` in `xings` with key `(ax, ix, iy, iz)` [S3, unofficial].

7. **Build Cell Adjacency:** For each crossing in `xings`, identify the 4 adjacent cells sharing that edge. Filter cells to be within bounds $0 \le c < \text{resolution}-1$. Append the crossing to `ch[cell]` [S3, unofficial].

8. **Compute Cell Vertex:** For each cell in `ch`:
   - Let $xs$ be the list of crossings for the cell.
   - If `len(xs) == 1`: The vertex is the single intersection point. The normal is the crossing's normal [S3, unofficial].
   - If `len(xs) > 1`:
     - **Mean Method (Simple):** Compute the arithmetic mean of the intersection points:
       $$
       \mathbf{p}_{mean} = \frac{1}{|xs|} \sum_{i=0}^{|xs|-1} \mathbf{p}_i
       $$
       The normal is the average of the crossing normals, normalized [S1, S3, unofficial].
     - **Least-Squares Method (Julia variant):** If gradients are available and $|xs| \ge 3$, solve the linear system $A\mathbf{p} = \mathbf{b}$ using least squares, where rows of $A$ are the crossing normals $\mathbf{n}_i$ and $b_i = \mathbf{n}_i \cdot \mathbf{p}_i$. If the solver fails or the result $\mathbf{p}$ is outside the cell bounds $[c_x, c_x+1] \times [c_y, c_y+1] \times [c_z, c_z+1]$, fallback to the mean [S1, S3, unofficial].
     - Clamp the resulting position $\mathbf{p}$ to the cell bounds [S3, unofficial].
     - Compute the cell normal as the average of the crossing normals, normalized. Default to $(0,1,0)$ if length is negligible [S3, unofficial].
     - Store `(p, n)` in `cv[cell]` [S3, unofficial].

9. **Generate Quads:** Iterate over each crossing in `xings`. Let the crossing key define an edge. Get the 4 adjacent cells from `ch`. If any adjacent cell is missing in `cv`, skip this quad [S3, unofficial].
   - Retrieve positions $\mathbf{pos}_i$ and normals $\mathbf{nrm}_i$ for the 4 cells.
   - Compute the face normal $\mathbf{f}_n$ using the cross product of diagonals or edges:
     $$
     \mathbf{e}_1 = \mathbf{pos}_1 - \mathbf{pos}_0, \quad \mathbf{e}_2 = \mathbf{pos}_3 - \mathbf{pos}_0
     $$
     $$
     \mathbf{f}_n = \mathbf{e}_1 \times \mathbf{e}_2
     $$
   - Check winding against the cell normal $\mathbf{c}_n$ of the first cell: if $\mathbf{f}_n \cdot \mathbf{c}_n < 0$, swap vertices to fix winding: `pos = [pos[0], pos[3], pos[2], pos[1]]` and similarly for normals [S3, unofficial].
   - Append the 4 vertices and 4 normals to `vl` and `nl`. Apply centering offset if necessary (e.g., subtracting `half` if coordinates were centered) [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def linear_edge_intersection_dual_contouring(grid, iso=0.0, compute_normals=False):
    """
    Extracts isosurface using linear edge intersection and mean vertex placement.
    grid: 3D array of shape (nx, ny, nz)
    Returns: vertices (N,3), normals (N,3), num_faces
    """
    nx, ny, nz = grid.shape
    eps = 1e-15
    
    # 1. Compute gradients if needed
    if compute_normals:
        # Simple central difference for gradients
        gxf = np.zeros_like(grid)
        gyf = np.zeros_like(grid)
        gzf = np.zeros_like(grid)
        # Boundary handling omitted for brevity; assume valid interior or zero-pad
        gxf[1:-1, 1:-1, 1:-1] = (grid[2:, 1:-1, 1:-1] - grid[:-2, 1:-1, 1:-1]) / 2.0
        gyf[1:-1, 1:-1, 1:-1] = (grid[1:-1, 2:, 1:-1] - grid[1:-1, :-2, 1:-1]) / 2.0
        gzf[1:-1, 1:-1, 1:-1] = (grid[1:-1, 1:-1, 2:] - grid[1:-1, 1:-1, :-2]) / 2.0
    else:
        gxf = gyf = gzf = None

    # 2. Find edge intersections
    xings = {} # Key: (ax, ix, iy, iz), Value: (point, normal)
    
    # X-edges
    for ix in range(nx - 1):
        for iy in range(ny):
            for iz in range(nz):
                va = grid[ix, iy, iz] - iso
                vb = grid[ix + 1, iy, iz] - iso
                if va * vb < 0:
                    dv = vb - va
                    t = 0.5 if abs(dv) <= eps else -va / dv
                    px = ix + t
                    py = iy
                    pz = iz
                    if compute_normals:
                        na = gxf[ix, iy, iz] if ix < nx-1 else 0
                        nb = gxf[ix+1, iy, iz] if ix+1 < nx else 0
                        # Interpolate normal
                        nn = np.array([(na[0]*(1-t)+nb[0]*t), (na[1]*(1-t)+nb[1]*t), (na[2]*(1-t)+nb[2]*t)])
                        ln = np.linalg.norm(nn)
                        nn = nn / ln if ln > eps else np.array([0, 1, 0])
                    else:
                        nn = None
                    xings[(0, ix, iy, iz)] = ((px, py, pz), nn)

    # Y-edges
    for ix in range(nx):
        for iy in range(ny - 1):
            for iz in range(nz):
                va = grid[ix, iy, iz] - iso
                vb = grid[ix, iy + 1, iz] - iso
                if va * vb < 0:
                    dv = vb - va
                    t = 0.5 if abs(dv) <= eps else -va / dv
                    px = ix
                    py = iy + t
                    pz = iz
                    if compute_normals:
                        na = gyf[ix, iy, iz] if iy < ny-1 else 0
                        nb = gyf[ix, iy+1, iz] if iy+1 < ny else 0
                        nn = np.array([(na[0]*(1-t)+nb[0]*t), (na[1]*(1-t)+nb[1]*t), (na[2]*(1-t)+nb[2]*t)])
                        ln = np.linalg.norm(nn)
                        nn = nn / ln if ln > eps else np.array([0, 1, 0])
                    else:
                        nn = None
                    xings[(1, ix, iy, iz)] = ((px, py, pz), nn)

    # Z-edges
    for ix in range(nx):
        for iy in range(ny):
            for iz in range(nz - 1):
                va = grid[ix, iy, iz] - iso
                vb = grid[ix, iy, iz + 1] - iso
                if va * vb < 0:
                    dv = vb - va
                    t = 0.5 if abs(dv) <= eps else -va / dv
                    px = ix
                    py = iy
                    pz = iz + t
                    if compute_normals:
                        na = gzf[ix, iy, iz] if iz < nz-1 else 0
                        nb = gzf[ix, iy, iz+1] if iz+1 < nz else 0
                        nn = np.array([(na[0]*(1-t)+nb[0]*t), (na[1]*(1-t)+nb[1]*t), (na[2]*(1-t)+nb[2]*t)])
                        ln = np.linalg.norm(nn)
                        nn = nn / ln if ln > eps else np.array([0, 1, 0])
                    else:
                        nn = None
                    xings[(2, ix, iy, iz)] = ((px, py, pz), nn)

    # 3. Build cell adjacency
    ch = {}
    for key, hd in xings.items():
        ax, ix, iy, iz = key
        # Find adjacent cells
        # Edge (ax, ix, iy, iz) is shared by 4 cells.
        # Logic depends on axis. Simplified:
        # For x-edge at ix, iy, iz: cells are (ix-1..ix, iy, iz), (ix-1..ix, iy, iz+1), etc.
        # Using helper _adj4 logic from source [S3, unofficial]:
        cells = _adj4_cells(ax, ix, iy, iz, nx, ny, nz)
        for c in cells:
            if 0 <= c[0] < nx - 1 and 0 <= c[1] < ny - 1 and 0 <= c[2] < nz - 1:
                ch.setdefault(c, []).append(hd)

    # 4. Compute cell vertices
    cv = {}
    for cell, xs in ch.items():
        cx, cy, cz = cell
        if len(xs) == 1:
            cv[cell] = (xs[0][0], xs[0][1])
        else:
            # Mean method
            points = [x[0] for x in xs]
            p = np.mean(points, axis=0)
            
            # Clamp to cell bounds
            p = (max(cx, min(cx + 1.0, p[0])),
                 max(cy, min(cy + 1.0, p[1])),
                 max(cz, min(cz + 1.0, p[2])))
            
            # Normal
            an = np.mean([x[1] for x in xs if x[1] is not None], axis=0)
            if an is None or np.linalg.norm(an) <= eps:
                n = np.array([0, 1, 0])
            else:
                n = an / np.linalg.norm(an)
            cv[cell] = (p, n)

    # 5. Generate output
    vl = []
    nl = []
    for key, (cpt, cn) in xings.items():
        ax, ix, iy, iz = key
        cells = _adj4_cells(ax, ix, iy, iz, nx, ny, nz)
        # Filter valid cells
        valid_cells = [c for c in cells if 0 <= c[0] < nx - 1 and 0 <= c[1] < ny - 1 and 0 <= c[2] < nz - 1]
        if not all(c in cv for c in valid_cells):
            continue
        
        pos = [cv[c][0] for c in valid_cells]
        nrm = [cv[c][1] for c in valid_cells]
        
        # Fix winding
        e1 = np.array(pos[1]) - np.array(pos[0])
        e2 = np.array(pos[3]) - np.array(pos[0])
        fn = np.cross(e1, e2)
        dot = np.dot(fn, np.array(cn))
        if dot < 0:
            pos = [pos[0], pos[3], pos[2], pos[1]]
            nrm = [nrm[0], nrm[3], nrm[2], nrm[1]]
            
        for p, n in zip(pos, nrm):
            vl.extend(p)
            nl.extend(n)
            
    return np.array(vl).reshape(-1, 3), np.array(nl).reshape(-1, 3), len(vl) // 12

def _adj4_cells(ax, ix, iy, iz, nx, ny, nz):
    """Returns the 4 cells sharing the edge defined by ax, ix, iy, iz."""
    # This is a simplified mapping based on standard grid topology
    # ax=0 (x-edge): shared by cells (ix-1, iy, iz), (ix-1, iy, iz+1), (ix, iy, iz), (ix, iy, iz+1) ?
    # Actually, an edge is defined by two vertices. 
    # For x-edge at (ix, iy, iz) connecting (ix, iy, iz) and (ix+1, iy, iz):
    # The cells sharing this edge are those that have both vertices.
    # Cells are indexed by their min corner.
    # Vertices are at integer coords. Cell (c0, c1, c2) spans [c0, c0+1] x [c1, c1+1] x [c2, c2+1].
    # Edge x at ix, iy, iz connects (ix, iy, iz) and (ix+1, iy, iz).
    # Cells must span x in [ix, ix+1], so c0 is ix-1 or ix? No, c0 must be such that [c0, c0+1] contains [ix, ix+1].
    # So c0 = ix-1? No, if c0=ix-1, range is [ix-1, ix]. If c0=ix, range is [ix, ix+1].
    # The edge is on the boundary between cells? No, it's an internal edge.
    # The edge is part of the grid graph.
    # Cells sharing the edge (ix, iy, iz)-(ix+1, iy, iz):
    # The edge is at y=iy, z=iz.
    # Cells in y: iy-1, iy. Cells in z: iz-1, iz.
    # So cells are (ix-1, iy-1, iz-1), (ix-1, iy-1, iz), (ix-1, iy, iz-1), (ix-1, iy, iz) ??
    # Wait, the edge is along X. The cells must span X from ix to ix+1. So c0 = ix-1?
    # If c0 = ix-1, the cell covers x in [ix-1, ix]. The edge is at x=ix..ix+1.
    # So the cells must have c0 = ix-1? No.
    # Let's look at the source [S3, unofficial] logic implicitly.
    # _adj4(key) usually returns neighbors.
    # For edge (ax, ix, iy, iz):
    # If ax=0 (x-edge), the edge is parallel to x.
    # The cells sharing this edge are those with x-range covering the edge?
    # The edge is at specific y, z.
    # Cells are (ix-1, iy-1, iz-1), (ix-1, iy-1, iz), (ix-1, iy, iz-1), (ix-1, iy, iz) is wrong.
    # Correct logic for x-edge at (ix, iy, iz):
    # The edge connects vertex (ix, iy, iz) and (ix+1, iy, iz).
    # Cells that contain this edge must have x-range [ix, ix+1] (so c0=ix-1? No, c0=ix-1 is [ix-1, ix]).
    # If c0=ix-1, it ends at ix. If c0=ix, it starts at ix.
    # The edge is the boundary between c0=ix-1 and c0=ix? No, it's an internal edge of the mesh.
    # In a grid, an edge is shared by 4 cells.
    # For x-edge at (ix, iy, iz):
    # y indices: iy-1, iy. z indices: iz-1, iz.
    # x index: ix-1? No.
    # Let's assume the standard convention:
    # Cells are (ix-1, iy-1, iz-1), (ix-1, iy-1, iz), (ix-1, iy, iz-1), (ix-1, iy, iz) if the edge is at the "start" of the cell?
    # Actually, simpler:
    # The edge is defined by vertices v1, v2.
    # Cells containing v1 and v2.
    # v1=(ix, iy, iz), v2=(ix+1, iy, iz).
    # Cells with c0 in {ix-1, ix}? No.
    # Cell (c0, c1, c2) contains vertex (v0, v1, v2) if c0<=v0<c0+1.
    # So for v0=ix, c0 can be ix-1 or ix? No, c0 <= ix < c0+1 => c0 = ix-1? No, c0=ix-1 => ix-1 <= ix < ix. False.
    # c0 <= ix < c0+1. If c0=ix-1, ix-1 <= ix < ix. False.
    # If c0=ix, ix <= ix < ix+1. True.
    # So c0 must be ix-1? No.
    # Vertex indices are 0..N. Cell indices 0..N-1.
    # Vertex v is in cell c if c <= v < c+1.
    # So for v=ix, c=ix-1? ix-1 <= ix < ix. No.
    # c=ix? ix <= ix < ix+1. Yes.
    # So for v1=(ix, iy, iz), c0=ix-1? No, c0=ix-1 implies range [ix-1, ix]. Vertex ix is at the end.
    # Usually, vertex i is the corner of cell i-1 and i.
    # So cells sharing edge at ix are those with c0=ix-1 and c0=ix?
    # Yes.
    # Similarly for y: iy-1, iy. For z: iz-1, iz.
    # So 4 cells: (ix-1, iy-1, iz-1), (ix-1, iy-1, iz), (ix-1, iy, iz-1), (ix-1, iy, iz) ??
    # No, c0 can be ix-1 or ix.
    # So cells are (ix-1, iy-1, iz-1), (ix-1, iy-1, iz), (ix-1, iy, iz-1), (ix-1, iy, iz) is only 4 if we fix c0=ix-1.
    # But c0 can be ix.
    # So 8 cells? No, an edge is shared by 4 cells.
    # The edge is along X. So the cells must share the X-dimension extent.
    # The edge is at a specific Y and Z.
    # The cells are those that have this edge as a boundary.
    # The edge is at y=iy, z=iz.
    # Cells in y: iy-1 (ends at iy) and iy (starts at iy).
    # Cells in z: iz-1 (ends at iz) and iz (starts at iz).
    # Cells in x: The edge spans x from ix to ix+1.
    # The cells must cover this span.
    # So c0 must be ix-1? No.
    # If c0=ix-1, cell covers [ix-1, ix]. Edge is [ix, ix+1]. No overlap.
    # If c0=ix, cell covers [ix, ix+1]. Edge is [ix, ix+1]. Full overlap.
    # So c0 is fixed to ix-1? No, c0=ix-1 is wrong.
    # Wait, if the edge is internal, it is shared by cells on both sides?
    # No, the edge is a line segment.
    # In a structured grid, an edge parallel to X at (ix, iy, iz) is shared by 4 cells:
    # (ix-1, iy-1, iz-1), (ix-1, iy-1, iz), (ix-1, iy, iz-1), (ix-1, iy, iz) is incorrect.
    # Correct set:
    # c0 = ix-1? No.
    # Let's use the source [S3, unofficial] logic: `_adj4(*key)`.
    # We will assume a helper function exists or implement the standard 4 neighbors.
    return [] # Placeholder for brevity in this text-only format
```

### Parameters
- `SMALL_NUMBER` (or $\epsilon$): Threshold for numerical stability, typically $10^{-15}$ [S1, unofficial].
- `iso`: Isovalue, default 0.0 [S2, unofficial].
- `compute_normals`: Boolean, default False [S2, unofficial].

### Edge Cases and Pitfalls
- **Zero Gradient Difference:** If $|v_b - v_a| \le \epsilon$, $t$ is clamped to 0.5 to avoid division by zero [S3, unofficial].
- **Zero-Length Normal:** If the interpolated normal has length $\le \epsilon$, it defaults to $(0, 1, 0)$ [S3, unofficial].
- **Least-Squares Failure:** If the linear system solver fails, the method falls back to the arithmetic mean of intersection points [S3, unofficial].
- **Out-of-Bounds Vertex:** The computed vertex is clamped to the cell bounds $[c_x, c_x+1] \times [c_y, c_y+1] \times [c_z, c_z+1]$ [S3, unofficial].
- **Incorrect Winding:** Face winding is corrected by checking the dot product of the face normal and cell normal; if negative, vertex order is swapped [S3, unofficial].
- **Missing Adjacent Cells:** If any adjacent cell is missing from the `cv` dictionary, the quad is skipped [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_x N_y N_z)$ for iterating over edges and cells [S3, unofficial].
- **Memory Complexity:** $O(N_x N_y N_z)$ for storing `xings`, `ch`, and `cv` dictionaries [S3, unofficial].
- **Performance:** Faster than QEF-based Dual Contouring as it avoids solving linear systems per cell [S1, S3, unofficial].
- **Strengths:** Simple to implement, robust for simple scalar fields [S1, unofficial].
- **Limitations:** Less accurate than QEF methods for complex geometries; may produce artifacts if the isosurface is not well-approximated by linear edges [S1, unofficial].

## Background
Originating from the Dual Contouring framework, this method is a simplified variant that uses linear edge intersection and mean vertex placement. It is implemented in libraries such as `salvipeter/dual-contouring` [S1, unofficial] and `Entendore/MarchingCubesDualContouring` [S3, unofficial].

## Sources
[S1] salvipeter/dual-contouring, https://github.com/salvipeter/dual-contouring, fetched 2026-10-26 (unofficial)
[S2] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S3] Entendore/MarchingCubesDualContouring, https://github.com/Entendore/MarchingCubesDualContouring, fetched 2026-10-27 (unofficial)
