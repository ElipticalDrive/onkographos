# Solid Voxelization via 6-Direction Odd-Even Ray Parity

Also called: 6-direction majority vote, odd-even ray parity solid voxelization

## Overview
This method converts a normalized, watertight mesh into a solid voxel target by determining the inside/outside status of each voxel center using the parity of ray intersections along six axis-aligned directions [S1, unofficial]. It differs from its parent [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) by using deterministic, axis-aligned rays and a majority vote on parity rather than random rays and intersection counts, providing a robust, non-stochastic solution for solid interior filling [S1, unofficial].

## How it works
**Inputs:**
- `mesh`: A watertight mesh defined by vertex positions and triangle indices [S1, unofficial].
- `voxelMm`: The physical size of a voxel in millimeters [S1, unofficial].
- `colorId`: The identifier to assign to filled voxels [S1, unofficial].

**Outputs:**
- A `Map` with keys `"i,j,k"` mapping to `colorId` [S1, unofficial].
- A `seed` string representing the key of the seed voxel [S1, unofficial].
- Voxel centers are located at $(i, j, k) \cdot \text{voxelMm}$ in world coordinates [S1, unofficial].

**Data Structures:**
- **Per-Axis Uniform Triangle Grids:** Three shared grids are used (one per axis pair: XY, YZ, ZX) [S1, unofficial]. Each grid maps a 2D cell index (e.g., `"ui,vi"`) to a list of triangle indices that intersect that column [S1, unofficial].
- **Triangle Indices Retrieval:** Given a world coordinate $(x, y, z)$ and an axis direction, the relevant triangles are retrieved by mapping the two non-axis coordinates to grid cell indices [S1, unofficial].

**Algorithm Steps:**

1. **Grid Construction:** For each of the three axis pairs, construct a uniform grid covering the mesh bounding box [S1, unofficial]. Project each triangle onto the corresponding 2D plane and bucket its index into intersecting grid cells [S1, unofficial].
2. **Seed Selection:** Identify the lowest $k$ layer of voxels that have at least one filled voxel [S1, unofficial]. Find the $(i,j)$ bounding box center of these voxels [S1, unofficial]. Select the filled voxel on this lowest $k$ layer nearest to this $(i,j)$ center as the seed [S1, unofficial].
3. **Voxel Classification (for each voxel $(i, j, k)$):** Let the voxel center be $C = (i \cdot \text{voxelMm}, j \cdot \text{voxelMm}, k \cdot \text{voxelMm})$ [S1, unofficial]. Initialize `inside_votes = 0` [S1, unofficial]. For each of the six axis directions $D \in \{+X, -X, +Y, -Y, +Z, -Z\}$: cast a ray from $C$ in direction $D$ [S1, unofficial]. Retrieve candidate triangles using the corresponding per-axis grid [S1, unofficial]. Perform a ray-triangle intersection test using the Möller–Trumbore algorithm [S1, unofficial]. Count the number of intersections $N_D$ [S1, unofficial]. If $N_D$ is odd, increment `inside_votes` [S1, unofficial]. **Majority Vote:** If `inside_votes` $\ge 3$, the voxel is considered inside [S1, unofficial]. Ties (3-3) count as inside [S1, unofficial].
4. **Output Generation:** If the voxel is inside, add `("i,j,k", colorId)` to the output map [S1, unofficial]. After processing all voxels, identify the seed voxel using the logic in Step 2 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solid_voxelize(mesh, voxel_mm, color_id):
    # 1. Build per-axis triangle grids [G]
    # Grids are maps from "ui,vi" string to list of triangle indices
    # Axis pairs: (Y,Z) for X-rays, (X,Z) for Y-rays, (X,Y) for Z-rays
    grids = build_per_axis_grids(mesh, voxel_mm)
    
    # 2. Determine voxel bounds and iterate
    bbox = mesh.bounding_box()
    min_i = floor(bbox.min_x / voxel_mm)
    max_i = ceil(bbox.max_x / voxel_mm)
    min_j = floor(bbox.min_y / voxel_mm)
    max_j = ceil(bbox.max_y / voxel_mm)
    min_k = floor(bbox.min_z / voxel_mm)
    max_k = ceil(bbox.max_z / voxel_mm)
    
    voxels = {}
    filled_voxels_by_k = {} # Map k -> list of (i, j)
    
    for k in range(min_k, max_k + 1):
        filled_in_k = []
        for j in range(min_j, max_j + 1):
            for i in range(min_i, max_i + 1):
                # Voxel center
                cx = i * voxel_mm
                cy = j * voxel_mm
                cz = k * voxel_mm
                
                inside_votes = 0
                
                # Check 6 directions
                directions = [
                    (1, 0, 0), (-1, 0, 0),
                    (0, 1, 0), (0, -1, 0),
                    (0, 0, 1), (0, 0, -1)
                ]
                
                for dx, dy, dz in directions:
                    # Determine which grid to use and compute cell indices
                    if dx != 0:
                        grid = grids['yz']
                        ui = cell_index(cy, bbox.min_y, voxel_mm)
                        vi = cell_index(cz, bbox.min_z, voxel_mm)
                    elif dy != 0:
                        grid = grids['xz']
                        ui = cell_index(cx, bbox.min_x, voxel_mm)
                        vi = cell_index(cz, bbox.min_z, voxel_mm)
                    else:
                        grid = grids['xy']
                        ui = cell_index(cx, bbox.min_x, voxel_mm)
                        vi = cell_index(cy, bbox.min_y, voxel_mm)
                        
                    tri_indices = grid.get(f"{ui},{vi}", [])
                    
                    # Ray-triangle intersections
                    intersection_count = 0
                    for tri_idx in tri_indices:
                        if ray_triangle_intersect(cx, cy, cz, dx, dy, dz, mesh.triangles[tri_idx]):
                            intersection_count += 1
                    
                    if intersection_count % 2 == 1:
                        inside_votes += 1
                
                # Majority vote
                if inside_votes >= 3:
                    voxels[f"{i},{j},{k}"] = color_id
                    filled_in_k.append((i, j))
        
        if filled_in_k:
            filled_voxels_by_k[k] = filled_in_k

    # 3. Seed Selection
    seed = pick_seed(filled_voxels_by_k)
    
    return voxels, seed

def cell_index(world_coord, origin, voxel_mm):
    return floor((world_coord - origin) / voxel_mm)

def ray_triangle_intersect(ox, oy, oz, dx, dy, dz, triangle):
    # Hand-rolled Möller–Trumbore [G]
    v0, v1, v2 = triangle
    e1 = (v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2])
    e2 = (v2[0]-v0[0], v2[1]-v0[1], v2[2]-v0[2])
    h = (dy*e2[2]-dz*e2[1], dz*e2[0]-dx*e2[2], dx*e2[1]-dy*e2[0])
    a = e1[0]*h[0] + e1[1]*h[1] + e1[2]*h[2]
    if a < 0.0000001 and a > -0.0000001:
        return False
    f = 1.0 / a
    s = (ox-v0[0], oy-v0[1], oz-v0[2])
    u = f * (s[0]*h[0] + s[1]*h[1] + s[2]*h[2])
    if u < 0.0 or u > 1.0:
        return False
    q = (dy*s[2]-dz*s[1], dz*s[0]-dx*s[2], dx*s[1]-dy*s[0])
    v = f * (e1[0]*q[0] + e1[1]*q[1] + e1[2]*q[2])
    if v < 0.0 or u + v > 1.0:
        return False
    t = f * (e2[0]*q[0] + e2[1]*q[1] + e2[2]*q[2])
    return t > 0.0000001

def pick_seed(filled_voxels_by_k):
    if not filled_voxels_by_k:
        return None
    min_k = min(filled_voxels_by_k.keys())
    min_k_cells = filled_voxels_by_k[min_k]
    if not min_k_cells:
        return None
    is_ = [c[0] for c in min_k_cells]
    js_ = [c[1] for c in min_k_cells]
    cx = (min(is_) + max(is_)) / 2
    cy = (min(js_) + max(js_)) / 2
    best = min_k_cells[0]
    best_d = float('inf')
    for c in min_k_cells:
        d = (c[0] - cx) ** 2 + (c[1] - cy) ** 2
        if d < best_d:
            best_d = d
            best = c
    return f"{best[0]},{best[1]},{min_k}"

def build_per_axis_grids(mesh, voxel_mm):
    # Standard uniform grid bucketing [G]
    grids = {'yz': {}, 'xz': {}, 'xy': {}}
    bbox = mesh.bounding_box()
    for tri_idx, tri in enumerate(mesh.triangles):
        # Project triangle to 2D planes and bucket into cells
        # (Simplified bucketing logic for demonstration; matches source intent)
        # In practice, compute bounding box of projected triangle and insert into cells
        pass
    return grids
```

**Parameters:**
- **Majority Threshold:** 3. A voxel is inside if $\ge 3$ directions yield odd parity [S1, unofficial].
- **Voxel Size:** Determined by `voxelMm` input [S1, unofficial].

**Pitfalls:**
- **Axis-aligned ray degeneracy:** Rays parallel to triangle edges or lying in triangle planes can cause numerical instability or missed intersections [S1, unofficial]. The source notes that the test sphere was placed off-origin to avoid this issue [S1, unofficial].
- **Watertight assumption:** The method assumes a watertight mesh [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(V \cdot T_{avg})$, where $V$ is the number of voxels and $T_{avg}$ is the average number of candidate triangles per ray [G]. The grid acceleration reduces $T_{avg}$ significantly [G].
- **Memory Complexity:** $O(T + G)$, where $T$ is the number of triangles and $G$ is the number of grid cells [G].
- **Performance:** ~75ms for a 40³-equivalent icosphere (20,480 faces, 32,764 voxels) [S1, unofficial].
- **Accuracy:** Sphere error rate 1.44% (r=20mm, 1280 faces) [S1, unofficial]. Box error rate 0.0047% [S1, unofficial]. Criterion was $\le 2\%$ [S1, unofficial].
- **Strengths:** Deterministic, robust for watertight meshes, fast due to grid acceleration and simple parity checks [S1, unofficial].
- **Limitations:** Assumes watertight mesh [S1, unofficial]. Axis-aligned rays may suffer from degeneracy if not handled carefully (e.g., offsetting rays or meshes) [S1, unofficial].

## Background
This method was implemented as part of issue 0010 in the Clothespin-Studio project, building on the design fixed by issue 0008 spike [S1, unofficial]. It was developed to convert normalized MeshData into a solid voxel target for the `growAssembly` process [S1, unofficial]. The implementation avoids dependencies on three.js for the domain layer and uses a hand-rolled Möller–Trumbore intersection test [S1, unofficial].

## Sources
[S1] feat: solid voxelization of normalized meshes (issue 0010), https://github.com/fumito-ito/Clothespin-Studio/pull/6, fetched 2026-10-26 (unofficial)
