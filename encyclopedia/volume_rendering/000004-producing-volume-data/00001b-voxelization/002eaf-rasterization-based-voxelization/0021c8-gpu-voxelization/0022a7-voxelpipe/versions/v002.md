# VoxelPipe
## Overview
VoxelPipe is a highly flexible, programmable software pipeline for triangle voxelization implemented in CUDA [S1]. It solves the problem of converting triangle meshes into voxel grids with support for fully conservative and thin voxelizations, arbitrary render targets (boolean, floating-point, vector), and user-defined shaders [S1]. It distinguishes itself from prior binary voxelizers by supporting arbitrary framebuffer types and programmability via vertex and fragment shaders, tackling load balancing via a tile-based algorithm instead of assigning individual triangles to single threads with global memory atomics, and optimizing the triangle/box overlap test [S1].

## How it works
VoxelPipe implements two distinct pipelines depending on the output mode: a sort-middle tile-based rasterizer for blending-based (standard) voxelization, and a sort-last feedforward pipeline for A-buffer (bucketing) generation [S1].

### Inputs and Outputs
*   **Inputs:** Triangle mesh (vertices, indices), voxel grid resolution, voxelization type (thin or conservative), render target type (boolean, float, vector), and optional user-defined vertex/fragment shaders [S1].
*   **Outputs:** A 3D voxel grid. In standard mode, each voxel contains a blended value (e.g., max-blended normal) [S1]. In A-buffer mode, each voxel contains a list of all fragments belonging to it [S1].

### Data Structures
The pipeline relies on efficient sorting primitives to avoid inter-thread communication queues [S1].
*   **Coarse Tiles:** The 3D grid is divided into coarse tiles [S1]. The tile size is limited by shared memory constraints; for floating-point buffers, the maximum tile size is $16^3$ [S1].
*   **Triangle/Tile Pairs:** Lists of triangles overlapping each coarse tile are generated during the coarse raster stage [S1].
*   **Fragments:** Generated in the fine raster stage. In A-buffer mode, fragments are sorted into voxels [S1].

### Blending-Based Rasterization (Standard Mode)
This mode uses a sort-middle pipeline divided into two stages: coarse raster and fine raster [S1].

1.  **Coarse Raster:**
    *   The grid is divided into coarse tiles [S1].
    *   Triangle setup is performed [S1].
    *   An overlap test determines which triangles intersect which coarse tiles [S1].
    *   Triangles overlapping each tile are emitted, creating a list of triangle/tile pairs [S1].
    *   Triangles are sorted into tiles using efficient sorting primitives [S1].
2.  **Fine Raster:**
    *   Each coarse tile is assigned to a single GPU core (thread block) [S1].
    *   The core generates fragments for all triangles in its assigned tile list [S1].
    *   Blending is performed in shared memory to resolve fragment conflicts (e.g., max-blending for normals) [S1].
    *   The resulting voxel values are written to global memory [S1].

**Triangle/Box Overlap Test:**
The overlap test is an extension of the 2D variant by Möller and Aila [2005], as described by Schwarz and Seidel [2010], with factorized computations [S1]. It determines if a triangle intersects a voxel box [S1]. The specific implementation details of the factorization are not provided in the sources [S1].

### A-Buffer Mode (Bucketing)
This mode performs 3D binning of triangles over uniform grids using a sort-last pipeline [S1].

1.  **Batching:** Input triangles are batched by size and orientation [S1].
2.  **Fragment Generation:** Fragments are generated for all triangles [S1].
3.  **Sorting:** Fragments are sorted by voxel ID using efficient sorting primitives [S1].
4.  **Fragment Storage:** User-defined fragment shaders store specific fragment data, including interpolated attributes generated using barycentric coordinates, into the voxel lists [S1]. Unlike previous binning systems that store plain triangle ID lists, VoxelPipe stores full fragment data [S1].

### Parameters
*   **Tile Size:** Limited to $16^3$ for floating-point buffers due to shared memory constraints [S1].
*   **Voxelization Type:** Fully conservative or thin [S1].
*   **Separation:** Supports accurate 6-separating and 26-separating voxelization [S1].

### Edge Cases and Pitfalls
*   **Hairball Scene Performance:** In floating-point buffers, the Hairball scene (2.88M triangles) performs poorly (<2x better than Schwarz and Seidel 2010) at high resolutions [S1]. This is due to: 1) Limited tile size ($16^3$) increasing coarse raster work (sorting more triangle/tile pairs) [S1]. 2) Many long/skinny triangles generating variable fragment counts, leading to low SIMT utilization in the fine raster stage [S1].
*   **Shared Memory Constraints:** The tile size limit for floating-point buffers is a hard constraint imposed by hardware shared memory limits [S1].

## Subtopics

### Blending-Based Rasterization (VoxelPipe Standard Mode)
A sort-middle tile-based rasterization pipeline divided into coarse raster and fine raster stages [S1]. Coarse raster divides the grid into coarse tiles, performs triangle setup, and emits triangles overlapping each tile [S1]. Fine raster assigns each tile to a core, generates fragments, and performs blending in shared memory [S1]. Advantages include load balancing by breaking large triangles across multiple tiles and reduced expensive memory bandwidth by doing heavy-lifting in the upper memory hierarchy (shared memory) [S1].

## Operating characteristics
*   **Performance:** VoxelPipe is up to 28 times superior to state-of-the-art binary voxelizers (Schwarz and Seidel 2010) and up to 4 times faster than previous triangle binning implementations (Kalojanov and Slusallek 2009; Kalojanov et al. 2011) [S1].
*   **Hardware:** Benchmarks were run on a GeForce GTX480 GPU with 1GB GPU memory and an Intel Core i7 860 @ 2.8GHz CPU with 4GB main memory [S1].
*   **Timing Breakdown (Binary Voxelization):**
    *   Conference (282k triangles): 128 res: Coarse 0.5ms, Sorting 0.7ms, Fine 2.2ms [S1]. 512 res: Coarse 0.5ms, Sorting 1.2ms, Fine 2.6ms [S1]. 1024 res: Coarse 0.6ms, Sorting 1.4ms, Fine 6.4ms [S1].
    *   Hairball (2.88M triangles): 128 res: Coarse 2.1ms, Sorting 2.4ms, Fine 7.9ms [S1]. 512 res: Coarse 2.3ms, Sorting 4.8ms, Fine 10.7ms [S1]. 1024 res: Coarse 2.8ms, Sorting 6.9ms, Fine 24.0ms [S1].
    *   Asia Dragon (7.21M triangles): 128 res: Coarse 4.8ms, Sorting 4.2ms, Fine 14.0ms [S1]. 512 res: Coarse 4.8ms, Sorting 7.5ms, Fine 10.1ms [S1]. 1024 res: Coarse 4.9ms, Sorting 8.6ms, Fine 10.1ms [S1].
*   **Comparison with Kalojanov and Slusallek (KS) for Bucketing:**
    *   Conference (282k): KS 13.5ms vs VP 3.8ms (3.55x speedup) [S1].
    *   Dragon (180k): KS 4.0ms vs VP 3.7ms (1.08x) [S1].
    *   Fairy (174k): KS 12.0ms vs VP 4.2ms (2.85x) [S1].
    *   Soda Hall (2.2M): KS 65.0ms vs VP 15.0ms (4.33x) [S1].
    *   Note: KS timings were divided by 2x to estimate GTX480 performance from GTX280 data [S1].
*   **Strengths:** High flexibility, programmability, efficient load balancing for uneven geometric density, and fast performance for both binary and A-buffer modes [S1].
*   **Limitations:** Performance degrades for scenes with many long/skinny triangles in floating-point buffers at high resolutions due to SIMT utilization issues and coarse raster overhead [S1].

## Background
VoxelPipe was introduced in [S1]. The source code is available at http://code.google.com/p/voxelpipe/ [S1]. It is used in real-time global illumination systems (AnonGI) to create a scene proxy for tracing incoherent rays, rebuilt every frame for dynamic geometry, materials, and lighting, running at over 50 FPS at 1024x1024 resolution [S1].

## Sources
[S1] VoxelPipe: A Programmable Pipeline for 3D Voxelization, https://research.nvidia.com/sites/default/files/pubs/2011-08_VoxelPipe-A-Programmable/voxel-pipe.pdf, fetched 2026-10-27
