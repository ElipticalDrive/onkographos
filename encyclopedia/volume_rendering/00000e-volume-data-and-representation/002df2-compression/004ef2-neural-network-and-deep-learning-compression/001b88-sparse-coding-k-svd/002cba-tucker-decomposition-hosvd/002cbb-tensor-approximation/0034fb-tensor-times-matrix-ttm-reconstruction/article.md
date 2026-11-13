# Tensor Times Matrix (TTM) Reconstruction

Also called: CUDA Tensor Times Matrix (TTM) Reconstruction, GPU Tensor Reconstruction via TTM, TTM Reconstruction, TTM-based decoding, n-mode product reconstruction

## Overview
Tensor Times Matrix (TTM) Reconstruction is a GPU-based volume reconstruction method that sequentially applies three Tensor Times Matrix (TTM) kernels to reconstruct a volume brick from a core tensor and factor matrices, reducing computational complexity to linear growth with rank $R$ [S2]. It builds on the Tucker decomposition framework, specifically addressing the computational bottleneck of reconstructing volume bricks from compressed tensor representations on GPUs [S2].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Core tensor $B$ of size $R_1 \times R_2 \times R_3$ (typically $16 \times 16 \times 16$) [S2].
    *   Factor matrices $U^{(1)}, U^{(2)}, U^{(3)}$, each of size $B \times R_n$, where $B$ is the brick dimension (typically 32) and $R_n$ is the Tucker rank for mode $n$ [S2].
    *   Quantization parameters (min/max values) for dequantization [S2].
*   **Outputs:**
    *   Final volume brick $A_f$ of size $B \times B \times B$ [S2].

### Data Structures and Layout
The reconstruction is performed in three successive steps (TTM1, TTM2, TTM3) [S2]. Intermediate results are stored in device linear memory [S2].

1.  **TTM1:**
    *   **Input:** $U^{(1)}$ and slices of $B$.
    *   **Output:** Intermediate tensor $B'$ of size $B \times R_2 \times R_3$ [S2].
2.  **TTM2:**
    *   **Input:** $U^{(2)}$ and slices of $B'$.
    *   **Output:** Intermediate tensor $B''$ of size $B \times B \times R_3$ [S2].
3.  **TTM3:**
    *   **Input:** $U^{(3)}$ and slices of $B''$.
    *   **Output:** Final volume brick $A_f$ of size $B \times B \times B$ [S2].

### Grid Configuration
*   One thread block is assigned per decoded brick [S2].
*   The grid size is determined by the number of bricks to be decoded in the current frame [S2].
*   For TTM2 and TTM3, each block computes only half of a slice to optimize shared memory usage [S2].

### Shared Memory Usage
Shared memory is used to cache factor matrices and tensor slices to avoid bank conflicts [S2]. Data is dequantized to 32-bit floating point words before upload to ensure all accesses are aligned on 32-bit words [S2].

*   **TTM1:** $(B \cdot R \cdot 4 + R \cdot R \cdot 4)$ bytes per block [S2].
*   **TTM2/TTM3:** $(B/2 \cdot R \cdot 4 + B \cdot R \cdot 4)$ bytes per block [S2].
*   **Constraints:** Maximum 16 KB for CUDA 1.x and 48 KB for CUDA 2.x per multiprocessor. Maximum 8 blocks per multiprocessor [S2].

### Formulas
The full reconstruction is defined as the sequential n-mode product:
$$ A_f = B \times_1 U^{(1)} \times_2 U^{(2)} \times_3 U^{(3)} $$
where $B$ is the core tensor and $U^{(n)}$ are the factor matrices for mode $n$ [S2].

### Parameters
*   **Brick Size ($B$):** 32 [S2].
*   **Rank ($R$):** 16 [S2].
*   **Quantization:** 16-bit for factor matrices $U^{(n)}$ and core tensor $B$ [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dequantize(value, min_val, max_val, bits=16):
    """Linear dequantization from 16-bit integer to float [G]."""
    return min_val + (value / (2**bits - 1)) * (max_val - min_val)

def ttm1_kernel(U1, B_slice, B_prime, R1, R2, R3, min_U1, max_U1, min_B, max_B):
    """
    Computes B' = B x_1 U1.
    U1: shape (B, R1)
    B_slice: shape (R1, R2, R3)
    B_prime: shape (B, R2, R3)
    Each thread handles one (i1, r2, r3) output element.
    """
    for i1 in range(B):
        for r2 in range(R2):
            for r3 in range(R3):
                voxel = 0.0
                for r1 in range(R1):
                    u_val = dequantize(U1[i1, r1], min_U1, max_U1)
                    b_val = dequantize(B_slice[r1, r2, r3], min_B, max_B)
                    voxel += u_val * b_val
                B_prime[i1, r2, r3] = voxel

def ttm2_kernel(U2, B_prime, B_double_prime, R2, R3, min_U2, max_U2, min_Bp, max_Bp):
    """
    Computes B'' = B' x_2 U2.
    U2: shape (B, R2)
    B_prime: shape (B, R2, R3)
    B_double_prime: shape (B, B, R3)
    Each thread handles one (i1, i2, r3) output element.
    """
    for i1 in range(B):
        for i2 in range(B):
            for r3 in range(R3):
                voxel = 0.0
                for r2 in range(R2):
                    u_val = dequantize(U2[i2, r2], min_U2, max_U2)
                    bp_val = dequantize(B_prime[i1, r2, r3], min_Bp, max_Bp)
                    voxel += u_val * bp_val
                B_double_prime[i1, i2, r3] = voxel

def ttm3_kernel(U3, B_double_prime, A_f, R3, min_U3, max_U3, min_Bpp, max_Bpp, hot_corner_val):
    """
    Computes A_f = B'' x_3 U3.
    U3: shape (B, R3)
    B_double_prime: shape (B, B, R3)
    A_f: shape (B, B, B)
    Each thread handles one (i1, i2, i3) output element.
    """
    for i1 in range(B):
        for i2 in range(B):
            for i3 in range(B):
                voxel = 0.0
                for r3 in range(R3):
                    u_val = dequantize(U3[i3, r3], min_U3, max_U3)
                    bpp_val = dequantize(B_double_prime[i1, i2, r3], min_Bpp, max_Bpp)
                    voxel += u_val * bpp_val
                # Add contribution of hot-corner core value B(0,0,0)
                voxel += hot_corner_val
                A_f[i1, i2, i3] = voxel
```

### Edge Cases and Pitfalls
*   **Shared Memory Bank Conflicts:** Avoided by dequantizing factor matrices and core tensor to 32-bit floating point words before uploading to shared memory, ensuring all accesses are aligned on 32-bit words [S2].
*   **Hot-Corner Coefficient:** The coefficient $B(0,0,0)$ is handled separately in TTM3. To simplify decoding and avoid special case handling in the inner loop, a zero quantized coefficient is encoded at the corresponding core tensor position, and the unquantized value is added directly [S2].
*   **Memory Limits:** Shared memory usage must fit within the limits of the CUDA architecture (16 KB for 1.x, 48 KB for 2.x). The configuration described works for 32-cubed bricks [S2].

## Operating characteristics

### Time and Memory Complexity
*   **Computational Complexity:** The sequential TTM approach reduces the computational complexity per reconstructed voxel to linear growth with rank $R$, compared to cubic order $R^3$ in naive triple-loop implementations [S2].
*   **Memory Efficiency:** By storing intermediate results ($B'$ and $B''$) in global memory and loading only slices into shared memory, the method is more memory efficient than approaches requiring the complete core tensor in each thread [S2].

### Measured Performance
*   **Hardware:** GeForce GTX 480, Intel Core 2 E8500, 4GB RAM [S2].
*   **Frame Rates:** Interactive rendering frame rates higher than 12Hz in demanding situations, averaging 50-100Hz [S2].
*   **Overhead:** Tensor reconstruction constitutes negligible overhead compared to ray-casting and data transfer [S2]. The most costly part of the reconstruction process is the final device-to-device copy of decoded bricks to the texture cache [S2].
*   **Preprocessing Time:**
    *   2GB chameleon dataset: 25min 15sec, producing a 231M file [S2].
    *   17GB tooth dataset: 8h 45min, producing a 5.5GB file [S2].

### Strengths and Limitations
*   **Strengths:** High performance on GPUs, linear complexity with rank, suitable for interactive rendering of large volumes [S2].
*   **Limitations:** Requires significant preprocessing time for large datasets [S2].

## Background

### Origin and History
The method was introduced in the paper "Interactive Multiscale Tensor Reconstruction for Multiresolution Volume Rendering" [S2]. It builds on the Tucker decomposition framework, specifically addressing the computational bottleneck of reconstructing volume bricks from compressed tensor representations on GPUs [S2].

### Usage
The method is used in multiresolution volume rendering, particularly for large-scale micro-CT datasets such as the veiled chameleon and dental enamel (tooth) scans [S2]. It is integrated into adaptive GPU ray-casters to provide interactive frame rates [S2].

## Sources

[S2] Interactive Multiscale Tensor Reconstruction for Multiresolution Volume Rendering, https://ics.uci.edu/~gopi/SamplePubs/MultiscaleTensorVis.pdf, fetched 2026-10-26
