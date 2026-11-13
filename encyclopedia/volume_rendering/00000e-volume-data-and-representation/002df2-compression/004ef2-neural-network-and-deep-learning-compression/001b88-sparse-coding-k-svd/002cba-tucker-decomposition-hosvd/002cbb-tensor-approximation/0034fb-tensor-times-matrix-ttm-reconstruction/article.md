# Tensor Times Matrix (TTM) Reconstruction

Also called: CUDA TTM Reconstruction, CUDA Tensor Times Matrix (TTM) Reconstruction, GPU Tensor Reconstruction via TTM, TTM Reconstruction, TTM-based decoding, n-mode product reconstruction

## Overview
Tensor Times Matrix (TTM) Reconstruction is a GPU-accelerated method for reconstructing volume data bricks from their Tucker decomposition (core tensor and factor matrices) by sequentially applying n-mode products [S2]. It solves the problem of high computational complexity and memory pressure associated with naive tensor reconstruction, reducing the per-voxel complexity from cubic $O(R^3)$ to linear $O(R)$ with respect to the rank $R$ [S2]. Unlike naive approaches that require the entire core tensor in each thread, TTM reconstruction refactors the computation into three sequential matrix-matrix multiplications, optimizing for the GPGPU memory hierarchy by storing intermediate results in shared memory and device linear memory [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A core tensor $B$ of size $R_1 \times R_2 \times R_3$, and three factor matrices $U^{(1)}$ ($I_1 \times R_1$), $U^{(2)}$ ($I_2 \times R_2$), and $U^{(3)}$ ($I_3 \times R_3$) [S1, S2]. The data is typically quantized (e.g., 16-bit) [S2].
*   **Outputs:** A reconstructed volume brick $A_f$ of size $I_1 \times I_2 \times I_3$ [S2].

**Data Structures and Layout:**
The reconstruction is performed brick-wise. One CUDA thread block is assigned to decode one brick [S2].
*   **Shared Memory:** Used to store slices of the factor matrices and core tensors to avoid bank conflicts. Data is dequantized to 32-bit floating point words before upload [S2].
*   **Intermediate Tensors:**
    *   $B'$: Size $B \times R_2 \times R_3$ (where $B$ is the brick dimension, e.g., 32) [S2].
    *   $B''$: Size $B \times B \times R_3$ [S2].
*   **Final Buffer:** $A_f$ is stored in an intermediate decoding buffer in device linear memory, then copied to the GPU texture cache (bound to a CUDA array) for rendering [S2].

**Parameters:**
*   **Brick Size:** Typically $32^3$ [S2].
*   **Rank:** Typically $(16, 16, 16)$ [S2].
*   **Quantization:** 16-bit for factor matrices and core tensor [S2].
*   **Shared Memory Limits:** 16 KB (CUDA 1.x) or 48 KB (CUDA 2.x) per multiprocessor [S2].

**Algorithm Steps:**
The reconstruction follows the Tucker formula:
$$ A_f = B \times_1 U^{(1)} \times_2 U^{(2)} \times_3 U^{(3)} $$
This is executed as three sequential GPU kernel calls [S2].

1.  **TTM1 (Mode 1):** Computes $B' = B \times_1 U^{(1)}$.
    *   Input: $U^{(1)}$ and slices of $B$.
    *   Output: Intermediate tensor $B'$ of size $B \times R_2 \times R_3$.
    *   Computation: Each thread computes one element of $B'$ by summing the product of elements from $U^{(1)}$ and $B$ over the first mode [S2].
2.  **TTM2 (Mode 2):** Computes $B'' = B' \times_2 U^{(2)}$.
    *   Input: $U^{(2)}$ and slices of $B'$.
    *   Output: Intermediate tensor $B''$ of size $B \times B \times R_3$.
    *   Computation: Similar to TTM1, but operates on the second mode [S2].
3.  **TTM3 (Mode 3):** Computes $A_f = B'' \times_3 U^{(3)}$.
    *   Input: $U^{(3)}$ and slices of $B''$.
    *   Output: Final volume brick $A_f$ of size $B \times B \times B$ (i.e., $I_1 \times I_2 \times I_3$).
    *   Computation: Computes the final voxel values. A special case handles the "hot-corner" coefficient $B(0,0,0)$, which is added directly from its unquantized value to avoid special-case handling in the inner loop [S2].

**Pseudocode:**
The following pseudocode represents the three CUDA kernels. It assumes standard CUDA primitives (shared memory, synchronization) and that data is loaded into shared memory before the kernel execution loop.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global constants
BRICK_SIZE = 32
RANK = 16

# Shared memory declarations (size depends on brick and rank)
# TTM1: U_shared[RANK][BRICK_SIZE], B_shared[RANK][RANK][RANK]
# TTM2/TTM3: U_shared[RANK][BRICK_SIZE/2], B_shared[BRICK_SIZE][BRICK_SIZE][RANK]

def kernel_ttm1(U1, B_core, B_prime_out):
    """
    Computes B' = B x1 U(1)
    U1: Factor matrix U(1) of size I1 x R1
    B_core: Core tensor B of size R1 x R2 x R3
    B_prime_out: Output intermediate tensor B' of size B x R2 x R3
    """
    # 1. Load data to shared memory
    # Each thread loads one element of U1 and one element of B slice
    # Dequantize U1 and B to 32-bit float during load
    load_shared(U1, B_core)
    sync_threads()

    # 2. Compute
    # Thread block computes one slice of B'
    # Each thread computes one element of the slice
    thread_id = get_thread_id()
    i1 = thread_id % I1
    r2 = (thread_id // I1) % R2
    r3 = (thread_id // I1) // R2
    
    voxel = 0.0
    for r1 in range(R1):
        # Access shared memory
        u_val = U_shared[r1][i1]
        b_val = B_shared[r1][r2][r3]
        voxel += u_val * b_val
    
    # 3. Store result
    B_prime_out[i1][r2][r3] = voxel

def kernel_ttm2(U2, B_prime, B_double_prime_out):
    """
    Computes B'' = B' x2 U(2)
    U2: Factor matrix U(2) of size I2 x R2
    B_prime: Intermediate tensor B' of size B x R2 x R3
    B_double_prime_out: Output intermediate tensor B'' of size B x B x R3
    """
    # 1. Load data to shared memory
    # Load half of U2 and full slice of B' to optimize shared memory
    load_shared(U2, B_prime)
    sync_threads()

    # 2. Compute
    thread_id = get_thread_id()
    i2 = thread_id % I2
    i1 = (thread_id // I2) % B
    r3 = (thread_id // I2) // B
    
    voxel = 0.0
    for r2 in range(R2):
        u_val = U_shared[r2][i2]
        b_val = B_shared[i1][r2][r3]
        voxel += u_val * b_val
    
    # 3. Store result
    B_double_prime_out[i1][i2][r3] = voxel

def kernel_ttm3(U3, B_double_prime, A_f_out, hot_corner_val):
    """
    Computes A_f = B'' x3 U(3)
    U3: Factor matrix U(3) of size I3 x R3
    B_double_prime: Intermediate tensor B'' of size B x B x R3
    A_f_out: Output volume brick A_f of size B x B x B
    hot_corner_val: Unquantized value of B(0,0,0)
    """
    # 1. Load data to shared memory
    load_shared(U3, B_double_prime)
    sync_threads()

    # 2. Compute
    thread_id = get_thread_id()
    i3 = thread_id % I3
    i1 = (thread_id // I3) % B
    i2 = (thread_id // I3) // B
    
    voxel = 0.0
    for r3 in range(R3):
        u_val = U_shared[r3][i3]
        b_val = B_shared[i1][i2][r3]
        voxel += u_val * b_val
    
    # 3. Add hot-corner contribution
    # The hot-corner B(0,0,0) is handled separately to simplify decoding
    if i1 == 0 and i2 == 0 and i3 == 0:
        voxel += hot_corner_val
    
    # 4. Store result
    A_f_out[i1][i2][i3] = voxel
```

**Edge Cases and Pitfalls:**
*   **Shared Memory Bank Conflicts:** Avoided by dequantizing factor matrices and core tensors to 32-bit floating point words before uploading to shared memory, ensuring all accesses are aligned on 32-bit words [S2].
*   **Hot-Corner Coefficient:** The coefficient $B(0,0,0)$ is handled separately in TTM3. To simplify decoding and avoid special case handling in the inner loop, a zero quantized coefficient is encoded at the corresponding core tensor position, and the unquantized value is added directly [S2].
*   **Memory Limits:** Shared memory usage must fit within the multiprocessor limits (16 KB for CUDA 1.x, 48 KB for CUDA 2.x). The configuration uses $(B \cdot R \cdot 4 + R \cdot R \cdot 4)$ bytes for TTM1 and $(B/2 \cdot R \cdot 4 + B \cdot R \cdot 4)$ bytes for TTM2/TTM3, which works with 32-cubed bricks [S2].

## Subtopics

### [Partial Decoding](14061-partial-decoding.md)
Partial Decoding is a technique for visualizing 4D Tensor Train (TT) volumes that enables interactive time access and low memory usage by avoiding the reconstruction of the full 4D tensor. It differs from naive reconstruction, which recreates the entire 4D volume in memory, and per-time-step reconstruction, which is too slow for interactive frame rates on large datasets. By pre-computing a reduced base tensor, it reduces the per-time-step cost to a single tensor-vector multiplication.

## Operating characteristics
*   **Time Complexity:** The computational complexity per reconstructed voxel grows linearly with the rank $R$, i.e., $O(R)$, compared to the cubic $O(R^3)$ complexity of simple reconstruction [S2].
*   **Memory Complexity:** The method is memory efficient compared to naive reconstruction, which requires the complete core tensor in each thread. It uses intermediate tensors of size $B \times R^2$ and $B^2 \times R$ [S2].
*   **Performance:** On a GeForce GTX 480 (Intel Core 2 E8500, 4GB RAM), interactive rendering frame rates were higher than 12Hz in demanding situations, averaging 50-100Hz [S2]. Tensor reconstruction constitutes negligible overhead compared to ray-casting and data transfer [S2].
*   **Preprocessing Time:** For a 2GB chameleon dataset, preprocessing took 25min 15sec, producing a 231M file. For a 17GB tooth dataset, it took 8h 45min, producing a 5.5GB file [S2].
*   **Strengths:** Optimizes computational costs, leverages GPGPU memory hierarchy, enables interactive rendering for large volumes [S2].
*   **Limitations:** Requires three sequential kernel calls and intermediate storage. Performance is limited by the final copy of decoded bricks to texture cache [S2].

## Background
TTM Reconstruction was introduced as an optimized GPU-based volume reconstruction method for multiresolution volume rendering [S2]. It builds on the Tucker decomposition (HOSVD) and addresses the limitations of naive tensor reconstruction by refactoring computations into sequential tensor times matrix multiplications [S2]. The method was developed to support interactive rendering of large-scale micro-CT datasets, such as the veiled chameleon and great ape molar (tooth) datasets [S2].

## Sources
[S1] An Algorithm for Choosing Optimal Rank in Tensor Approximation of Multi-Resolution Volume Rendering, https://doi.org/10.12677/csa.2018.811183, fetched 2026-10-26
[S2] PDF Interactive Multiscale Tensor Reconstruction for Multiresolution Volume ..., https://ics.uci.edu/~gopi/SamplePubs/MultiscaleTensorVis.pdf, fetched 2026-10-26
