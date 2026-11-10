# CPU Mesh Voxelization

Also called: Triangle Rasterization to Grid, Triangle-to-Voxel Rasterization, Triangle rasterization into voxel grid, 3D Triangle Rasterization into Voxel Grid, AABB voxel traversal with plane/edge tests

## Overview
CPU Mesh Voxelization converts a triangle mesh into a 3D grid of voxels by iterating over the bounding box of each triangle and testing each voxel center against the triangle's plane and edge constraints [S2, S3, S5, unofficial]. It differs from its parent [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by outputting a binary occupancy grid or bit-packed table rather than generating a cube mesh per occupied voxel [S3, S5, unofficial]. It differs from [3D Grid Rasterization](903-3d-grid-rasterization.md) by using exact geometric overlap tests (SAT or plane/projection) rather than simple grid cell intersection, and from [Voxelization via Triangle Scattering](3779-voxelization-via-triangle-scattering.md) by iterating cells against triangles rather than scattering triangles into cells [S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
* Input: A triangle mesh defined by vertices $V = \{v_0, v_1, \dots\}$ and face indices [S2, S3, unofficial].
* Input: Grid resolution $res$ (integer, typically 32, 64, or 128) [S3, unofficial].
* Input: Global mesh bounding box $B_{min}$ and $B_{max}$ used to compute a scale factor $s = \frac{res}{\max(B_{max} - B_{min})}$ [S1, S2, unofficial].
* Output: A voxel grid. Common layouts include a 1D array of integers/bytes `grid->data` of size $res^3$ where values are 0 (empty) or 1 (filled) [S3, unofficial], or a bit-packed `unsigned int*` array where each bit represents a voxel [S5, unofficial]. Some implementations output a `Vec<u128>` packing `rgba32ui` GPU texels, where each texel represents a $4 \times 4 \times 8$ voxel block [S2, unofficial].

**Data Structures:**
* **Voxel Grid:** A contiguous memory block [S3, S5, unofficial]. For a linear array, index mapping is $idx = x + y \cdot res + z \cdot res^2$ [S3, S5, unofficial]. For bit-packed storage, bit index $i$ is set in `voxel_table[i / 32]` at position `31 - (i % 32)` [S5, unofficial].
* **Bounding Box:** Axis-Aligned Bounding Box (AABB) of the triangle in grid coordinates, defined by $[X_0, X_1] \times [Y_0, Y_1] \times [Z_0, Z_1]$ [S4, unofficial].
* **Left/Right Buffers (Optional):** For scanline-style rasterization, 1D arrays `lx`, `ly`, `lz` and `rx`, `ry`, `rz` of size $vsz = \max(vxs, vys, vzs)$ [S4, unofficial].

**Steps:**

1. **Preprocessing:**
    * Parse the mesh and extract vertices and triangle indices [S2, unofficial].
    * Compute the global AABB of the mesh: $B_{min} = \min_i(v_i)$, $B_{max} = \max_i(v_i)$ [S2, S5, unofficial].
    * Normalize the mesh to fit the grid. Calculate scale $s = \frac{res}{\max(B_{max} - B_{min})}$ [S1, S2, unofficial].
    * Transform vertices to grid space: $p' = (v - B_{min}) \cdot s$ [S1, unofficial].
    * Clamp transformed vertices to $[0, res-1]$ (or `resolution.next_down()`) to prevent out-of-bounds access [S1, S2, unofficial].
    * Skip degenerate triangles (zero area) and non-triangle faces (quads, n-gons) [S1, S5, unofficial].

2. **Per-Triangle Processing:**
    For each triangle with vertices $v_0, v_1, v_2$:
    * Compute the AABB in grid coordinates:
      $$X_0 = \lfloor \min(v_{0x}, v_{1x}, v_{2x}) \rfloor, \quad X_1 = \lceil \max(v_{0x}, v_{1x}, v_{2x}) \rceil$$
      (Similarly for $Y$ and $Z$) [S4, S5, unofficial].
    * Clamp $[X_0, X_1] \times [Y_0, Y_1] \times [Z_0, Z_1]$ to $[0, \text{gridsize}-1]$ [S5, unofficial].
    * **Method A: SAT / Plane-Edge Test (Exact Overlap)**
        * Compute triangle normal $n = \text{normalize}((v_1 - v_0) \times (v_2 - v_0))$ [S3, S5, unofficial].
        * Compute plane bounds for the voxel slab [S2, unofficial]:
          $$lower = n \cdot v_0 - \frac{1}{2} \left( \sum |n_i| + n \cdot \text{signum}(n) \right), \quad upper = n \cdot v_0 - \frac{1}{2} \left( \sum |n_i| - n \cdot \text{signum}(n) \right)$$
        * Iterate voxel coordinates $vx \in [X_0, X_1]$, $vy \in [Y_0, Y_1]$, $vz \in [Z_0, Z_1]$ [S3, unofficial].
        * Compute voxel center:
          $$c_x = \frac{vx + 0.5}{res}, \quad c_y = \frac{vy + 0.5}{res}, \quad c_z = \frac{vz + 0.5}{res}$$
          (If using grid-space coordinates directly, $c = (vx, vy, vz)$) [S3, S5, unofficial].
        * **Plane Test:** Compute $d = n \cdot c$. If $d < lower$ or $d > upper$, skip voxel [S2, unofficial]. Alternatively, compute $d_1 = n \cdot (c - v_0)$ and $d_2 = n \cdot ((c - v_0) - \text{half\_voxel})$, and skip if $(d + d_1)(d + d_2) > 0.0$ [S5, unofficial].
        * **Edge Tests:** Project triangle onto XY, YZ, and ZX planes [S2, S5, unofficial]. Compute edge normals for each projection [S2, S5, unofficial]. If the voxel center fails any of the 9 edge tests (dot product with edge normal < distance threshold), skip voxel [S5, unofficial]. Alternatively, use Separating Axis Theorem (SAT) precomputed vectors [S3, unofficial].
        * If all tests pass, mark voxel: `grid[idx] = 1` or `setBit(voxel_table, idx)` [S3, S5, unofficial].
    * **Method B: Scanline Rasterization (Left/Right Buffers)**
        * Select major axis: the dimension with the largest AABB span ($dx, dy, dz$) [S4, unofficial].
        * Rasterize the three triangle edges into left/right buffers using a 3D DDA line algorithm [S4, unofficial].
            * For an edge, determine if it goes to left or right buffer based on the sign of the major coordinate difference [S4, unofficial].
            * Store the other two coordinates in the buffer indexed by the major coordinate [S4, unofficial].
        * Iterate through the major axis range [S4, unofficial]. For each index, retrieve left $(y_0, z_0)$ and right $(y_1, z_1)$ coordinates from buffers [S4, unofficial].
        * Fill the voxel line between left and right endpoints using a 3D DDA line algorithm [S4, unofficial].
        * If the triangle is "diagonal" (normal dot major axis < 0.9), also add the voxel directly above the rasterized position if within bounds [S1, unofficial].

3. **Post-processing (Optional):**
    * Deduplicate voxel list by sorting lexicographically by $(x, y, z)$ and removing consecutive duplicates [S1, unofficial].
    * If using bit-packed output, finalize the `voxel_table` [S5, unofficial].

**Pseudocode (Method A: SAT/Plane Test):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_mesh_voxelization(mesh_vertices, mesh_faces, resolution):
    # 1. Preprocessing
    min_bound = [min(v[i] for v in mesh_vertices) for i in range(3)]
    max_bound = [max(v[i] for v in mesh_vertices) for i in range(3)]
    scale = resolution / max(max_bound[i] - min_bound[i] for i in range(3))

    grid_vertices = []
    for v in mesh_vertices:
        p = [(v[i] - min_bound[i]) * scale for i in range(3)]
        p = [max(0.0, min(resolution - 1.0, coord)) for coord in p]
        grid_vertices.append(p)

    grid = [0] * (resolution ** 3)

    # 2. Process Triangles
    for face in mesh_faces:
        v0, v1, v2 = grid_vertices[face[0]], grid_vertices[face[1]], grid_vertices[face[2]]
        if v0 == v1 or v1 == v2 or v0 == v2:
            continue

        X0, X1 = int(min(v0[0], v1[0], v2[0])), int(max(v0[0], v1[0], v2[0]))
        Y0, Y1 = int(min(v0[1], v1[1], v2[1])), int(max(v0[1], v1[1], v2[1]))
        Z0, Z1 = int(min(v0[2], v1[2], v2[2])), int(max(v0[2], v1[2], v2[2]))
        X0, X1 = max(0, X0), min(resolution - 1, X1)
        Y0, Y1 = max(0, Y0), min(resolution - 1, Y1)
        Z0, Z1 = max(0, Z0), min(resolution - 1, Z1)

        e0 = [v1[i] - v0[i] for i in range(3)]
        e1 = [v2[i] - v0[i] for i in range(3)]
        n = [e0[1]*e1[2] - e0[2]*e1[1], e0[2]*e1[0] - e0[0]*e1[2], e0[0]*e1[1] - e0[1]*e1[0]]
        n_len = (n[0]**2 + n[1]**2 + n[2]**2)**0.5
        if n_len == 0:
            continue
        n = [c / n_len for c in n]

        half = 0.5 / resolution
        signum_n = [1.0 if c > 0 else -1.0 for c in n]
        sum_abs_n = sum(abs(c) for c in n)
        dot_signum_n = sum(n[i] * signum_n[i] for i in range(3))
        lower = sum(n[i] * v0[i] for i in range(3)) - 0.5 * (sum_abs_n + dot_signum_n)
        upper = sum(n[i] * v0[i] for i in range(3)) - 0.5 * (sum_abs_n - dot_signum_n)

        for vx in range(X0, X1 + 1):
            for vy in range(Y0, Y1 + 1):
                for vz in range(Z0, Z1 + 1):
                    cx, cy, cz = vx + 0.5, vy + 0.5, vz + 0.5
                    dist = n[0]*cx + n[1]*cy + n[2]*cz

                    if dist < lower or dist > upper:
                        continue

                    # Edge tests (simplified SAT projection check)
                    edges = [(v1, v0), (v2, v1), (v0, v2)]
                    outside = False
                    for axis in range(3):
                        p1, p2 = axis, (axis + 1) % 3
                        for e_start, e_end in edges:
                            edge = [e_end[i] - e_start[i] for i in range(3)]
                            n_edge = [-edge[p2], edge[p1]]
                            p_proj = [v0[p1], v0[p2]]
                            d_edge = n_edge[0]*p_proj[0] + n_edge[1]*p_proj[2] - max(n_edge[0], 0) - max(n_edge[1], 0)
                            # Check voxel center projection against edge slab
                            v_proj = [cx, cy, cz]
                            if n_edge[0]*v_proj[p1] + n_edge[1]*v_proj[p2] < d_edge:
                                outside = True
                                break
                        if outside:
                            break
                    if outside:
                        continue

                    idx = vx + vy * resolution + vz * resolution * resolution
                    grid[idx] = 1

    return grid
```

## Subtopics

### [Voxelization via Triangle Scattering](3779-voxelization-via-triangle-scattering.md)
Voxelization via Triangle Scattering converts a triangle mesh into a 3D grid of voxels to identify the air region for acoustic simulation. It rasterizes surface triangles into the grid using an exact overlap test and isolates the connected air volume via flood fill, avoiding acceleration structures like BVHs. It differs from its parent [CPU Mesh Voxelization](3137-cpu-mesh-voxelization.md) by scattering triangles into cells rather than iterating cells against triangles, and from [Inner Loop Optimizations for CPU Mesh Voxelization](4048-inner-loop-optimizations-for-cpu-mesh-voxelization.md) by prioritizing exact geometric overlap over conservative bounding box approximations.

### [Inner Loop Optimizations for CPU Mesh Voxelization](4048-inner-loop-optimizations-for-cpu-mesh-voxelization.md)
This method applies structural and algorithmic optimizations to the core inner loop of `MeshProcessor::voxelize` to reduce constant overhead when processing millions of grid cells. It distinguishes itself from sibling methods by focusing on cache-friendly memory allocation, loop-invariant hoisting, and replacing binary searches with rolling indices to achieve amortized $O(1)$ lookups, rather than changing the fundamental rasterization or scattering approach.

## Operating characteristics
* **Time Complexity:** $O(T \cdot V_{bb})$, where $T$ is the number of triangles and $V_{bb}$ is the average number of voxels in each triangle's bounding box [S2, unofficial]. Each voxel test performs a constant number of dot products (typically 10 total) [S2, unofficial].
* **Memory Complexity:** $O(res^3)$ for the voxel grid [S3, unofficial]. Buffer size for scanline method is $O(\max(res_x, res_y, res_z))$ [S4, unofficial].
* **Performance:** Performance scales linearly with resolution cubed and triangle count. Optimizations like early-exit traversal (breaking loops if a voxel fails intersection after success) can improve average case [S2, unofficial].
* **Strengths:** Exact geometric overlap (no over-rasterization if SAT is used), simple implementation, no GPU required [S2, S3, unofficial].
* **Limitations:** Slow for high resolutions due to $O(res^3)$ grid access per triangle bounding box. Bounding box traversal can be inefficient for small triangles in large grids [S2, unofficial].
* **When to use:** Not given in the sources.
* **When not to use:** Not given in the sources.

## Background
CPU Mesh Voxelization is a foundational technique in volume rendering and 3D modeling, originating from the need to convert polygonal meshes into volumetric data for ray tracing, finite element analysis, and manufacturing simulation [S2, S4, unofficial]. The method is described in various open-source projects and Q&A forums, such as the "Parallel 3D Voxelization Project" which benchmarks sequential, OpenMP, pthreads, MPI, and CUDA implementations [S3, unofficial], and Stack Overflow discussions on 3D triangle rasterization [S4, unofficial]. It is often used as a baseline for comparing GPU-based voxelization techniques [S3, unofficial].

## Sources
[S1] rayferric/voxel-engine, https://github.com/rayferric/voxel-engine, fetched 2026-10-24 (unofficial)
[S2] GitHub - DeadlockCode/voxel_ray_traversal, https://github.com/DeadlockCode/voxel_ray_traversal, fetched 2026-10-25 (unofficial)
[S3] simenzzz/Parallel-Project, https://github.com/simenzzz/Parallel-Project, fetched 2026-10-26 (unofficial)
[S4] 3D triangle rasterization into voxel grid, https://stackoverflow.com/questions/74251620/3d-triangle-rasterization-into-voxel-grid, fetched 2026-10-27 (unofficial)
[S5] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
