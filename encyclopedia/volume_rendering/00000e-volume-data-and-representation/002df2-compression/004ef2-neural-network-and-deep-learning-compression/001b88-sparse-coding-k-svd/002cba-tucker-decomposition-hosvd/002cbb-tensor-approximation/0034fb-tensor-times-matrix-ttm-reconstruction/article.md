# Tensor Times Matrix (TTM) Reconstruction

Also called: CUDA TTM Reconstruction, CUDA Tensor Times Matrix (TTM) Reconstruction, GPU Tensor Reconstruction via TTM, TTM Reconstruction, TTM-based decoding, n-mode product reconstruction

## Overview

Tensor Times Matrix (TTM) Reconstruction is a GPU-accelerated method for reconstructing volume data bricks from their Tucker decomposition, specifically designed to optimize computational cost and memory hierarchy usage on GPGPUs [S2]. It solves the problem of high memory bandwidth and computational complexity associated with naive tensor reconstruction by refactoring the core tensor and factor matrix multiplications into three sequential, memory-efficient kernel calls [S2]. Unlike simple reconstruction which requires cubic complexity $O(R^3)$ and full core tensor availability in each thread, TTM reconstruction reduces per-voxel complexity to linear growth $O(R)$ with respect to the rank $R$ [S2].

## How it works

**Inputs and Outputs**
*   **Input:** A core tensor $B$ of size $R_1 \times R_2 \times R_3$, and three factor matrices $U^{(1)}$ ($I_1 \times R_1$), $U^{(2)}$ ($I_2 \times R_2$), and $U^{(3)}$ ($I_3 \times R_3$) [S2]. The data is typically quantized (e.g., 16-bit) [S2].
*   **Output:** A reconstructed volume brick $A_f$ of size $I_1 \times I_2 \times I_3$ (typically $32^3$ voxels) [S2].

**Data Structures**
*   **Core Tensor $B$:** Stored as a 3D array of size $R_1 \times R_2 \times R_3$ [S2].
*   **Factor Matrices $U^{(n)}$:** Stored as 2D arrays where $U^{(1)}$ is $I_1 \times R_1$, $U^{(2)}$ is $I_2 \times R_2$, and $U^{(3)}$ is $I_3 \times R_3$ [S2].
*   **Intermediate Tensors:** $B'$ of size $I_1 \times R_2 \times R_3$ (output of TTM1, input to TTM2) and $B''$ of size $I_1 \times I_2 \times R_3$ (output of TTM2, input to TTM3) [S2].
*   **Shared Memory:** Used per thread block to store slices of factor matrices and core tensors to avoid global memory latency [S2]. Bank conflicts are avoided by dequantizing data to 32-bit floating-point words before uploading to shared memory [S2].

**Formulas**
The reconstruction is defined by the sequential n-mode products [S1, S2]:
$$ A_f = B \times_1 U^{(1)} \times_2 U^{(2)} \times_3 U^{(3)} $$

Where $\times_n$ denotes the n-mode product [S1]. The sequential steps are:
1.  $B' = B \times_1 U^{(1)}$
2.  $B'' = B' \times_2 U^{(2)}$
3.  $A_f = B'' \times_3 U^{(3)}$

**Parameters**
*   **Brick Size:** Typically $32^3$ voxels [S2].
*   **Rank:** Typically $(16, 16, 16)$ for tested configurations [S2].
*   **Quantization:** 16-bit for factor matrices and core tensor in tested configurations [S2].
*   **Shared Memory Limits:** 16 KB for CUDA 1.x, 48 KB for CUDA 2.x per multiprocessor [S2].

**Algorithm Steps**
The reconstruction is performed on the GPU using three successive kernel calls (TTM1, TTM2, TTM3) [S2]. One thread block is assigned per decoded brick [S2]. Within a block, one thread computes one voxel of the output brick [S2].

1.  **TTM1 (Mode 1):** Load $U^{(1)}$ and a slice of $B$ into shared memory [S2]. Dequantize elements to 32-bit floats [S2]. Compute $B'(i_1, r_2, r_3) = \sum_{r_1=1}^{R_1} U^{(1)}(i_1, r_1) \cdot B(r_1, r_2, r_3)$ [S2]. Store result in $B'$ [S2].
2.  **TTM2 (Mode 2):** Load $U^{(2)}$ and a slice of $B'$ into shared memory [S2]. Dequantize elements [S2]. Compute $B''(i_1, i_2, r_3) = \sum_{r_2=1}^{R_2} U^{(2)}(i_2, r_2) \cdot B'(i_1, r_2, r_3)$ [S2]. Store result in $B''$ [S2].
3.  **TTM3 (Mode 3):** Load $U^{(3)}$ and a slice of $B''$ into shared memory [S2]. Dequantize elements [S2]. Compute $A_f(i_1, i_2, i_3) = \sum_{r_3=1}^{R_3} U^{(3)}(i_3, r_3) \cdot B''(i_1, i_2, r_3)$ [S2]. Add the contribution of the "hot-corner" coefficient $B(0,0,0)$ directly from its unquantized value [S2]. Store $A_f$ in a decoding buffer [S2].
4.  **Post-Reconstruction:** Perform a device-to-device copy of the decoded brick to the GPU texture cache (bound to a CUDA array) for rendering [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zeros(shape):
    """Helper to allocate a 3D list of zeros."""
    return [[[0.0 for _ in range(shape[2])] for _ in range(shape[1])] for _ in range(shape[0])]

def dequantize(val, scale, offset):
    """Helper for dequantization. Sources specify linear dequantization for factor matrices
    and log dequantization for the core tensor [S2]."""
    return val * scale + offset

def ttm_reconstruct(core_tensor_B, factor_U1, factor_U2, factor_U3, hot_corner_val, scales, offsets):
    """
    Reconstructs a volume brick using sequential TTM kernels.
    Inputs:
        core_tensor_B: 3D list [R1, R2, R3]
        factor_U1: 2D list [I1, R1]
        factor_U2: 2D list [I2, R2]
        factor_U3: 2D list [I3, R3]
        hot_corner_val: float, unquantized value of B(0,0,0)
        scales, offsets: tuples of (scale, offset) for dequantization per tensor/matrix
    Output:
        brick: 3D list [I1, I2, I3]
    """
    I1, R1 = len(factor_U1), len(factor_U1[0])
    I2, R2 = len(factor_U2), len(factor_U2[0])
    I3, R3 = len(factor_U3), len(factor_U3[0])
    
    # Step 1: TTM1 -> B' = B x1 U1
    # B' shape: [I1, R2, R3]
    B_prime = zeros((I1, R2, R3))
    for i1 in range(I1):
        for r2 in range(R2):
            for r3 in range(R3):
                sum_val = 0.0
                for r1 in range(R1):
                    u1_val = dequantize(factor_U1[i1][r1], scales[0][0], scales[0][1])
                    b_val = dequantize(core_tensor_B[r1][r2][r3], scales[1][0], scales[1][1])
                    sum_val += u1_val * b_val
                B_prime[i1][r2][r3] = sum_val

    # Step 2: TTM2 -> B'' = B' x2 U2
    # B'' shape: [I1, I2, R3]
    B_double_prime = zeros((I1, I2, R3))
    for i1 in range(I1):
        for i2 in range(I2):
            for r3 in range(R3):
                sum_val = 0.0
                for r2 in range(R2):
                    u2_val = dequantize(factor_U2[i2][r2], scales[2][0], scales[2][1])
                    b_prime_val = B_prime[i1][r2][r3]
                    sum_val += u2_val * b_prime_val
                B_double_prime[i1][i2][r3] = sum_val

    # Step 3: TTM3 -> Af = B'' x3 U3
    # Af shape: [I1, I2, I3]
    Af = zeros((I1, I2, I3))
    for i1 in range(I1):
        for i2 in range(I2):
            for i3 in range(I3):
                sum_val = 0.0
                for r3 in range(R3):
                    u3_val = dequantize(factor_U3[i3][r3], scales[3][0], scales[3][1])
                    b_double_prime_val = B_double_prime[i1][i2][r3]
                    sum_val += u3_val * b_double_prime_val
                
                # Add hot-corner contribution directly from unquantized value [S2]
                sum_val += hot_corner_val
                Af[i1][i2][i3] = sum_val
                
    return Af
```
*Note: The actual GPU implementation uses shared memory tiling and parallel thread blocks as described in the sources [S2]. Specifically, TTM2 and TTM3 split the matrix-matrix multiplication into two blocks to optimize shared memory usage, loading only half of the factor matrix per block [S2]. The sources do not specify the exact shared memory tiling layout or synchronization primitives beyond `syncthreads()` [S2].*

## Subtopics

### [Partial Decoding](14061-partial-decoding.md)
A technique for visualizing 4D Tensor Train volumes that enables interactive time access by avoiding full 4D tensor reconstruction. It differs from TTM by focusing on skipping data blocks for time-series data rather than reconstructing 3D spatial bricks. Choose it when interactive frame rates are required for large 4D datasets and memory is constrained.

### [Incremental Tensor Reconstruction](15431-incremental-tensor-reconstruction.md)
A GPU-based method that optimizes the sequential application of TTM kernels by rearranging the summation order to reduce floating-point operations. It differs from TTM by focusing on algebraic rearrangement of the full summation to minimize operations rather than skipping data blocks. Choose it when computational throughput is the primary bottleneck and the Tucker model is already applied.

## Operating characteristics

*   **Time Complexity:** The computational complexity per reconstructed voxel grows linearly with the rank $R$, i.e., $O(R)$, compared to $O(R^3)$ for simple reconstruction [S2].
*   **Memory Complexity:** The method is memory efficient and does not require the complete core tensor to be available in each thread [S2]. Intermediate tensors $B'$ and $B''$ are stored in global memory between kernel calls [S2].
*   **Measured Performance:** On a GeForce GTX 480 (Intel Core 2 E8500, 4GB RAM), interactive rendering frame rates average 50-100Hz, with a minimum of 12Hz in demanding situations [S2]. Tensor reconstruction constitutes negligible overhead compared to ray-casting and data transfer [S2].
*   **Scaling:** The method scales with the number of bricks to be decoded in the current frame, with grid size determined dynamically [S2].
*   **Strengths:** Optimized for GPGPU memory hierarchy and shared memory usage [S2]. High interactivity is achieved through negligible reconstruction overhead relative to ray-casting [S2].
*   **Limitations:** Requires three sequential kernel calls, introducing some overhead [S2]. The most costly part of the tensor reconstruction process is the final copy of the decoded bricks to the texture cache [S2].
*   **When to use/not use:** Use for multiresolution volume ray-casting on GPUs where brick-wise reconstruction is required and memory bandwidth is limited [S2]. Avoid when preprocessing time is prohibitive or when single-step reconstruction fits within thread memory constraints [S2].

## Background

The method was introduced as a GPU-based volume reconstruction technique for multiresolution volume rendering, specifically optimizing brick-wise reconstruction for volume ray-casting using a multiresolution octree hierarchy [S2]. It addresses the limitations of simple tensor reconstruction, which suffers from cubic complexity and high memory requirements [S2]. The approach leverages the CUDA framework to optimize the n-mode product operations inherent in Tucker decomposition [S2].

## Sources

[S1] An Algorithm for Choosing Optimal Rank in Tensor Approximation of Multi-Resolution Volume Rendering, https://doi.org/10.12677/csa.2018.811183, fetched 2026-10-26
[S2] Interactive Multiscale Tensor Reconstruction for Multiresolution Volume Rendering, https://ics.uci.edu/~gopi/SamplePubs/MultiscaleTensorVis.pdf, fetched 2026-10-26
