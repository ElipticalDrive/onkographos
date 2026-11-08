# CUDA Backward Pass Optimization

Also called: Backward pass CUDA optimisation, CUDA Backward Pass Optimisations, CUDA Parallel Optimisation for Backward Pass, CUDA-level optimisation, Efficient CUDA Backward Optimisation, cu-opt

## Overview
This method applies low-level CUDA kernel optimizations to the 3D Gaussian Splatting backward pass to reduce memory access overhead and improve cache utilization [S1, unofficial]. It specifically targets the reduction of redundant global memory transactions and improves warp-level parallelism without altering rendering results or optimization behavior [S1, unofficial]. It differs from structural optimizations like Loss-Accumulation Guided (LAG) densification or pruning by reducing training time independently of the number of Gaussians, providing consistent speedups across all configurations [S1, unofficial].

## How it works
The optimization redesigns the thread organization and memory layout of the backward pass kernel to enable data reuse and vectorized access [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Gradient tensors for Gaussian parameters (position, scale, rotation, opacity, spherical harmonics coefficients) and intermediate forward pass data (screen-space positions, covariances, depths) [S1, unofficial].
*   **Outputs:** Updated gradients accumulated in global memory for the optimizer [S1, unofficial].

**Data Structures and Layout:**
*   **Thread Block Organization:** Each block uses 8 warps (256 threads) [S1, unofficial]. Each thread corresponds to a single pixel (1:1 thread-to-pixel mapping) [S1, unofficial]. This contrasts with the original 3DGS implementation where each block contains 32 threads and each thread loads data from multiple pixels [S1, unofficial].
*   **Memory Hierarchy:** Tile-level data required for the backward pass are stored in shared memory for reuse across warps within the block [S1, unofficial].
*   **Vectorized Access:** Memory layouts are adjusted to support vectorized memory access (e.g., `float4` loads/stores) to reduce the number of global memory transactions [S1, unofficial].

**Steps:**
1.  **Tile Loading:** The kernel loads the necessary forward pass intermediates (e.g., projected centers, covariances, depths) for the current tile into shared memory [S1, unofficial].
2.  **Thread Mapping:** Assign each of the 256 threads in the block to a unique pixel within the tile [S1, unofficial].
3.  **Gradient Computation:** Each thread computes the local gradient contribution for its assigned pixel using the shared data [S1, unofficial].
4.  **Atomic Accumulation:** Threads atomically accumulate their computed gradients into the global gradient buffers [S1, unofficial]. The 1:1 mapping reduces the contention compared to the original multi-pixel-per-thread approach where threads might access overlapping memory regions [S1, unofficial].
5.  **Vectorized Stores:** Use vectorized store instructions (e.g., `float4`) to write gradient updates to global memory, improving memory bandwidth utilization [S1, unofficial].

**Parameters:**
*   **Block Size:** 256 threads (8 warps) [S1, unofficial].
*   **Warp Size:** 32 threads (standard NVIDIA GPU warp size) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Ensure that threads corresponding to pixels outside the valid tile bounds perform no work or handle boundary checks efficiently to avoid divergence [G].
*   **Shared Memory Limits:** The tile size must be chosen such that the required forward pass data fits within the shared memory limit of the GPU architecture [G].

## Subtopics

### Backward Pass Optimization for Gaussian Splatting
This variant, introduced in FaCT-GS, reformulates the backward pass in a per-Gaussian manner to eliminate write conflicts and warp stalls caused by per-pixel backward pass calculations [S2]. It constrains the range of Gaussian splats using axis-aligned, rectangular bounding boxes with factored-in density to replace square bounding boxes, reducing overestimation for anisotropic/transparent Gaussians [S2]. It also coalesces function parameters into compact `float4` structs, combining conics (projected covariances) with $\mu$ in the rasterizer to reduce memory traffic [S2]. This approach achieves a 22x acceleration on an RTX 4070 Ti SUPER, reducing rasterization time from 210 ms to 9.3 ms for a 2K image [S2].

## Operating characteristics
*   **Time Complexity:** The optimization reduces the constant factor of the backward pass execution time by minimizing memory access overhead [S1, unofficial].
*   **Memory Complexity:** No significant change in memory complexity; however, shared memory usage increases due to tile-level data storage [S1, unofficial].
*   **Measured Performance:** On Mip-NeRF 360, combining LAG with this CUDA optimization (LAG+cu-opt) achieves a training time of 200.1s versus 215.4s for LAG alone, a ~7% reduction [S1, unofficial].
*   **Strengths:** Reduces training time independently of the number of Gaussians; consistent speedups across all configurations; does not affect rendering quality [S1, unofficial].
*   **Limitations:** Requires careful tuning of tile size to fit shared memory; specific to CUDA implementation [S1, unofficial].
*   **When to Use:** When training time is a bottleneck and the hardware supports efficient warp-level parallelism and vectorized memory access [S1, unofficial].
*   **When Not to Use:** When memory bandwidth is not the primary bottleneck or when using hardware with different warp architectures [G].

## Background
This method was introduced in "Loss-Guided Density Control and Efficient Backward Optimisation for Compact Gaussian Splatting" [S1, unofficial]. It builds on the standard 3D Gaussian Splatting backward pass [S1, unofficial]. The optimization focuses on low-level GPU kernel details, specifically thread organization and data access patterns, to improve performance without changing the mathematical formulation of the rendering or optimization process [S1, unofficial].

## Sources
[S1] Loss-Guided Density Control and Efficient Backward Optimisation for Compact Gaussian Splatting, https://doi.org/10.21203/rs.3.rs-10572680/v1, fetched 2026-10-26 (unofficial)
[S2] FaCT-GS: Fast and Scalable CT Reconstruction with Gaussian Splatting, https://doi.org/10.1007/978-3-032-37464-6_13, fetched 2026-10-26
