# CUDA Backprojection Implementation
Also called: CUDA Core Implementation, CUDA Parallelization Strategy
## Overview
This method implements the backprojection step of differentiable X-ray computed tomography on NVIDIA GPUs using CUDA [S1, S2, unofficial]. It solves the problem of efficiently mapping projection data back into a 3D reconstruction volume by parallelizing computation over the output space (reconstruction voxels) [S1, S2, unofficial]. It is distinguished from CPU implementations by its use of 3D thread blocks and 3D texture memory for input data access, and it supports both Siddon and SF projector models [S1, S2, unofficial]. It builds on [GPU Backprojection](5843-gpu-backprojection.md).

## How it works
**Inputs and Outputs:**
*   **Input:** CT projection data (for backprojection) stored as a contiguous 32-bit floating-point array [S1, S2, unofficial].
*   **Input:** Geometric parameters defining the CT scan geometry (e.g., source-detector distance, detector spacing, rotation angles) [S1, S2, unofficial].
*   **Output:** Reconstruction volume stored as a contiguous 32-bit floating-point array [S1, S2, unofficial].

**Data Structures:**
*   **Projection Data:** Contiguous 1D array of `float32` values representing the 2D projection images [S1, S2, unofficial].
*   **Reconstruction Volume:** Contiguous 1D array of `float32` values representing the 3D voxel grid [S1, S2, unofficial].
*   **Texture Memory:** 3D texture objects bound to the input data for hardware-accelerated interpolation [S1, S2, unofficial].

**Parameters:**
*   **Geometry:** Specifies the CT scan setup (e.g., parallel, cone, or fan beam) [S1, S2, unofficial].
*   **Projector Model:** Either "Siddon" or "SF" (Sinc-Filtered or similar, as specified in the source context) [S1, S2, unofficial].
*   **Thread Block Dimensions:** 3D thread blocks are used [S1, S2, unofficial].

**Algorithm Steps:**
1.  **Data Preparation:** If the input projection data is not already on the GPU, copy it from CPU memory to GPU global memory [S1, S2, unofficial].
2.  **Texture Binding:** Bind the input projection data to a 3D texture object to enable efficient hardware interpolation [S1, S2, unofficial].
3.  **Kernel Launch:** Launch a CUDA kernel where each thread block corresponds to a voxel in the reconstruction volume [S1, S2, unofficial].
4.  **Voxel Processing (Backprojection):**
    *   Each thread computes the backprojection contribution for its assigned voxel [S1, S2, unofficial].
    *   For each projection angle, calculate the ray path from the X-ray source through the voxel to the detector [S1, S2, unofficial].
    *   Sample the projection data at the corresponding detector coordinates using the 3D texture memory [S1, S2, unofficial].
    *   Accumulate the sampled values into the voxel's value in the reconstruction volume [S1, S2, unofficial].
5.  **Result Retrieval:** If the output was computed on the GPU, copy the reconstruction volume back to CPU memory [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CUDA Kernel for Backprojection
# Each thread handles one voxel in the reconstruction volume
__global__ void backprojection_kernel(
    float* reconstruction_volume,
    float* projection_data,
    int num_voxels_x,
    int num_voxels_y,
    int num_voxels_z,
    int num_projections,
    float voxel_size,
    float source_to_detector_distance,
    float detector_spacing,
    float* rotation_angles,
    cudaTextureObject_t projection_texture
) {
    // Calculate voxel coordinates from thread/block indices
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;
    int z = blockIdx.z * blockDim.z + threadIdx.z;

    // Check bounds
    if (x >= num_voxels_x || y >= num_voxels_y || z >= num_voxels_z) {
        return;
    }

    float voxel_value = 0.0f;

    // Get voxel center in world coordinates
    float vx = (x + 0.5f) * voxel_size;
    float vy = (y + 0.5f) * voxel_size;
    float vz = (z + 0.5f) * voxel_size;

    // Iterate over all projection angles
    for (int i = 0; i < num_projections; ++i) {
        float angle = rotation_angles[i];

        // Calculate ray direction based on geometry (Siddon or SF model)
        // This involves rotating the voxel position relative to the source
        // and projecting onto the detector plane
        float detector_u, detector_v;
        compute_detector_coordinates(
            vx, vy, vz, angle,
            source_to_detector_distance,
            detector_spacing,
            detector_u, detector_v
        );

        // Sample projection data using 3D texture memory
        // The texture object handles interpolation and boundary checks
        float projection_value = tex3D<float>(
            projection_texture,
            detector_u,
            detector_v,
            i // Assuming projections are stored in the z-dimension of texture
        );

        // Accumulate contribution
        voxel_value += projection_value;
    }

    // Write result to reconstruction volume
    int voxel_index = x + y * num_voxels_x + z * num_voxels_x * num_voxels_y;
    reconstruction_volume[voxel_index] = voxel_value;
}

// Host function to launch the kernel
void run_backprojection(
    float* d_reconstruction_volume,
    float* d_projection_data,
    int num_voxels_x,
    int num_voxels_y,
    int num_voxels_z,
    int num_projections,
    float voxel_size,
    float source_to_detector_distance,
    float detector_spacing,
    float* d_rotation_angles,
    cudaTextureObject_t projection_texture
) {
    dim3 block(16, 16, 16); // 3D thread block
    dim3 grid(
        (num_voxels_x + block.x - 1) / block.x,
        (num_voxels_y + block.y - 1) / block.y,
        (num_voxels_z + block.z - 1) / block.z
    );

    backprojection_kernel<<<grid, block>>>(
        d_reconstruction_volume,
        d_projection_data,
        num_voxels_x,
        num_voxels_y,
        num_voxels_z,
        num_projections,
        voxel_size,
        source_to_detector_distance,
        detector_spacing,
        d_rotation_angles,
        projection_texture
    );
}
```

**Edge Cases and Pitfalls:**
*   **Data Transfer Overhead:** If data is not on the GPU, the copy overhead can dominate computation time [S1, S2, unofficial].
*   **Texture Memory Limits:** 3D texture memory has size limits; large projection datasets may require tiling or alternative storage [S1, S2, unofficial].
*   **Interpolation Artifacts:** Hardware interpolation in texture memory may introduce artifacts if not configured correctly [S1, S2, unofficial].
*   **Boundary Conditions:** Rays passing outside the detector area must be handled to avoid invalid memory accesses [S1, S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V \cdot N)$, where $V$ is the number of voxels and $N$ is the number of projections [S1, S2, unofficial].
*   **Memory Complexity:** $O(V + P)$, where $V$ is the volume size and $P$ is the projection data size [S1, S2, unofficial].
*   **Performance:** Significant speedup over CPU implementations due to parallelization over voxels and hardware interpolation [S1, S2, unofficial].
*   **Strengths:** High throughput for large volumes, efficient memory access via texture memory, support for differentiable operations via PyTorch integration [S1, S2, unofficial].
*   **Limitations:** Memory bandwidth bound for very large datasets, texture memory size constraints [S1, S2, unofficial].
*   **When to Use:** When high-speed backprojection is required for large 3D volumes and GPU resources are available [S1, S2, unofficial].
*   **When Not to Use:** When memory bandwidth is saturated or texture memory limits are exceeded [S1, S2, unofficial].

## Background
The CUDA backprojection implementation was developed as part of a differentiable forward projector for X-ray computed tomography [S1, S2, unofficial]. It was introduced in a paper published in 2023 [S1, S2, unofficial]. The implementation is part of a library that provides both CPU and GPU versions of forward and backprojection algorithms for various CT geometries [S1, S2, unofficial]. It integrates with PyTorch for automatic differentiation [S1, S2, unofficial].

## Sources
[S1] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
[S2] Differentiable Forward Projector for X-ray Computed ..., https://openreview.net/forum?id=dX8khDFGHv, fetched 2026-11-06 (unofficial)
