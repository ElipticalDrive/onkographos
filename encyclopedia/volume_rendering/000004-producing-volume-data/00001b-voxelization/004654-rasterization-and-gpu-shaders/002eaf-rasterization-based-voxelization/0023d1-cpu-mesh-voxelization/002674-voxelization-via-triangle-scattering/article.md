# Voxelization via Triangle Scattering

Also called: Phase 2, voxelize

## Overview
Voxelization via Triangle Scattering converts a triangle mesh into a 3D grid of voxels to identify the air region for acoustic simulation [S1, unofficial]. It rasterizes surface triangles into the grid using an exact overlap test and isolates the connected air volume via flood fill, avoiding acceleration structures like BVHs [S1, unofficial]. It differs from its parent [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md) by scattering triangles into cells rather than iterating cells against triangles, and from [Inner Loop Optimizations for CPU Mesh Voxelization](4048-inner-loop-optimizations-for-cpu-mesh-voxelization.md) by prioritizing exact geometric overlap over conservative bounding box approximations [S1, unofficial].

## How it works
**Inputs:** A list of triangles (each defined by 3 vertices in 3D space) and a target grid resolution (cell size $\Delta$) [S1, unofficial].
**Outputs:** A 3D boolean grid where `true` indicates solid (geometry) and `false` indicates air [S1, unofficial]. A `leaked` flag indicating if the air region is not sealed [S1, unofficial].

**Data Structures:**
*   `Grid`: A 3D array of booleans with dimensions $(N_x+2) \times (N_y+2) \times (N_z+2)$, providing a 1-cell padding on all sides [S1, unofficial].
*   `Triangle`: Struct with fields `v0`, `v1`, `v2` (each a 3D vector).

**Algorithm Steps:**

1.  **Grid Initialization:**
    Initialize the `Grid` to all `false` (air) [S1, unofficial]. The padding cells are conceptually exterior [S1, unofficial].

2.  **Triangle Scattering (Rasterization):**
    For each triangle $T$ in the input mesh:
    a.  Compute the axis-aligned bounding box (AABB) of $T$ in world coordinates [S1, unofficial].
    b.  Convert the AABB to grid coordinates to determine the range of cells $[i_{min}, i_{max}] \times [j_{min}, j_{max}] \times [k_{min}, k_{max}]$ that intersect the triangle's bounding box [S1, unofficial].
    c.  Iterate over every cell $(i, j, k)$ in this range [S1, unofficial].
    d.  Perform an **exact 13-axis Separating Axis Theorem (SAT)** overlap test between the triangle $T$ and the voxel cell $C_{i,j,k}$ [S1, unofficial].
        *   The 13 axes consist of the 3 face normals of the cell, the 1 face normal of the triangle, and the 9 cross products of the triangle edges with the cell axes [S1, unofficial].
        *   If a separating axis is found, the triangle and cell do not overlap [S1, unofficial].
        *   If no separating axis is found, they overlap [S1, unofficial].
    e.  If they overlap, mark `Grid[i, j, k] = true` (solid) [S1, unofficial].

3.  **Flood Fill for Air Region:**
    a.  Select a seed point $P_{seed}$ inside the room [S1, unofficial].
    b.  Map $P_{seed}$ to grid coordinates $(i_s, j_s, k_s)$ [S1, unofficial].
    c.  If `Grid[i_s, j_s, k_s]` is `true` (solid), the seed is invalid [S1, unofficial]. Perform **Seed Relocation**:
        *   Generate candidate cells ordered by increasing radius from the seed, then by distance to the grid center [S1, unofficial].
        *   For each candidate, perform a temporary flood fill [S1, unofficial].
        *   Select the first candidate that results in a sealed fill (does not reach the padded rim) [S1, unofficial].
    d.  Perform a 6-connected flood fill starting from the valid seed cell [S1, unofficial].
    e.  Mark all reachable cells as `false` (air) [S1, unofficial].
    f.  If the fill reaches any cell in the 1-cell padding, set `leaked = true` and issue a warning. Do not throw an error [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from collections import deque

def voxelize(triangles, grid_dims, cell_size):
    """
    Voxelize a triangle mesh into a grid.
    grid_dims: (Nx, Ny, Nz) number of interior cells.
    cell_size: float, size of one voxel.
    """
    # 1. Initialize grid with padding
    shape = (grid_dims[0] + 2, grid_dims[1] + 2, grid_dims[2] + 2)
    grid = [[[False] * shape[2] for _ in range(shape[1])] for _ in range(shape[0])]

    # Vector ops
    def cross(a, b):
        return (a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0])
    def dot(a, b):
        return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

    # Helper: Convert world point to grid index
    def world_to_grid(p):
        return (
            int(math.floor(p[0] / cell_size)) + 1,
            int(math.floor(p[1] / cell_size)) + 1,
            int(math.floor(p[2] / cell_size)) + 1
        )

    # Helper: Get cell AABB in world coordinates
    def get_cell_aabb(i, j, k):
        x0, y0, z0 = (i - 1) * cell_size, (j - 1) * cell_size, (k - 1) * cell_size
        x1, y1, z1 = i * cell_size, j * cell_size, k * cell_size
        return (x0, y0, z0, x1, y1, z1)

    # Helper: 13-axis SAT overlap test
    def triangle_cell_overlap(tri, i, j, k):
        x0, y0, z0, x1, y1, z1 = get_cell_aabb(i, j, k)
        v0, v1, v2 = tri
        tri_edges = [(v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2]),
                     (v2[0]-v1[0], v2[1]-v1[1], v2[2]-v1[2]),
                     (v0[0]-v2[0], v0[1]-v2[1], v0[2]-v2[2])]
        cell_corners = [
            (x0, y0, z0), (x1, y0, z0), (x0, y1, z0), (x1, y1, z0),
            (x0, y0, z1), (x1, y0, z1), (x0, y1, z1), (x1, y1, z1)
        ]
        tri_verts = [v0, v1, v2]

        # 1. Box face normals (3 axes)
        if max(v[0] for v in tri_verts) < x0 or min(v[0] for v in tri_verts) > x1: return False
        if max(v[1] for v in tri_verts) < y0 or min(v[1] for v in tri_verts) > y1: return False
        if max(v[2] for v in tri_verts) < z0 or min(v[2] for v in tri_verts) > z1: return False

        # 2. Triangle face normal (1 axis)
        n = cross(tri_edges[0], tri_edges[1])
        n_len_sq = dot(n, n)
        if n_len_sq > 0:
            t_proj = [dot(v, n) / n_len_sq for v in tri_verts]
            c_proj = [dot(c, n) / n_len_sq for c in cell_corners]
            if max(t_proj) < min(c_proj) or min(t_proj) > max(c_proj): return False

        # 3. Edge cross products (9 axes)
        for te in tri_edges:
            for axis in [(1,0,0), (0,1,0), (0,0,1)]:
                ax = cross(te, axis)
                ax_len_sq = dot(ax, ax)
                if ax_len_sq == 0: continue
                t_proj = [dot(v, ax) / ax_len_sq for v in tri_verts]
                c_proj = [dot(c, ax) / ax_len_sq for c in cell_corners]
                if max(t_proj) < min(c_proj) or min(t_proj) > max(c_proj): return False

        return True

    # 2. Scatter Triangles
    for tri in triangles:
        min_x = min(v[0] for v in tri)
        max_x = max(v[0] for v in tri)
        min_y = min(v[1] for v in tri)
        max_y = max(v[1] for v in tri)
        min_z = min(v[2] for v in tri)
        max_z = max(v[2] for v in tri)

        i_min = max(1, int(math.floor(min_x / cell_size)))
        i_max = min(grid_dims[0], int(math.ceil(max_x / cell_size)) - 1)
        j_min = max(1, int(math.floor(min_y / cell_size)))
        j_max = min(grid_dims[1], int(math.ceil(max_y / cell_size)) - 1)
        k_min = max(1, int(math.floor(min_z / cell_size)))
        k_max = min(grid_dims[2], int(math.ceil(max_z / cell_size)) - 1)

        for i in range(i_min, i_max + 1):
            for j in range(j_min, j_max + 1):
                for k in range(k_min, k_max + 1):
                    if triangle_cell_overlap(tri, i, j, k):
                        grid[i][j][k] = True

    # 3. Flood Fill
    seed_world = (grid_dims[0]*cell_size/2, grid_dims[1]*cell_size/2, grid_dims[2]*cell_size/2)
    si, sj, sk = world_to_grid(seed_world)
    si = max(1, min(si, grid_dims[0]))
    sj = max(1, min(sj, grid_dims[1]))
    sk = max(1, min(sk, grid_dims[2]))

    leaked = False

    # Seed relocation if solid
    if grid[si][sj][sk]:
        candidates = []
        for i in range(1, grid_dims[0]+1):
            for j in range(1, grid_dims[1]+1):
                for k in range(1, grid_dims[2]+1):
                    if not grid[i][j][k]:
                        dist_sq = (i-si)**2 + (j-sj)**2 + (k-sk)**2
                        center_dist_sq = (i - grid_dims[0]/2)**2 + (j - grid_dims[1]/2)**2 + (k - grid_dims[2]/2)**2
                        candidates.append((dist_sq, center_dist_sq, i, j, k))
        candidates.sort(key=lambda x: (x[0], x[1]))
        valid_seed = None
        for c in candidates:
            ci, cj, ck = c[2], c[3], c[4]
            if is_sealed_fill(grid, ci, cj, ck, grid_dims):
                valid_seed = (ci, cj, ck)
                break
        if valid_seed:
            si, sj, sk = valid_seed

    # Perform flood fill
    queue = deque([(si, sj, sk)])
    visited = set()
    visited.add((si, sj, sk))

    while queue:
        i, j, k = queue.popleft()
        if i == 0 or i == grid_dims[0]+1 or j == 0 or j == grid_dims[1]+1 or k == 0 or k == grid_dims[2]+1:
            leaked = True
            continue
        for di, dj, dk in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]:
            ni, nj, nk = i+di, j+dj, k+dk
            if 0 <= ni < shape[0] and 0 <= nj < shape[1] and 0 <= nk < shape[2]:
                if (ni, nj, nk) not in visited:
                    visited.add((ni, nj, nk))
                    if not grid[ni][nj][nk]:
                        queue.append((ni, nj, nk))

    return grid, leaked

def is_sealed_fill(grid, si, sj, sk, grid_dims):
    shape = (grid_dims[0] + 2, grid_dims[1] + 2, grid_dims[2] + 2)
    queue = deque([(si, sj, sk)])
    visited = set()
    visited.add((si, sj, sk))
    while queue:
        i, j, k = queue.popleft()
        if i == 0 or i == grid_dims[0]+1 or j == 0 or j == grid_dims[1]+1 or k == 0 or k == grid_dims[2]+1:
            return False
        for di, dj, dk in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]:
            ni, nj, nk = i+di, j+dj, k+dk
            if 0 <= ni < shape[0] and 0 <= nj < shape[1] and 0 <= nk < shape[2]:
                if (ni, nj, nk) not in visited and not grid[ni][nj][nk]:
                    visited.add((ni, nj, nk))
                    queue.append((ni, nj, nk))
    return True
```

**Parameters:**
*   **Overlap Test:** Exact 13-axis SAT. Conservative AABB tests are rejected because they thicken oblique walls, changing the air volume and modal frequencies [S1, unofficial].
*   **Padding:** 1 cell on all sides [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Exterior Bias:** Seeds on -x/-y/-z walls or within half a cell inward could previously leak into the exterior padding due to scan order bias. This is fixed by validating that the fill remains sealed [S1, unofficial].
*   **Leak Detection:** A gap of one cell drains the interior, making the 'air region' the whole bounding box. Silently decomposing this produces large, quiet nonsense. The algorithm reports a leak flag rather than throwing an error, allowing the caller to inspect the grid [S1, unofficial].
*   **Sealed Voids:** The 6-connected flood fill excludes sealed voids (e.g., a closed closet), marking them as solid. This is intentional as they are not part of the room's acoustic volume [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the surface area in cells (number of solid cells produced) rather than grid volume $\times \log(\text{triangles})$ [S1, unofficial].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the grid [G]. No acceleration structures (BVH) are required [S1, unofficial].
*   **Strengths:** No BVH construction overhead. Exact overlap test preserves geometry fidelity for acoustic modal frequencies. Leak detection allows the caller to inspect the grid [S1, unofficial].
*   **Limitations:** Staircasing of non-axis-aligned surfaces is intrinsic to grid methods. Corner attribution is ambiguous (first triangle to reach a cell wins) [S1, unofficial].
*   **When to use:** When exact geometry is critical for acoustic simulation and BVH construction is undesirable or too slow [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
This method was implemented as Phase 2 of the ARD (Acoustic Room Decomposition) pipeline in the `cram` project [S1, unofficial]. It was developed to replace a BVH-based approach specified in an earlier design, as scattering triangles into cells was found to be strictly better in terms of work proportional to surface area and avoiding acceleration structure overhead [S1, unofficial]. Phases 1–5 are done and tested together, with Phase 6 onward remaining [S1, unofficial].

## Sources
[S1] feat(ard): voxelization and rectangular decomposition (Phases 2 and 3), https://github.com/10log/cram/pull/192, fetched 2026-10-27 (unofficial)
