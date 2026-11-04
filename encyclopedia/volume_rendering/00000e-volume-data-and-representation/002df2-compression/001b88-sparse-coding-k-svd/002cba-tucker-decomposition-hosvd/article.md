# Tucker Decomposition (HOSVD)

Also called: Higher-Order Singular Value Decomposition, Tucker Model

## Overview
Tucker Decomposition (HOSVD) is a global basis generation method that decomposes an $N$-dimensional input array into $N$ basis matrices and an $N$-dimensional core tensor [S1][S2, unofficial]. It solves the problem of storing and transmitting large volume datasets by reducing the reconstruction rank, thereby achieving data reduction [S1]. Unlike its sibling [CP Decomposition](2255-cp-decomposition.md), which sums rank-one tensors, Tucker Decomposition uses a multilinear product structure, allowing for a more compact representation of correlated data [S2, unofficial]. It is distinguished from its parent [Sparse Coding (K-SVD)](1018-sparse-coding-k-svd.md) by its specific application to global multiresolution basis generation for volume rendering, often integrated with octree hierarchies [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An $N$-dimensional input dataset $A \in \mathbb{R}^{I_1 \times I_2 \times \dots \times I_N}$ [S2, unofficial].
*   **Outputs:** $N$ basis matrices $U^{(n)} \in \mathbb{R}^{I_n \times R_n}$ for $n=1 \dots N$, and an $N$-dimensional core tensor $B \in \mathbb{R}^{R_1 \times R_2 \times \dots \times R_N}$ [S2, unofficial].

**Data Structures:**
*   **Basis Matrices ($U^{(n)}$):** Each matrix has $I_n$ rows (corresponding to the spatial resolution in dimension $n$) and $R_n$ columns (the rank). The rows are orthonormal [S1][S2, unofficial].
*   **Core Tensor ($B$):** An $N$-dimensional array of size $R_1 \times R_2 \times \dots \times R_N$. It is "all-orthogonal," meaning the inner product of any two slices along the same mode is zero [S1][S2, unofficial].
*   **Multilinear Rank:** Defined as $(R_1, R_2, \dots, R_N)$, where $R_n$ is the rank for dimension $n$ [S2, unofficial].

**Parameters:**
*   **Initial Rank ($R_n$):** Typically chosen as half the input data size in that dimension, i.e., $R_n = I_n / 2$ [S1][S2, unofficial]. This corresponds to an initial data reduction.
*   **Block Rank (for multiresolution):** When applied to equally sized blocks of size $B$, the multilinear rank is often reduced to $R_1 = R_2 = R_3 = B/2$ [S1][S2, unofficial].
*   **Quantization:** 8-bit encoding for core tensor coefficients and 16-bit encoding for basis matrices [S2, unofficial].

**Steps:**
1.  **Generate Basis Matrices:** Apply Higher-Order Singular Value Decomposition (HOSVD) or Higher-Order Orthogonal Iteration (HOOI) to the input volume $A$ along every data direction [S1][S2, unofficial]. HOOI is an Alternating Least Squares (ALS) algorithm that optimizes the basis matrices one after another [S2, unofficial].
2.  **Compute Core Tensor:** Once the basis matrices $U^{(n)}$ cannot be improved further, compute the core tensor $B$. $B$ represents the projection of $A$ onto the basis matrices $U^{(n)}$ [S1][S2, unofficial]. The coefficients of $B$ show the relationship between the original data and the bases [S2, unofficial].
3.  **Rank Truncation:** Apply rank-truncation to the core tensor $B$. Because $B$ is all-orthogonal, this process is similar to matrix SVD truncation, exploiting the singular values embedded in the diagonal slices of the core tensor [S1][S2, unofficial]. Further adaptive rank reductions can be applied after the initial decomposition [S2, unofficial].
4.  **Multiresolution Basis Generation (Optional/Contextual):** For multiresolution rendering, generate lower-resolution basis matrices $U^{(n)}_{\downarrow k}$ by employing mipmap reduction on the row space of $U^{(n)}$ [S1].
5.  **Re-orthogonalization:** Perform re-orthogonalization to match basis matrix row blocks to multiresolution volume octree nodes [S1].
6.  **Octree Construction:** Define a multiresolution volume octree with one small core tensor per node. Each node's core tensor is constructed by projecting the corresponding input volume brick onto the corresponding basis matrix rows [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tucker_decomposition_hosvd(A, initial_ranks=None):
    """
    Performs Tucker Decomposition using HOSVD/HOOI.
    
    Args:
        A: Input N-dimensional tensor of shape (I1, I2, ..., IN).
        initial_ranks: List of ranks (R1, R2, ..., RN). If None, defaults to In/2.
        
    Returns:
        U: List of N basis matrices.
        B: Core tensor.
    """
    N = A.ndim
    I = A.shape
    
    if initial_ranks is None:
        initial_ranks = [I[n] // 2 for n in range(N)]
    
    # Step 1: Generate Basis Matrices using HOOI
    # HOOI optimizes basis matrices one after another via ALS
    U = []
    for n in range(N):
        # Mode-n unfolding of A
        A_unfolded = mode_n_unfold(A, n)
        
        # Compute initial basis from SVD of unfolded matrix
        # U_n_init, S_n, V_n = svd(A_unfolded)
        # Select top R_n columns
        R_n = initial_ranks[n]
        U_n, _, _ = truncated_svd(A_unfolded, k=R_n)
        
        # Optimize U_n using ALS (HOOI)
        # Iterate until convergence:
        #   For each mode m != n:
        #     Compute projection of A onto other bases
        #     Update U_n via SVD of the projected matrix
        # [G] Standard HOOI implementation details omitted for brevity
        #     but must ensure U_n remains orthonormal.
        U_n = hooi_optimize_mode_n(A, U, n)
        U.append(U_n)
        
    # Step 2: Compute Core Tensor B
    # B is the projection of A onto the basis matrices U
    # B = A x1 U1^T x2 U2^T ... xN UN^T
    B = A
    for n in range(N):
        # Tensor-times-matrix product along mode n
        # B = B x_n (U[n]^T)
        B = mode_n_tensor_matrix_product(B, U[n].T, n)
        
    # Step 3: Rank Truncation (Optional)
    # Exploit all-orthogonal property of B
    # [G] In practice, compute singular values of core tensor slices
    #     and truncate dimensions where singular values are small.
    # B = truncate_core_tensor(B, target_ranks)
    
    return U, B

def mode_n_unfold(A, n):
    """Unfolds tensor A into a matrix along mode n."""
    # Reshape A such that mode n is the row dimension
    # and all other modes are combined into the column dimension
    pass

def truncated_svd(M, k):
    """Returns the top k left singular vectors of M."""
    U, S, Vt = np.linalg.svd(M, full_matrices=False)
    return U[:, :k], S[:k], Vt[:k, :]

def hooi_optimize_mode_n(A, U, n):
    """
    Optimizes the n-th basis matrix U[n] using Alternating Least Squares.
    Assumes other U[m] are fixed.
    """
    # Compute the product of all other basis matrices
    # This effectively projects A into the subspace defined by other bases
    # [G] Implementation involves tensor contractions or unfolding
    #     followed by SVD to update U[n].
    pass

def mode_n_tensor_matrix_product(T, M, n):
    """
    Computes the tensor-matrix product T x_n M.
    """
    # Reshape T, multiply M along mode n, reshape back
    pass
```

**Edge Cases and Pitfalls:**
*   **Memory Management:** Large volumes require memory-mapped files and parallel memory-aligned matrix-matrix multiplications to handle the computational load [S1].
*   **Coefficient Ordering:** Unlike matrix SVD, the ordering of coefficients in the core tensor $B$ is not strictly decreasing [S1][S2, unofficial]. However, progressive tensor rank truncation still works well in practice for adaptive multiscale reconstruction [S1][S2, unofficial].
*   **Global vs. Local:** The global basis approach keeps storage costs low by maintaining only one set of basis matrices, independent of the initial data resolution $I_n$ [S1].

## Subtopics

### [Tensor Approximation](5362-tensor-approximation.md)
Tensor Approximation (TA) is a volume data compression method that interprets a volume dataset as a multi-dimensional tensor and factorizes it into a set of basis matrices and a core tensor. It solves the problem of storing and transmitting large volume datasets by reducing the reconstruction rank, thereby achieving data reduction. Unlike wavelet transforms, rank-reduced TA captures non-axis aligned features at different scales more effectively using the same number of coefficients. It is distinguished from its parent [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md) by its specific integration into a multiresolution octree hierarchy with per-brick Tucker formulation and optimized GPU reconstruction.

## Operating characteristics
*   **Time Complexity:** The decomposition involves SVDs on unfolded matrices and ALS iterations. The complexity is dominated by the SVDs of the mode-n unfoldings, which are $O(I_n \cdot \prod_{m \neq n} I_m \cdot R_n)$ per mode, plus the iterative convergence of HOOI [S1][S2, unofficial].
*   **Memory Complexity:** Requires storing $N$ basis matrices of size $I_n \times R_n$ and a core tensor of size $\prod R_n$. With $R_n = I_n/2$, the storage is significantly reduced compared to the original data $I_1 \times \dots \times I_N$ [S1][S2, unofficial].
*   **Measured Performance:** Achieves good approximation quality and data reduction levels for volume datasets [S2, unofficial].
*   **Strengths:** Global bases capture non-axis aligned features effectively [S1]. The all-orthogonal core tensor allows for efficient rank truncation [S1][S2, unofficial]. Multiresolution generation via mipmap reduction on basis rows is efficient [S1].
*   **Limitations:** The coefficient ordering in the core tensor is not strictly decreasing, requiring careful truncation strategies [S1][S2, unofficial]. Large volumes pose memory challenges [S1].
*   **When to Use:** When global basis generation is desired for multiresolution volume rendering and data reduction is critical [S1][S2, unofficial].
*   **When Not to Use:** When local adaptivity is more important than global basis efficiency, or when strict singular value ordering is required for truncation guarantees.

## Background
Tucker Decomposition, also known as the Tucker Model, was introduced as a generalization of matrix SVD to higher-order tensors. It is also known as Higher-Order PCA [S2, unofficial]. The method was applied to volume rendering by Suter et al. [SMP13] and [SIM*11], who exploited the Tucker model for global multiresolution basis generation [S1][S2, unofficial]. This approach allows for the creation of a multiresolution volume octree where each node contains a small core tensor projected from the global basis [S1].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27 (unofficial)
