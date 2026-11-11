# Adaptive Rank Selection via Binary Search

Also called: Binary Search for Best Rank, Optimal Rank Selection

## Overview
Adaptive Rank Selection via Binary Search is a method to determine the optimal Tucker decomposition rank $(R_1, R_2, R_3)$ for individual data blocks in multi-resolution volume rendering [S1]. It solves the trade-off between compression ratio and reconstruction error (measured by Peak Signal-to-Noise Ratio, PSNR) by finding the smallest rank that satisfies a specific error threshold for each block [S1]. Unlike uniform rank decomposition, which applies the same rank to all blocks (leading to either quality loss or inefficient compression), this method assigns different ranks to different blocks based on their local complexity [S1]. It builds on [Truncated Tensor Decomposition](7540-truncated-tensor-decomposition.md) by using a pre-computed high-rank decomposition to efficiently search for the best truncation point without recomputing the full HOSVD for every trial rank [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume data block of size $I_1 \times I_2 \times I_3$. A pre-computed high-rank Tucker decomposition consisting of factor matrices $U^{(1)} \in \mathbb{R}^{I_1 \times R_{max,1}}$, $U^{(2)} \in \mathbb{R}^{I_2 \times R_{max,2}}$, $U^{(3)} \in \mathbb{R}^{I_3 \times R_{max,3}}$ and core tensor $\mathcal{B} \in \mathbb{R}^{R_{max,1} \times R_{max,2} \times R_{max,3}}$ [S1]. An error threshold $T_e$ (and optionally $T_p$) [S1].
*   **Output:** The optimal rank tuple $(R_1, R_2, R_3)$ for the block [S1].

**Data Structures:**
The method operates on the factor matrices and core tensor stored in memory [S1]. The factor matrices $U^{(n)}$ are dense matrices where column $r$ represents the $r$-th basis vector in dimension $n$ [S1]. The core tensor $\mathcal{B}$ is a 3D array [S1]. No special bit-packing is described for the search process itself; the search relies on the dense representation of the truncated components [G].

**Parameters:**
*   **Initial Search Range:** The binary search is performed between a lower bound $R_s$ and an upper bound $R_e$ [S1]. For a block size of 32, typical values are $R_s = 4$ and $R_e = 16$ [S1].
*   **Error Threshold ($T_e$):** The threshold used to determine if the rank is sufficient [S1]. In the cited experiments, $T_e = T_p = 0.25$ was used, with other tests ranging from 0.1 to 0.4 [S1]. The threshold is applied to a normalized error metric derived from the reconstruction [S1]. The source defines PSNR in Eq (2)-(3) but uses $T_e$ as a threshold for the error in the binary search logic, implying a normalized error metric rather than raw PSNR dB value, as PSNR is maximized while error is minimized [S1].

**Algorithm Steps:**
1.  **Pre-compute High-Rank Decomposition:** Perform a full Tucker decomposition (HOSVD) on the data block to obtain the high-rank factor matrices $U^{(1)}, U^{(2)}, U^{(3)}$ and core tensor $\mathcal{B}$ with ranks $(R_{max,1}, R_{max,2}, R_{max,3})$ [S1].
2.  **Initialize Search Bounds:** Set the lower bound $R_s$ and upper bound $R_e$ [S1]. Set the current rank $R = \lfloor (R_s + R_e) / 2 \rfloor$ using integer division [S1].
3.  **Perform Rank Truncation:** Truncate the pre-computed factor matrices and core tensor to the current rank $R$ [S1]. Let $U_k^{(n)}$ be the first $R$ columns of $U^{(n)}$ for $n=1,2,3$ [S1]. Let $\mathcal{B}_k$ be the sub-tensor of $\mathcal{B}$ with dimensions $R \times R \times R$ [S1]. This step avoids recomputing the SVD for every trial rank [S1].
4.  **Evaluate Error:** Reconstruct the data approximation $\hat{\mathcal{X}}$ from the truncated components using the Tucker product: $\hat{\mathcal{X}} = \mathcal{B}_k \times_1 U_k^{(1)} \times_2 U_k^{(2)} \times_3 U_k^{(3)}$ [S1]. Calculate the reconstruction error using the normalized metric [S1].
5.  **Check Convergence (Lower Bound):** If the current rank $R$ satisfies the error threshold and $R - R_s < 2$, output $R$ as the optimal rank and terminate [S1]. Otherwise, update the upper bound $R_e = R$, update the current rank $R = \lfloor (R_s + R_e) / 2 \rfloor$, and return to Step 3 [S1].
6.  **Check Convergence (Upper Bound):** If the current rank $R$ does not satisfy the error threshold and $R_e - R < 2$, output $R + 1$ as the optimal rank and terminate [S1]. Otherwise, update the lower bound $R_s = R$, update the current rank $R = \lfloor (R_s + R_e) / 2 \rfloor$, and return to Step 3 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hosvd(data):
    """Computes the Higher-Order Singular Value Decomposition (HOSVD) of a 3D tensor.
    Returns factor matrices U1, U2, U3 and core tensor B. Standard operation [G]."""
    # Implementation of HOSVD via mode-n SVDs is standard [G].
    # For brevity, we assume a standard routine provides this.
    pass # Standard HOSVD routine [G]

def reconstruct_tucker(B, U1, U2, U3):
    """Reconstructs the tensor approximation from core B and factor matrices U1, U2, U3.
    X = B x1 U1 x2 U2 x3 U3. Standard tensor-tucker product [G]."""
    # Mode-1 product: X_temp[i1, r2, r3] = sum_{r1} B[r1, r2, r3] * U1[i1, r1]
    X_temp = [[[sum(B[r1][r2][r3] * U1[i1][r1] for r1 in range(len(U1[0]))) for r3 in range(len(B[0][0]))] for r2 in range(len(B[0]))] for i1 in range(len(U1))]
    # Mode-2 product: X_temp2[i1, i2, r3] = sum_{r2} X_temp[i1, r2, r3] * U2[i2][r2]
    X_temp2 = [[[sum(X_temp[i1][r2][r3] * U2[i2][r2] for r2 in range(len(U2[0]))) for r3 in range(len(X_temp[0][0]))] for i2 in range(len(U2))] for i1 in range(len(X_temp))]
    # Mode-3 product: X_hat[i1, i2, i3] = sum_{r3} X_temp2[i1][i2][r3] * U3[i3][r3]
    X_hat = [[[sum(X_temp2[i1][i2][r3] * U3[i3][r3] for r3 in range(len(U3[0]))) for i3 in range(len(U3))] for i2 in range(len(U2))] for i1 in range(len(U1))]
    return X_hat

def calculate_normalized_error(X_hat, X_original):
    """Calculates normalized mean squared error. Source defines PSNR, but binary search uses a normalized error threshold [S1]."""
    n = len(X_original) * len(X_original[0]) * len(X_original[0][0])
    mse = sum((X_hat[i][j][k] - X_original[i][j][k]) ** 2 for i in range(len(X_original)) for j in range(len(X_original[0])) for k in range(len(X_original[0][0]))) / n
    data_range = max(max(max(row) for row in plane) for plane in X_original) - min(min(min(row) for row in plane) for plane in X_original)
    if data_range == 0:
        return 0.0
    return mse / (data_range ** 2)

def adaptive_rank_selection(block_data, threshold_te):
    """Finds the optimal Tucker rank for a data block using binary search [S1]."""
    # Step 1: Pre-compute high-rank decomposition [S1]
    U1, U2, U3, B = compute_hosvd(block_data)
    R_max = len(U1[0]) # Assume isotropic max rank for simplicity

    # Step 2: Initialize search bounds [S1]
    Rs = 4
    Re = 16
    R = (Rs + Re) // 2

    while True:
        # Step 3: Perform Rank Truncation [S1]
        U1_trunc = [row[:R] for row in U1]
        U2_trunc = [row[:R] for row in U2]
        U3_trunc = [row[:R] for row in U3]
        B_trunc = [[[B[r1][r2][r3] for r3 in range(R)] for r2 in range(R)] for r1 in range(R)]

        # Reconstruct approximation
        X_hat = reconstruct_tucker(B_trunc, U1_trunc, U2_trunc, U3_trunc)

        # Step 4: Evaluate Error [S1]
        error = calculate_normalized_error(X_hat, block_data)

        # Step 5: Check Convergence (Lower Bound) [S1]
        if error <= threshold_te:
            if R - Rs < 2:
                return R
            Re = R
            R = (Rs + Re) // 2
        # Step 6: Check Convergence (Upper Bound) [S1]
        else:
            if Re - R < 2:
                return R + 1
            Rs = R
            R = (Rs + Re) // 2
```

**Edge Cases and Pitfalls:**
*   **Precision:** The reconstruction involves multiple matrix multiplications, and floating-point precision errors may accumulate near the threshold boundary [G].
*   **Boundary Conditions:** The convergence checks ($R - R_s < 2$ and $R_e - R < 2$) ensure termination [S1].
*   **Degenerate Input:** If the data block is constant or zero, the optimal rank might be lower than the search lower bound, but the method is designed for massive data blocks where complexity varies [S1].

## Subtopics

### Uniform Rank Decomposition
Applies the same rank $(R, R, R)$ to all blocks. Simple to implement but inefficient for heterogeneous data. [Link](7540-truncated-tensor-decomposition.md) discusses the general concept of rank reduction.

### Information Entropy Based Selection
Uses information entropy to determine block resolution. Adaptive but may not directly optimize for PSNR [S1].

### CIELUV Error Criterion
Uses color space error for texture data. Specific to color volumes [S1].

## Operating characteristics
*   **Time Complexity:** The preprocessing time is higher than uniform high-rank decomposition. For a threshold of 0.25, the adaptive method took 32.7 minutes compared to 21.5 minutes for uniform rank (16,16,16) [S1]. This is because it performs multiple reconstructions and error checks per block, although it avoids recomputing the HOSVD [S1].
*   **Memory Complexity:** The memory usage is dominated by the storage of the high-rank factor matrices and core tensor for all blocks. The adaptive method does not increase memory overhead compared to uniform high-rank decomposition, as it stores the same high-rank components initially [S1].
*   **Compression Ratio:** The adaptive method achieves a significantly better compression ratio than uniform high-rank decomposition. At threshold 0.25, it achieved 6.7% compression, compared to 17.19% for uniform rank (16,16,16) [S1]. It is less compressed than uniform low-rank (4,4,4) which is 1.37%, but maintains higher quality [S1].
*   **Quality:** The rendering quality (PSNR) is comparable to or better than uniform high-rank decomposition. For threshold 0.25, the adaptive method's PSNR is not explicitly given as a single number, but the text states the result is "not inferior" to uniform (16,16,16) and "significantly better" than uniform (8,8,8) and (4,4,4) [S1].
*   **Strengths:** Adapts to local data complexity, optimizing the balance between compression and quality [S1].
*   **Limitations:** Higher preprocessing time than uniform high-rank decomposition. The search range and threshold must be tuned for the specific dataset [S1].

## Background
The method was proposed by Xiaoyan Nie and Cai Lu in 2018 in the paper "An Algorithm for Choosing Optimal Rank in Tensor Approximation of Multi-Resolution Volume Rendering" [S1]. It addresses the bottleneck of GPU memory storage in multi-resolution volume rendering by optimizing the rank selection for each block [S1]. Previous methods included uniform rank decomposition and entropy-based selection [S1]. The paper highlights that uniform rank decomposition leads to either excessive error or insufficient compression, motivating the adaptive approach [S1].

## Sources
[S1] An Algorithm for Choosing Optimal Rank in Tensor Approximation of Multi-Resolution Volume Rendering, https://doi.org/10.12677/csa.2018.811183, fetched 2026-10-26
