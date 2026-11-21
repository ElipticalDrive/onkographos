# Tensor-Train Decomposition for 4D Volumes

Also called: TT4D, TT4D Compression, TT4D Decomposition

## Overview
TT4D is a memory- and time-efficient lossy compression technique for four-dimensional (4D) time-varying volume data based on the Tensor-Train (TT) decomposition [S1][S2, unofficial]. It solves the problem of interactive visualization of large-scale, time-varying volumetric data by enabling random-access decoding of individual time steps without reconstructing the entire 4D tensor into memory [S1][S2, unofficial]. It sets itself apart from its siblings, such as [Value compression](6-value-compression.md) and [Blockwise Wavelet Compression](6100-blockwise-wavelet-compression.md), by guaranteeing fixed-rate compression and random-access patterns through linear tensor cores, avoiding variable-bit length encoding or vector quantization [S1][S2, unofficial]. Unlike [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md) which rely on hierarchical subbands, TT4D factorizes the data into a chain of low-order cores, allowing for efficient subsampling during decomposition and on-the-fly GPU-based reconstruction [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D tensor $T$ of size $N_1 \times N_2 \times N_3 \times N_4$, representing spatial dimensions $(N_1, N_2, N_3)$ and time steps $(N_4)$ [S1]. Data types are typically 32-bit floats for decomposition and 16-bit floats for storage [S1].
*   **Output:** A set of TT cores $\{C_1, C_2, C_3, C_4\}$ and associated ranks $\{r_1, r_2, r_3\}$ [S1][S2, unofficial]. The cores are stored as 16-bit floats [S1].

**Data Structures:**
The TT decomposition factorizes $T$ into a sequence of cores [S2, unofficial]. Each core $C_k$ is a tensor of order 3 (for $k=2,3$) or order 2 (for $k=1,4$) [S1][S2, unofficial].
*   $C_1$: Shape $(N_1, r_1)$.
*   $C_k$ ($k=2,3$): Shape $(r_{k-1}, N_k, r_k)$.
*   $C_4$: Shape $(r_3, N_4)$.
The ranks $r_k$ are determined by the target compression ratio [S1]. The total memory is proportional to $\sum N_k r_{k-1} r_k$, growing linearly with dimensionality [S2, unofficial]. The reconstruction formula is $T = \sum_{r_1} \cdots \sum_{r_{d-1}} C_1[:, r_1] \circ C_2[r_1, :, r_2] \circ \cdots \circ C_d[r_{d-1}, :]$ [S1][S2, unofficial].

**Algorithm Steps:**
The decomposition uses Sequential SVD (SSVD) with efficient subsampling to avoid full-scale tensor reshaping and SVD on large imbalanced matrices [S1][S2, unofficial]. The process iterates through modes $k=1$ to $d-1$ (where $d=4$) [S1].
1.  **Matricization:** Reshape the current residual tensor (initially $T$) along mode $k$ into a matrix $A$ of size $N_k \times (N_{k+1} \dots N_d)$ [S1].
2.  **Subsampling:** Select a submatrix $A_{sub}$ by sampling $R_k + p$ columns from $A$ [S1]. Two sampling strategies are used:
    *   *Random Column Sampling:* Selects columns uniformly at random [S1].
    *   *Maximum Volume (MaxVol):* Iteratively selects columns to maximize the volume of the parallelepiped spanned by them [S1][S2, unofficial].
3.  **SVD on Submatrix:** Compute the SVD of the sampled submatrix: $A_{sub} \approx U_{sub} \Sigma_{sub} V_{sub}^T$ [S1]. The left singular vectors $U_{sub}$ approximate the column space of $A$ [S1].
4.  **Projection:** Compute the full matrix $U$ by projecting $A$ onto the subspace spanned by $U_{sub}$: $U = A U_{sub}$ [S1]. This avoids computing SVD on the full large matrix $A$ [S1].
5.  **Orthogonalization:** Orthogonalize $U$ to satisfy $U^T U = I$ (e.g., via QR decomposition) [S1].
6.  **Core Extraction:** Form the factorization $A = U V^T$ [S1]. The core $C_k$ is derived from $U$ (and potentially reshaped from intermediate steps) [S1]. The residual for the next mode is $V^T$ [S1].
7.  **Iteration:** Repeat for $k=1$ to $d-1$ [S1]. The final core $C_d$ is the remaining factor [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tt_decompose_4d(T, target_ranks, sampling_method='maxvol', p=10, max_iter=15):
    """
    T: 4D array of shape (N1, N2, N3, N4)
    target_ranks: list of [r1, r2, r3]
    sampling_method: 'random' or 'maxvol'
    p: oversampling parameter
    max_iter: max iterations for maxvol
    """
    cores = []
    residual = T
    for k in range(1, 4):  # Modes 1, 2, 3
        # 1. Matricization along mode k
        N_k = residual.shape[k-1]
        other_dims = list(residual.shape[:k-1]) + list(residual.shape[k:])
        A = residual.reshape(N_k, -1)

        # 2. Subsampling
        r_k = target_ranks[k-1]
        num_cols = r_k + p
        if num_cols > A.shape[1]:
            num_cols = A.shape[1]

        if sampling_method == 'random':
            col_indices = random_sample_columns(A, num_cols)
        elif sampling_method == 'maxvol':
            # The exact iterative column selection procedure is not detailed in the sources [G]
            col_indices = maxvol_approximation(A, num_cols, max_iter)
        A_sub = A[:, col_indices]

        # 3. SVD on submatrix
        U_sub, _, _ = svd(A_sub, full_matrices=False)

        # 4. Projection
        U = matmul(A, U_sub)

        # 5. Orthogonalization
        U, _ = qr(U)

        # 6. Core extraction & residual update
        # A = U * V^T  =>  V^T = U^T * A
        V_T = matmul(U.T, A)

        # Form core C_k
        prev_rank = 1 if k == 1 else target_ranks[k-2]
        C_k = U.reshape(prev_rank, N_k, r_k) if k > 1 else U.reshape(N_k, r_k)
        cores.append(C_k)

        # Update residual for next mode
        remaining_dims = [r_k] + list(residual.shape[k:])
        residual = V_T.reshape(remaining_dims)

    # Final core C_4
    C_4 = residual.reshape(target_ranks[2], -1)
    cores.append(C_4)
    return cores

def random_sample_columns(A, num_cols):
    """Selects num_cols unique column indices uniformly at random."""
    import random
    return random.sample(range(A.shape[1]), num_cols)

def maxvol_approximation(A, num_cols, max_iter):
    """
    Iteratively selects columns to maximize the volume of the parallelepiped.
    The exact iterative procedure is not detailed in the sources [G].
    """
    # Placeholder for the iterative maxvol algorithm [G]
    return list(range(num_cols))

def svd(M, full_matrices=False):
    """Standard Singular Value Decomposition. Returns U, S, Vt."""
    # Standard linear algebra routine [G]
    pass

def qr(M):
    """Standard QR Decomposition. Returns Q, R."""
    # Standard linear algebra routine [G]
    pass

def matmul(A, B):
    """Standard matrix multiplication."""
    # Standard linear algebra routine [G]
    pass
```

**Parameters:**
*   **Oversampling parameter $p$:** Controls the number of additional columns sampled. Typical value: 10 [S1][S2, unofficial]. Higher $p$ reduces relative error but increases processing time and memory for larger matrices [S1].
*   **MaxVol Iterations:** Maximum number of iterations for the maximum volume approximation. Typical value: 15 [S1][S2, unofficial].
*   **Data Types:** TT cores stored as 16-bit floats. Decomposition done in 32-bit floats [S1].

**Edge Cases and Pitfalls:**
*   **Memory Cost:** Directly accessing all $N_1 N_2 N_3 N_4$ elements to form matrix $A$ is too costly for large datasets [S1]. The subsampling method mitigates this by only sampling $R_1 + p$ columns [S1].
*   **Reconstruction Time:** For large datasets (e.g., turbulence, channel), reconstruction time can exceed 100ms even with partial decoding, necessitating subsampled reconstruction for fluent interaction [S1][S2, unofficial].
*   **ZFP Limitation:** ZFP fixed-rate compression cannot compress more than 1:128 for 32-bit float data, limiting its use for large datasets [S1].

## Subtopics

### [TT Reconstruction for Time-Dependent Rendering](14085-tt-reconstruction-for-time-dependent-rendering.md)
TT Reconstruction is a method to reconstruct a specific 3D volume slice at a given time step $t$ from the Tensor-Train (TT) cores of a 4D dataset for time-dependent volume rendering. It solves the problem of efficiently accessing individual time steps without decompressing the entire 4D tensor into memory. It sets itself apart from naive TT evaluation by pre-multiplying the spatial cores, allowing the reconstruction of any time step via a single tensor-vector multiplication with the time-specific core.

## Operating characteristics
*   **Time Complexity:** The decomposition complexity is reduced from $O(N_1 N_2 N_3 N_4)$ to $O(N_1 (R_1 + p))$ per mode matricization, making it feasible for large 4D volumes [S1]. MaxVol sampling is slower than random sampling due to iterative processing [S1].
*   **Memory Complexity:** Memory consumption is significantly lower than TTHRESH, tntorch, and default SSVD because it performs most operations in-place or on small submatrices [S1]. It avoids creating large intermediate matrices [S2, unofficial].
*   **Measured Performance:**
    *   On a 4D turbulence dataset ($512^3 \times 64$, 32GB), TT4D achieved a compression ratio of 1:290 [S1].
    *   Reconstruction time for full resolution at $t=1$ was 70ms [S1].
    *   Reconstruction time for 1:8 subsampled rendering at $t=21$ was 11ms [S1].
    *   For the channel dataset (64GB) compressed at 1:128, a 1:8 subsampled time slice decoded in 20.8ms on average [S1].
    *   Applying a 1D Difference of Gaussian filter along the time dimension took less than 2ms [S1][S2, unofficial].
*   **Strengths:**
    *   Fixed-rate compression and random access [S1][S2, unofficial].
    *   Low memory usage enables processing of datasets up to 64GB [S1].
    *   Supports interactive filtering in the compressed domain [S1].
    *   Higher accuracy (PSNR/SSIM) than ZFP for most datasets [S1].
*   **Limitations:**
    *   Not designed for maximal compression; TTHRESH achieves higher ratios with similar accuracy [S1].
    *   Reconstruction times for large datasets can exceed 100ms, requiring subsampling for interactivity [S1].
    *   MaxVol sampling requires more memory than random sampling due to auxiliary matrices [S1].

## Background
The method was introduced by C. Hartmann, R. Ballester-Ripoll, and R. Pajarola in the paper 'TT4D' published in Computer Graphics Forum by Eurographics and John Wiley & Sons Ltd [S2, unofficial]. It addresses the challenges of visualizing large-scale, time-varying volumetric data using direct volume rendering, where data sizes and resolutions continue to increase [S1]. The approach builds on the Tensor-Train decomposition [Ose11] and Sequential SVD [S1], optimizing them for memory and time efficiency through subsampling [S1][S2, unofficial].

## Sources
[S1] TT4D: Tensor‐Train‐based 4D Time‐Dependent Volume Rendering, https://doi.org/10.1111/cgf.70471, fetched 2026-10-27
[S2] Tensor-Train-based 4D Time-Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
