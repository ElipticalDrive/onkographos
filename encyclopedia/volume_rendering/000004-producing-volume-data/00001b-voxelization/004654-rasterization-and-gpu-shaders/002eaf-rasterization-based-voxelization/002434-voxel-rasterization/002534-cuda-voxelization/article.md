# CUDA Voxelization

Also called: CUDA Voxel Generation Batch, CUDA Voxel Rasterization

## Overview
A CUDA GPU implementation of the voxelization algorithm that launches a 3D grid of thread blocks to process voxels in parallel [S1, unofficial]. It differs from its parent [Voxel Rasterization](3204-voxel-rasterization.md) by utilizing a specific 3D block launch configuration and precomputed face bounds to optimize device-side computation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Host-side mesh data (vertices, faces), bounding box parameters, and resolution `res` [S1, unofficial].
- **Output:** A 3D binary occupancy voxel grid on the host [S1, unofficial].

**Data Structures:**
- `verts`: Device pointer to `float3` array, size `mesh.num_vertices` [S1, unofficial].
- `faces`: Device pointer to `int3` array, size `mesh.num_faces` [S1, unofficial].
- `face_bounds`: Device pointer to `VoxelBounds` array, size `mesh.num_faces` [S1, unofficial]. Precomputed on the host to reduce repeated per-face min/max work in device code [S1, unofficial].
- `grid`: Device pointer to `unsigned int` array, size $res^3$ [S1, unofficial].

**Numbered Steps:**
1. Allocate device memory for `verts_d`, `faces_d`, `bounds_d`, and `grid_d` using `cudaMalloc` [S1, unofficial].
2. Copy host mesh data to device using `cudaMemcpy` with `cudaMemcpyHostToDevice` [S1, unofficial].
3. Zero the device grid `grid_d` using `cudaMemset` [S1, unofficial].
4. Launch the kernel: `voxelize_kernel<<<gridDim3, blockDim3>>>(verts_d, faces_d, bounds_d, mesh.num_faces, grid_d, opt.res)` [S1, unofficial].
5. Measure kernel execution time using CUDA events: record start event, launch kernel, record stop event, synchronize stop event, and get elapsed time in milliseconds [S1, unofficial].
6. Copy result grid from device to host: `cudaMemcpy(grid_h32, grid_d, (size_t)total_voxels * sizeof(unsigned int), cudaMemcpyDeviceToHost)` [S1, unofficial].
7. Convert to binary: `grid.data[i] = grid_h32[i] != 0U ? 1U : 0U` [S1, unofficial].

**Formulas:**
- Grid dimension calculation: `gridDim3 = ((opt.res + 7) // 8, (opt.res + 7) // 8, (opt.res + 7) // 8)` [S1, unofficial].
- Total voxels: `total_voxels = opt.res ** 3` [S1, unofficial].

**Parameters:**
- `res`: Integer resolution of the cubic grid [S1, unofficial].
- `b`: Command-line argument `-b` controlling block size, default `DEFAULT_BLOCK_SIZE`, tested values 64, 128, 256 [S1, unofficial].
- `blockDim3`: 3D block dimensions `dim3(8, 8, 8)`, resulting in 512 threads per block [S1, unofficial].
- `gridDim3`: 3D grid dimensions calculated via ceiling division by 8 for each dimension [S1, unofficial].

**Edge Cases and Pitfalls:**
- Memory allocation failure: If any `cudaMalloc` or `malloc` fails, the implementation must free all allocated resources (device memory, host memory, CUDA events) and return error code 1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cuda_voxelize(mesh, opt):
    # 1. Allocate device memory
    verts_d = allocate_device_array(mesh.num_vertices, dtype=float3)
    faces_d = allocate_device_array(mesh.num_faces, dtype=int3)
    bounds_d = allocate_device_array(mesh.num_faces, dtype=VoxelBounds)
    grid_d = allocate_device_array(opt.res ** 3, dtype=unsigned_int)

    # 2. Copy host data to device
    copy_to_device(verts_d, mesh.verts)
    copy_to_device(faces_d, mesh.faces)
    copy_to_device(bounds_d, mesh.face_bounds)

    # 3. Zero device grid
    zero_memory(grid_d, size=opt.res ** 3 * sizeof(unsigned_int))

    # 4. Configure launch dimensions
    block_dim = dim3(8, 8, 8)
    grid_dim = dim3(
        (opt.res + 7) // 8,
        (opt.res + 7) // 8,
        (opt.res + 7) // 8
    )

    # 5. Launch kernel and measure time
    start_event = create_event()
    stop_event = create_event()
    record_event(start_event)
    # Kernel body not provided in sources
    launch_kernel(voxelize_kernel, grid_dim, block_dim, verts_d, faces_d, bounds_d, mesh.num_faces, grid_d, opt.res)
    record_event(stop_event)
    synchronize_event(stop_event)
    elapsed_ms = get_elapsed_time(start_event, stop_event)

    # 6. Copy result back to host
    total_voxels = opt.res ** 3
    grid_h32 = allocate_host_array(total_voxels, dtype=unsigned_int)
    copy_to_host(grid_h32, grid_d, size=total_voxels * sizeof(unsigned_int))

    # 7. Convert to binary
    grid = [0] * total_voxels
    for i in range(total_voxels):
        grid[i] = 1 if grid_h32[i] != 0 else 0

    # On failure, free all resources and return 1
    return grid, elapsed_ms
```

## Subtopics

### [GPU Voxelization with Shared Memory Cache and Global Memory Fallback](3567-gpu-voxelization-with-shared-memory-cache-and-global-memory-.md)
A CUDA kernel that maps 3D LiDAR points to a voxel grid by aggregating attributes into a block-level shared memory cache, falling back to atomic operations on global memory for cache collisions. It differs from standard global-memory-only voxelization by reducing global memory contention through a small, fixed-size cache per block, and from pure shared-memory approaches by handling hash collisions gracefully.

## Operating characteristics
- **Time Complexity:** Not given in the sources. [S1, unofficial]
- **Memory Complexity:** Not given in the sources. [S1, unofficial]
- **Measured Performance:** Benchmarked on 9 input meshes from `data/` at resolutions 32, 64, and 128 [S1, unofficial]. Timing records kernel time only [S1, unofficial].
- **Scaling:** Tested with CUDA block sizes 64, 128, and 256 [S1, unofficial].
- **Strengths:** Parallel processing of voxels; precomputed face bounds reduce device-side computation [S1, unofficial].
- **Limitations:** Requires host-side precomputation of face bounds [S1, unofficial].
- **When to use:** Not given in the sources. [S1, unofficial]
- **When not:** Not given in the sources. [S1, unofficial]

## Background
Originated in the project `simenzzz/Parallel-Project` [S1, unofficial]. The implementation is part of a hybrid workflow comparing CPU backends (Sequential, OpenMP, Pthreads, MPI) and GPU backends (CUDA) [S1, unofficial]. It is designed to run on Google Colab with a T4 GPU [S1, unofficial].

## Sources
[S1] simenzzz/Parallel-Project, https://github.com/simenzzz/Parallel-Project, fetched 2026-10-26 (unofficial)
