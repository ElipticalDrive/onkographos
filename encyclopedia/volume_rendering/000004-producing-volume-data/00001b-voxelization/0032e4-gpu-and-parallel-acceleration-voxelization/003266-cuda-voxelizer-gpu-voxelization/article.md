# CUDA Voxelizer GPU Voxelization

Also called: CUDA Voxelizer

## Overview
A CUDA-accelerated method for fast parallel surface and solid voxelization of polygon meshes into dense 3D grids [S1][S3, S5, unofficial]. It differs from CPU-based voxelization by leveraging GPU parallelism for triangle rasterization and atomic bit-setting, and from other GPU methods by its specific use of bit-packed voxel storage and conservative rasterization techniques adapted for atomic contention [S1][S3, unofficial]. It is designed for real-time per-frame voxelization [S3, S5, unofficial].

## How it works
**Inputs:** Indexed triangle set (vertex buffer, index buffer), target voxel grid resolution (default $256^3$) [S1][S3, S5, unofficial].
**Outputs:** Voxel grid (surface or solid), typically in `.binvox` or `.vox` format, or a Morton-ordered grid [S3, S5, unofficial].

**Data Structures:**
*   **Voxel Grid:** Bit-packed 3D array. Coordinate mapping: $(x, y, z) \rightarrow (x + y \cdot n) \cdot (n / 32) + z / 32$, where $z \% 32$ is the bit index within the integer [S2, unofficial]. Each `int` (32 bits) stores 32 voxels along the Z-axis [S2, unofficial].
*   **Voxel Info (`voxinfo`):** Struct containing `bbox_mesh_cubed` (AABox<float3>), `gridsize` (uint3), and `num_faces` (size_t) [S3, unofficial].
*   **Device Triangles:** Flat float array on GPU. Layout: 9 floats per triangle (3 vertices $\times$ 3 coordinates). Total size: $9 \cdot N \cdot \text{sizeof(float)}$ [S3, unofficial].
*   **Voxel Table (`vtable`):** Bit-packed array. Size: $\lceil (N_x \cdot N_y \cdot N_z) / 32.0 \rceil \cdot 4$ bytes [S3, unofficial].

**Parameters:**
*   `gridsize`: Length of the cubical voxel grid. Default: 256 [S3, unofficial].
*   `epsilon`: Floating-point constant for error checking. Value: 0.00001 [S2, unofficial].
*   `solidVoxelization`: Boolean flag. If true, performs solid voxelization (requires watertight mesh) [S3, unofficial].
*   `block_size`: CUDA block size. Value: 128 [S2, unofficial].

**Steps:**
1.  **Bounding Box Calculation:** Compute the bounding box of the mesh. Pad it to form a cube (`bbox_mesh_cubed`) [S3, unofficial].
2.  **Memory Allocation:** Allocate CUDA-managed unified memory for triangle data and voxel table [S3, unofficial].
3.  **Data Transfer:** Copy vertex data into a flat float array on the GPU [S3, unofficial].
4.  **Kernel Launch:** Launch `device::voxelize_kernel` with `block_num = (N + block_size - 1) / block_size` blocks and `block_size` threads [S2, unofficial]. Pass vertices, triangles, bounding boxes, delta, inverse delta, stride, and voxel data pointers [S2, unofficial].
5.  **Triangle Processing (Kernel):**
    *   Transform vertices to voxel space [S1].
    *   Compute triangle bounding box in grid coordinates: `t_bbox_grid.min = clamp(floor(t_bbox_world.min / info.unit), 0, grid_max)`, `t_bbox_grid.max = clamp(ceil(t_bbox_world.max / info.unit), 0, grid_max)` [S3, unofficial].
    *   For each voxel in the triangle's bounding box:
        *   Evaluate depth test: $(n \cdot p + d_1) \cdot (n \cdot p + d_2) > 0$ to skip voxels outside the triangle's depth range [S2, unofficial].
        *   Perform projection tests on YZ and ZX planes using edge normals and dot products [S3, unofficial].
        *   If inside, set the corresponding bit in the voxel table using atomic operations [S2, unofficial].
6.  **Post-Processing (Optional):** Use radix sort (e.g., CUDPP) and scan/compaction (e.g., chag::pp) for sparse voxel generation or Morton ordering [S1][S3, unofficial].
7.  **Result Retrieval:** Copy voxel data back to host or save to file [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cuda_voxelize(vertices, indices, grid_size=256, solid=False):
    # 1. Compute mesh bounding box and pad to cube
    bbox_min, bbox_max = compute_mesh_bbox(vertices)
    size = max(bbox_max - bbox_min)
    center = (bbox_min + bbox_max) / 2.0
    bbox_cubed = AABox(center - size/2, center + size/2)
    
    # 2. Prepare voxel info
    voxinfo = {
        'bbox_mesh_cubed': bbox_cubed,
        'gridsize': (grid_size, grid_size, grid_size),
        'num_faces': len(indices) // 3
    }
    
    # 3. Allocate GPU memory
    d_vertices = allocate_unified_memory(9 * voxinfo['num_faces'] * sizeof(float))
    d_vtable = allocate_unified_memory(ceil(grid_size**3 / 32.0) * 4)
    
    # 4. Copy data to GPU
    copy_to_gpu(d_vertices, vertices, indices)
    clear_memory(d_vtable)
    
    # 5. Launch Kernel
    block_size = 128
    block_num = (voxinfo['num_faces'] + block_size - 1) // block_size
    
    # Kernel arguments: d_vertices, indices, N, bbox_cubed, grid_size, d_vtable, solid
    launch_kernel(device_voxelize_kernel, block_num, block_size, 
                  d_vertices, indices, voxinfo['num_faces'], 
                  bbox_cubed, grid_size, d_vtable, solid)
    
    # 6. Synchronize and retrieve
    synchronize()
    vtable_host = copy_to_host(d_vtable)
    return vtable_host

def device_voxelize_kernel(d_vertices, indices, N, bbox, grid_size, d_vtable, solid):
    tid = blockIdx.x * blockDim.x + threadIdx.x
    if tid >= N: return
    
    # Load triangle vertices
    v0 = load_triangle_vertex(d_vertices, tid, 0)
    v1 = load_triangle_vertex(d_vertices, tid, 1)
    v2 = load_triangle_vertex(d_vertices, tid, 2)
    
    # Compute normal and edge vectors
    e0 = v1 - v0
    e1 = v2 - v1
    e2 = v0 - v2
    n = normalize(cross(e0, e1))
    
    # Compute triangle bounding box in grid coordinates
    t_bbox_world = AABox(min(v0, v1, v2), max(v0, v1, v2))
    unit = bbox.size / grid_size
    t_bbox_grid_min = clamp(floor(t_bbox_world.min / unit), 0, grid_size-1)
    t_bbox_grid_max = clamp(ceil(t_bbox_world.max / unit), 0, grid_size-1)
    
    # Precompute plane test properties
    c = unit * 0.5 # Center offset for plane test
    d1 = dot(n, c - v0)
    d2 = dot(n, (bbox.max - c) - v0) # Simplified for demo
    
    # Iterate over voxels in bounding box
    for z in range(t_bbox_grid_min.z, t_bbox_grid_max.z + 1):
        for y in range(t_bbox_grid_min.y, t_bbox_grid_max.y + 1):
            for x in range(t_bbox_grid_min.x, t_bbox_grid_max.x + 1):
                p = (x + 0.5) * unit + bbox.min
                # Depth test
                if (dot(n, p) + d1) * (dot(n, p) + d2) <= 0:
                    continue
                # Projection tests (simplified)
                if is_inside_projection(p, v0, v1, v2, n):
                    # Set bit in vtable
                    idx = (x + y * grid_size) * (grid_size // 32) + z // 32
                    bit = z % 32
                    atomic_or(&d_vtable[idx], 1 << bit)

def is_inside_projection(p, v0, v1, v2, n):
    # Check barycentric coordinates or edge functions
    # Simplified: check against all three edges
    for edge in [(v0, v1), (v1, v2), (v2, v0)]:
        if dot(cross(edge[1] - edge[0], p - edge[0]), n) < 0:
            return False
    return True
```

## Subtopics

### [Parallel CPU/GPU Voxelization](7027-parallel-cpu-gpu-voxelization.md)
Hybrid approach for tetrahedral meshes, distributing workload between CPU (OpenMP) and GPU (CUDA). Choose for high-density tetrahedral meshes where CPU-GPU transfer overhead is justified by throughput gains.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and voxels in the bounding box [S1].
*   **Memory Complexity:** $O(N_x \cdot N_y \cdot N_z / 32)$ bits for the voxel grid [S3, unofficial].
*   **Performance:** On a GTX 1060, voxelizing the Stanford Bunny (70k triangles) takes 4.2 ms ($128^3$), 6.2 ms ($256^3$), 13.4 ms ($512^3$), and 38.6 ms ($1024^3$), including memory transfers [S5, unofficial]. On a GTX 1050 Ti, times are 0.2 ms ($64^3$), 0.3 ms ($128^3$), 0.6 ms ($256^3$), 1.8 ms ($512^3$) [S3, unofficial].
*   **Strengths:** High performance for real-time applications; conservative voxelization ensures no gaps [S1][S3, unofficial].
*   **Limitations:** Atomic contention can dominate runtime on older hardware (GT200); requires watertight mesh for solid voxelization [S1][S3, unofficial].
*   **Hardware:** Requires CUDA-compatible GPU with Compute Capability 2.0 or higher [S3, S5, unofficial].

## Background
Developed by Jeroen Baert, based on the 2010 paper "Fast Parallel Surface and Solid Voxelization on GPU's" by M. Schwarz and H.P. Seidel [S3, S5, unofficial]. The morton-encoded table implementation draws from Baert's 2013 HPG paper "Out-Of-Core construction of Sparse Voxel Octrees" and the libmorton library [S3, S5, unofficial].

## Sources
[S1] Fast parallel surface and solid voxelization on GPUs, https://doi.org/10.1145/1882261.1866201, fetched 2026-11-03
[S2] hpsoar/voxelizer, https://github.com/hpsoar/voxelizer, fetched 2026-10-26 (unofficial)
[S3] GitHub - Forceflow/cuda_voxelizer: CUDA Voxelizer to convert polygon ..., https://github.com/Forceflow/cuda_voxelizer, fetched 2026-10-27 (unofficial)
[S4] PDF Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
[S5] CUDA Voxelizer v0.1 - Jeroen Baert's Blog, https://www.forceflow.be/2016/10/02/cuda-voxelizer-v0-1/, fetched 2026-10-30 (unofficial)
