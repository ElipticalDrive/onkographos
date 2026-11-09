# CUDA Voxelizer GPU Voxelization

Also called: CUDA Implementation Details, Implementation Details, GPU Voxelization Implementation, voxelize

## Overview
A GPU-accelerated method to convert polygon meshes into voxel grids using CUDA, supporting both surface voxelization (faces) and solid voxelization (filled volume) [S2, unofficial]. It differs from sibling methods by utilizing a bit-packed voxel table and conservative rasterization techniques adapted for parallel atomic operations, often employing persistent threads and radix sorting for sparse data management [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An indexed triangle set (vertex buffer and index buffer) [S1].
*   **Output:** A voxel grid (occupancy or attributes) and optionally a Morton-ordered sparse representation [S2, unofficial].

**Data Structures:**
1.  **`voxinfo`**: A struct passed to kernel functions containing:
    *   `bbox_mesh_cubed`: An axis-aligned bounding box (`AABox<float3>`) of the mesh, padded to form a cube [S2, unofficial].
    *   `gridsize`: A `uint3` defining the resolution of the cubical voxel grid [S2, unofficial].
    *   `num_faces`: The number of triangles in the input mesh [S2, unofficial].
2.  **`vtable`**: A bit-packed array representing the voxel grid.
    *   **Layout:** Each voxel is represented by 1 bit.
    *   **Size:** `ceil(gridsize.x * gridsize.y * gridsize.z / 32.0f) * 4` bytes [S2, unofficial].
3.  **`device_triangles`**: A flat array of floats on the GPU.
    *   **Layout:** 9 floats per triangle (3 vertices × 3 coordinates).
    *   **Size:** `sizeof(float) * 9 * num_faces` [S2, unofficial].

**Steps:**
1.  **Initialization:** Transform vertices to voxel space and initialize the voxel grid (`vtable`) with zeroes [S1].
2.  **Bounding Box Calculation:** Compute the bounding box of the mesh and pad it to form a cube (`bbox_mesh_cubed`) [S2, unofficial].
3.  **Memory Allocation:** Allocate CUDA-managed unified memory for `device_triangles` and `vtable` based on the calculated sizes [S2, unofficial].
4.  **Data Transfer:** Copy vertex data into the flat `device_triangles` array [S2, unofficial].
5.  **Voxelization Execution:** Call `voxelize()` for surface voxelization or `voxelize_solid()` for solid voxelization, passing `voxinfo`, `device_triangles`, `vtable`, and a `morton_code` flag [S2, unofficial].
    *   *Surface Voxelization:* Uses conservative rasterization to set bits for all voxels overlapping the triangle [S1].
    *   *Solid Voxelization:* Requires a watertight input mesh and fills the interior volume [S2, unofficial].
6.  **Post-Processing (Optional):** If a sparse representation is required, use radix sort (e.g., from CUDPP) to sort significant bits, followed by scan and compaction (e.g., using `chag::pp`) to generate the final sparse structure [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the core voxelization kernel logic
# Note: This is a simplified representation of the conservative rasterization and atomic update.

def cuda_voxelize_kernel(triangles, voxinfo, vtable):
    # 1. Identify the triangle to process using persistent thread pattern
    triangle_idx = get_triangle_index_persistent()
    if triangle_idx >= voxinfo.num_faces:
        return

    # 2. Fetch triangle vertices
    v0 = triangles[triangle_idx * 9 + 0:3]
    v1 = triangles[triangle_idx * 9 + 3:6]
    v2 = triangles[triangle_idx * 9 + 6:9]

    # 3. Compute Triangle Bounding Box in Grid Coordinates
    # World bbox min/max
    world_min = min(v0, v1, v2)
    world_max = max(v0, v1, v2)
    
    # Convert to grid indices
    # grid_max is voxinfo.gridsize - 1
    grid_min = clamp(floor(world_min / voxinfo.unit), 0, voxinfo.gridsize - 1)
    grid_max = clamp(floor(world_max / voxinfo.unit), 0, voxinfo.gridsize - 1)

    # 4. Conservative Rasterization / Voxel Coverage
    # Iterate over all voxels in the bounding box
    for z in range(grid_min.z, grid_max.z + 1):
        for y in range(grid_min.y, grid_max.y + 1):
            for x in range(grid_min.x, grid_max.x + 1):
                # Check if voxel (x,y,z) overlaps with triangle
                # This involves plane tests and edge tests as per Schwarz & Seidel
                if voxel_overlaps_triangle(x, y, z, v0, v1, v2, voxinfo.unit):
                    # 5. Atomic Update
                    # Calculate linear index in the bit-packed vtable
                    voxel_idx = x + y * voxinfo.gridsize.x + z * voxinfo.gridsize.x * voxinfo.gridsize.y
                    bit_idx = voxel_idx // 32
                    bit_offset = voxel_idx % 32
                    
                    # Atomic OR to set the bit
                    atomic_or(&vtable[bit_idx], 1 << bit_offset)

def voxel_overlaps_triangle(x, y, z, v0, v1, v2, unit):
    # Implementation of conservative rasterization tests
    # 1. Plane test: Check if voxel center is on the correct side of the triangle plane
    # 2. Edge tests: Check if voxel center is on the correct side of the triangle edges projected onto axes
    
    # Compute normal
    e0 = v1 - v0
    e1 = v2 - v1
    n = normalize(cross(e0, e1))
    
    # Voxel center in world space
    center = make_float3(x * unit.x, y * unit.y, z * unit.z) + unit / 2.0
    
    # Plane test
    d = dot(n, center - v0)
    if d < 0: # Assuming consistent winding, adjust sign as needed
        return False
        
    # Edge tests (simplified logic, actual implementation requires specific vector math)
    # Check against XY, YZ, XZ projections
    # ... (Detailed edge vector math from [S2, unofficial])
    
    return True
```

**Parameters:**
*   `gridsize`: Length of the cubical voxel grid. Default: 256 [S2, unofficial].
*   `solidVoxelization`: Boolean flag. If true, performs solid voxelization. Requires watertight mesh [S2, unofficial].
*   `morton_code`: Flag to enable Morton ordering in the output [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Atomic Contention:** Conservative voxelization requires setting a voxel by *all* overlapping triangles, not just the one covering its center. This causes conflicting atomic writes, which are expensive on older hardware (GT200) [S1].
*   **Mitigation:** Make a warp access triangles with a certain stride to increase the chance that concurrently executed atomics refer to different memory locations, as contiguous triangles are often spatially close [S1].
*   **Hardware Dependency:** Performance varies significantly by GPU architecture; GF100-class GPUs offer significant improvements in atomic performance compared to GT200 [S1].

## Subtopics

### [Parallel CPU/GPU Voxelization](7027-parallel-cpu-gpu-voxelization.md)
Accelerates tetrahedral mesh voxelization by distributing workload between CPU (OpenMP) and GPU (CUDA) streams. Chosen for high-density meshes where pure GPU overhead is prohibitive.

## Operating characteristics
*   **Time Complexity:** Dominated by atomic operations and sorting/compaction if sparse output is required.
*   **Memory Complexity:** $O(N)$ for the voxel table, where $N$ is the number of voxels. $O(F)$ for triangle storage, where $F$ is the number of faces.
*   **Performance:** On a GTX 1050 Ti, voxelizing the Stanford Bunny (70k triangles) takes 0.2 ms for a $64^3$ grid, 0.3 ms for $128^3$, 0.6 ms for $256^3$, and 1.8 ms for $512^3$ (excluding I/O) [S2, unofficial]. This is significantly faster than CPU implementations (e.g., 39.8 ms for $64^3$ on an i7 8750H) [S2, unofficial].
*   **Strengths:** High performance for real-time applications; supports both surface and solid voxelization; generates Morton-ordered sparse grids efficiently.
*   **Limitations:** Atomic contention can dominate runtime on older hardware; solid voxelization requires watertight meshes; GPU call overhead may penalize very small datasets [S2, unofficial].

## Background
The method is based on the work of M. Schwarz and H.P. Seidel in their 2010 paper *Fast Parallel Surface and Solid Voxelization on GPU's* [S1, S3]. The implementation details, including the Morton-encoded table, are also based on the author's 2013 HPG paper *Out-Of-Core construction of Sparse Voxel Octrees* and the `libmorton` library [S2, unofficial]. It is implemented as the `cuda_voxelizer` tool by Forceflow [S2, unofficial].

## Sources
[S1] Fast parallel surface and solid voxelization on GPUs, https://doi.org/10.1145/1882261.1866201, fetched 2026-11-03
[S2] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
[S3] PDF Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
