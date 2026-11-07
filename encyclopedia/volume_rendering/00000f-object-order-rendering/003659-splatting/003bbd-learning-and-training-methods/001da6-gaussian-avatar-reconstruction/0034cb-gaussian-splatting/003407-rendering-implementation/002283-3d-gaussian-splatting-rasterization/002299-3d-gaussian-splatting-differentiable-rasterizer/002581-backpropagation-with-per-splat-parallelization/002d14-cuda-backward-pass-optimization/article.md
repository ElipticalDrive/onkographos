# CUDA Backward Pass Optimization

Also called: Backward pass CUDA optimisation, CUDA Backward Pass Optimisations, CUDA Parallel Optimisation for Backward Pass, CUDA-level optimisation, Efficient CUDA Backward Optimisation, cu-opt

## Overview
This method applies low-level GPU kernel optimizations to the 3D Gaussian Splatting backward pass to reduce memory access overhead and improve cache utilization [S1, unofficial]. It differs from structural optimizations like Loss-Accumulation Guided (LAG) densification or pruning by reducing training time independently of the number of Gaussian primitives, providing consistent speedups across all configurations [S1, unofficial]. While LAG and pruning reduce the Gaussian count, this CUDA-level optimization reduces per-iteration execution overhead without altering rendering results or optimization behavior [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The original 3D Gaussian Splatting backward pass kernel parameters (gradients, Gaussian attributes, camera parameters).
*   **Output:** Optimized backward pass execution with reduced global memory transactions and improved warp-level parallelism.

**Data Structures and Layout:**
The optimization relies on a specific thread block organization and memory hierarchy [S1, unofficial]:
1.  **Thread Block Organization:** Each block uses 8 warps, totaling 256 threads. Each thread corresponds to a single pixel (1:1 thread-to-pixel mapping) [S1, unofficial]. This contrasts with the original 3DGS implementation where each block contains 32 threads and each thread loads data from multiple pixels [S1, unofficial].
2.  **Memory Hierarchy:** Tile-level data are stored in shared memory to enable reuse across warps within the block [S1, unofficial].

**Steps:**
1.  **Redesign Thread Organization:** Reconfigure the kernel launch parameters so that each thread block consists of 256 threads (8 warps). Assign one thread per pixel to be processed [S1, unofficial].
2.  **Adjust Memory Layouts:** Restructure data access patterns to enable vectorised memory access. This involves loading tile-level data into shared memory so that threads within a warp can reuse the same data [S1, unofficial].
3.  **Reduce Redundant Transactions:** By leveraging warp-level data reuse via shared memory and vectorised access, eliminate redundant global memory transactions [S1, unofficial].

**Pseudocode:**
The following pseudocode illustrates the kernel structure. Note that the actual gradient computation logic remains unchanged; only the data access pattern is optimized [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel launch configuration
# gridDim.x = ceil(image_width / 16)
# gridDim.y = ceil(image_height / 16)
# blockDim.x = 16
# blockDim.y = 16
# Total threads per block = 256 (8 warps)

@cuda.jit
def backward_pass_kernel(gradients, gaussians, cameras, shared_mem):
    # Calculate pixel coordinates
    pixel_x = cuda.blockIdx.x * 16 + cuda.threadIdx.x
    pixel_y = cuda.blockIdx.y * 16 + cuda.threadIdx.y
    
    # Check bounds
    if pixel_x >= image_width or pixel_y >= image_height:
        return

    # Load tile-level data into shared memory for warp-level reuse
    # This step replaces repeated global memory loads
    tile_data = load_tile_data_into_shared_memory(pixel_x, pixel_y, shared_mem)
    
    # Synchronize to ensure all threads in the block have loaded data
    cuda.syncthreads()
    
    # Each thread processes one pixel
    # Access data from shared memory (tile_data) instead of global memory
    # Compute gradients for the specific pixel using vectorised access patterns
    pixel_gradients = compute_pixel_gradients(pixel_x, pixel_y, tile_data, gaussians, cameras)
    
    # Write results to global memory
    gradients[pixel_y, pixel_x] = pixel_gradients
```

**Parameters:**
*   **Block Size:** 256 threads (8 warps) [S1, unofficial].
*   **Warp Size:** 32 threads (standard NVIDIA GPU warp size) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The optimization does not alter the mathematical operations, so numerical precision remains identical to the original implementation [S1, unofficial].
*   **Boundaries:** Standard boundary checks for image dimensions are required, as with any rasterization kernel [G].

## Subtopics

### Loss-Accumulation Guided (LAG) Densification
A strategy that inhibits unnecessary Gaussian generation in well-reconstructed regions by combining opacity, opacity gradients, and training loss. Choose this when you want to reduce the total number of Gaussians during training. [LAG](3532-loss-accumulation-guided-densification.md)

### Pruning Strategy
Removes unstable or low-contribution primitives after densification based on opacity and pruning scores. Choose this to further reduce Gaussian count and training time with negligible impact on rendering quality. [Pruning](3531-pruning-strategy.md)

### Backpropagation with Per-Splat Parallelization
Maps threads to 2D splats instead of pixels to avoid atomic contention on gradient accumulation. Choose this when atomic contention is the primary bottleneck. [Per-Splat Parallelization](3533-backpropagation-with-per-splat-parallelization.md)

## Operating characteristics
*   **Time Complexity:** Reduces training time by approximately 7% on Mip-NeRF 360 datasets (200.1s vs 215.4s for LAG alone) [S1, unofficial].
*   **Memory Complexity:** Improves cache utilization by using shared memory for tile-level data reuse, reducing redundant global memory transactions [S1, unofficial].
*   **Scaling:** Provides consistent speedups across all configurations because it reduces overhead independent of the number of Gaussians [S1, unofficial].
*   **Strengths:** Low-level optimization that does not require changes to the high-level training loop or data structures. Maintains competitive PSNR, SSIM, and LPIPS performance [S1, unofficial].
*   **Limitations:** Requires specific CUDA kernel restructuring. Effectiveness depends on GPU architecture's shared memory and warp execution efficiency [S1, unofficial].
*   **When to Use:** When training time is a bottleneck and structural optimizations (LAG, pruning) have already been applied or are insufficient.
*   **When Not to Use:** When the primary bottleneck is not memory access overhead in the backward pass, or when hardware does not support efficient warp-level shared memory access.

## Background
This method was introduced in the paper "Loss-Guided Density Control and Efficient Backward Optimisation for Compact Gaussian Splatting" [S1, unofficial]. It is part of a framework that combines loss-aware density control (LAG), pruning, and CUDA-level optimizations to achieve faster and more compact Gaussian Splatting. The optimization specifically targets the backward pass kernel, which is a significant component of training time in 3D Gaussian Splatting [S1, unofficial].

## Sources
[S1] Loss-Guided Density Control and Efficient Backward Optimisation for Compact Gaussian Splatting, https://doi.org/10.21203/rs.3.rs-10572680/v1, fetched 2026-10-26 (unofficial)
