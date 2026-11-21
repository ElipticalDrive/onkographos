# CUDA Voxelization

Also called: CUDA Voxel Generation Batch, CUDA Voxel Rasterization

## Overview
A CUDA GPU implementation of the voxelization algorithm that launches a 3D grid of thread blocks to process voxels in parallel [S1, unofficial]. It differs from its parent [Voxel Rasterization](3204-voxel-rasterization.md) by utilizing a specific 3D block launch configuration and precomputed face bounds to optimize device-side computation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Host-side mesh data (vertices, faces), bounding box parameters, and resolution `res`.
- **Output:** A 3D voxel grid (binary occupancy) on the host.

**Data Structures:**
- `verts`: Device pointer to `float3` array, size `mesh.num_vertices` [S1, unofficial].
- `faces`: Device pointer to `int3` array, size `mesh.num_faces` [S1, unofficial].
- `face_bounds`: Device pointer to `VoxelBounds` array, size `mesh.num_faces` [S1, unofficial]. Precomputed on the host to reduce repeated per-face min/max work in device code [S1, unofficial].
- `grid`: Device pointer to `unsigned int` array, size $res^3$ [S1, unofficial].

**Parameters:**
- `res`: Integer resolution of the cubic grid [S1, unofficial].
- `b`: Block size parameter (default `DEFAULT_BLOCK_SIZE`, tested values 64, 128, 256) [S1, unofficial].
- `blockDim3`: 3D block dimensions `dim3 blockDim3(8, 8, 8)`, resulting in 512 threads per block [S1, unofficial].
- `gridDim3`: 3D grid dimensions `dim3 gridDim3((opt.res + 7) / 8, (opt.res + 7) / 8, (opt.res + 7) / 8)`, calculating ceiling division by 8 for each dimension [S1, unofficial].

**Steps:**
1. Allocate device memory for `verts_d`, `faces_d`, `bounds_d`, and `grid_d` using `cudaMalloc` [S1, unofficial].
2. Copy host mesh data to device using `cudaMemcpy` with `cudaMemcpyHostToDevice` [S1, unofficial].
3. Zero the device grid `grid_d` using `cudaMemset` [S1, unofficial].
4. Launch the kernel: `voxelize_kernel<<<gridDim3, blockDim3>>>(verts_d, faces_d, bounds_d, mesh.num_faces, grid_d, opt.res)` [S1, unofficial].
5. Measure kernel execution time using CUDA events: record start event, launch kernel, record stop event, synchronize stop event, and get elapsed time in milliseconds [S1, unofficial].
6. Copy result grid from device to host: `cudaMemcpy(grid_h32, grid_d, (size_t)total_voxels * sizeof(unsigned int), cudaMemcpyDeviceToHost)` [S1, unofficial].
7. Convert to binary: `grid.data[i] = grid_h32[i] != 0U ? 1U : 0U` [S1, unofficial].

**Pitfalls:**
- Memory allocation failure: If any `cudaMalloc` or `malloc` fails, the implementation must free all allocated resources (device memory, host memory, CUDA events) and return error code 1 [S1, unofficial].

## Subtopics

### [GPU Voxelization with Shared Memory Cache and Global Memory Fallback](3567-gpu-voxelization-with-shared-memory-cache-and-global-memory-.md)
A CUDA kernel that maps 3D LiDAR points to a voxel grid by aggregating attributes into a block-level shared memory cache, falling back to atomic operations on global memory for cache collisions. It differs from standard global-memory-only voxelization by reducing global memory contention through a small, fixed-size cache per block, and from pure shared-memory approaches by handling hash collisions gracefully.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Benchmarked on 9 input meshes from `data/` at resolutions 32, 64, and 128 [S1, unofficial]. Timing records kernel time only [S1, unofficial].
- **Scaling:** Tested with CUDA block sizes 64, 128, and 256 [S1, unofficial].
- **Strengths:** Parallel processing of voxels; precomputed face bounds reduce device-side computation [S1, unofficial].
- **Limitations:** Requires host-side precomputation of face bounds [S1, unofficial].
- **When to use:** When GPU acceleration is available and mesh voxelization is required [S1, unofficial].
- **When not:** When CPU-only environments are constrained or mesh complexity is low enough that overhead outweighs benefits (not explicitly stated, but implied by benchmark matrix) [S1, unofficial].

## Background
Originated in the project `simenzzz/Parallel-Project` [S1, unofficial]. The implementation is part of a hybrid workflow comparing CPU backends (Sequential, OpenMP, Pthreads, MPI) and GPU backends (CUDA) [S1, unofficial]. It is designed to run on Google Colab with a T4 GPU [S1, unofficial].

## Sources
[S1] simenzzz/Parallel-Project, https://github.com/simenzzz/Parallel-Project, fetched 2026-10-26 (unofficial)
