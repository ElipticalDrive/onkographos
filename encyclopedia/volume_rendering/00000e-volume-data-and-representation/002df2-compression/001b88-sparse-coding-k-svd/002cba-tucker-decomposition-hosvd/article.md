# Tucker Decomposition (HOSVD)

Also called: Higher-Order SVD, Higher-Order Singular Value Decomposition, Multilinear SVD, Tucker Model

## Overview
Tucker Decomposition (specifically via Higher-Order SVD) is a tensor factorization method that approximates an N-dimensional input array by decomposing it into a core tensor and a set of orthogonal factor matrices, one for each mode [S1, unofficial][S2][S3]. It solves the problem of representing high-dimensional data with a compact multilinear structure, serving as the foundational basis for Tensor Approximation (TA) in volume rendering [S2][S3]. Unlike CP Decomposition, which sums rank-one tensors, HOSVD produces a core tensor that retains the multi-linear structure of the data, allowing for efficient rank-truncation due to the core's all-orthogonal property [S1, unofficial][S2].

## How it works
**Inputs:**
*   Input tensor $\mathcal{A} \in \mathbb{R}^{I_1 \times I_2 \times \dots \times I_N}$, representing the volume data or a block thereof.
*   Target multilinear rank $(R_1, R_2, \dots, R_N)$, where $R_n \le I_n$. Typical initial values are $R_n = I_n / 2$ [S2][S3].

**Outputs:**
*   Factor matrices $\mathbf{U}^{(n)} \in \mathbb{R}^{I_n \times R_n}$ for each mode $n=1,\dots,N$. Columns are orthonormal.
*   Core tensor $\mathcal{B} \in \mathbb{R}^{R_1 \times R_2 \times \dots \times R_N}$.

**Data Structures:**
*   **Tensor $\mathcal{A}$**: Stored as an N-dimensional array. For computation, it is "unfolded" (matricized) into matrices $\mathbf{A}^{(n)}$ of size $I_n \times (I_1 \dots I_{n-1} I_{n+1} \dots I_N)$ [S1, unofficial].
*   **Factor Matrix $\mathbf{U}^{(n)}$**: A 2D matrix where column $k$ is the $k$-th basis vector for mode $n$.
*   **Core Tensor $\mathcal{B}$**: An N-dimensional array of coefficients.

**Algorithm Steps:**

1.  **Unfold and Decompose:** For each mode $n$ from $1$ to $N$:
    a. Unfold $\mathcal{A}$ into matrix $\mathbf{A}^{(n)}$ [S1, unofficial].
    b. Compute the Singular Value Decomposition (SVD) of $\mathbf{A}^{(n)}$:
       $$ \mathbf{A}^{(n)} = \mathbf{U}^{(n)} \mathbf{\Sigma}^{(n)} \mathbf{V}^{(n)T} $$
       where $\mathbf{U}^{(n)}$ contains the left singular vectors, $\mathbf{\Sigma}^{(n)}$ is diagonal, and $\mathbf{V}^{(n)}$ contains the right singular vectors [S1, unofficial].
    c. Select the $R_n$ leading left singular vectors from $\mathbf{U}^{(n)}$ to form the basis matrix for mode $n$. Let this truncated matrix be denoted $\mathbf{U}^{(n)}_{trunc} \in \mathbb{R}^{I_n \times R_n}$ [S1, unofficial].

2.  **Compute Core Tensor:** Calculate the core tensor $\mathcal{B}$ by projecting $\mathcal{A}$ onto the transposes of the computed factor matrices [S1, unofficial][S2]:
    $$ \mathcal{B} = \mathcal{A} \times_1 \mathbf{U}^{(1)T} \times_2 \mathbf{U}^{(2)T} \times_3 \cdots \times_N \mathbf{U}^{(N)T} $$
    In element-wise form:
    $$ b_{r_1 r_2 \dots r_N} = \sum_{i_1=1}^{I_1} \sum_{i_2=1}^{I_2} \dots \sum_{i_N=1}^{I_N} a_{i_1 i_2 \dots i_N} \cdot u_{i_1, r_1}^{(1)} \cdot u_{i_2, r_2}^{(2)} \cdots u_{i_N, r_N}^{(N)} $$
    [S1, unofficial].

3.  **Reconstruction (Verification):** The approximation $\tilde{\mathcal{A}}$ is reconstructed via the n-mode product:
    $$ \tilde{\mathcal{A}} = \mathcal{B} \times_1 \mathbf{U}^{(1)} \times_2 \mathbf{U}^{(2)} \times_3 \cdots \times_N \mathbf{U}^{(N)} $$
    [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hosvd(tensor_A, ranks):
    """
    Computes the Higher-Order SVD of tensor A.
    
    Args:
        tensor_A: numpy array of shape (I1, I2, ..., IN)
        ranks: list of integers [R1, R2, ..., RN]
        
    Returns:
        core_B: numpy array of shape (R1, R2, ..., RN)
        U_matrices: list of numpy arrays, where U_matrices[n] has shape (In, Rn)
    """
    N = tensor_A.ndim
    U_matrices = []
    
    # Step 1: Compute factor matrices for each mode
    for n in range(N):
        # Unfold tensor A along mode n
        # Shape of A_folded: (In, product of all other dimensions)
        A_folded = numpy.moveaxis(tensor_A, n, 0).reshape(
            tensor_A.shape[n], -1
        )
        
        # Compute SVD
        # We only need the left singular vectors (U)
        # numpy.linalg.svd returns U, S, Vh
        U, S, Vh = numpy.linalg.svd(A_folded, full_matrices=False)
        
        # Truncate to the desired rank Rn
        Rn = ranks[n]
        U_truncated = U[:, :Rn]
        
        U_matrices.append(U_truncated)
        
    # Step 2: Compute the core tensor
    # Core B = A x1 U1^T x2 U2^T ... xN UN^T
    # We apply the transpose of each U matrix sequentially using n-mode product
    core_B = tensor_A
    
    for n in range(N):
        # n-mode product: core_B = core_B x_n U_n^T
        # In numpy, this is equivalent to contracting the n-th dimension of core_B
        # with the columns of U_n.
        # core_B shape: (..., R_n, ...) -> (..., R_n, ...)
        # U_n^T shape: (R_n, I_n)
        
        # To perform tensor-matrix multiplication efficiently:
        # Move mode n to the front
        core_B_moved = numpy.moveaxis(core_B, n, 0)
        # Reshape to 2D: (R_n, product of other dims)
        core_B_2d = core_B_moved.reshape(core_B.shape[n], -1)
        
        # Multiply by U_n^T: (R_n, I_n) @ (I_n, other_dims) -> (R_n, other_dims)
        # Note: U_matrices[n] is (I_n, R_n), so we use it directly as the right multiplier
        # if we view the operation as contracting the I_n dimension.
        # Actually, standard n-mode product definition:
        # B = A x_n U means B_{... r_n ...} = sum_{i_n} A_{... i_n ...} U_{i_n, r_n}
        # Here we want U^T, so we use U directly in the contraction if we align indices correctly.
        # Let's use the explicit formula for clarity and correctness:
        # B_r1...rn = sum_i A_i * U_i,r
        # This is a contraction of the n-th index of A with the second index of U.
        
        # Efficient implementation using einsum or tensordot
        # core_B = numpy.tensordot(core_B, U_matrices[n], axes=([n], [0]))
        # This results in shape where the n-th dim of core_B is replaced by Rn.
        
        core_B = numpy.tensordot(core_B, U_matrices[n], axes=([n], [0]))
        
    return core_B, U_matrices
```

**Edge Cases and Pitfalls:**
*   **Non-Uniqueness:** The Tucker decomposition is not unique; core tensor rotations (simultaneous modification of $\mathcal{B}$ and $\mathbf{U}$) do not affect the model fit [S1, unofficial].
*   **All-Orthogonality:** The core tensor produced by HOSVD is all-orthogonal, which facilitates rank-truncation [S2][S3].
*   **Memory:** Large volumes require memory-mapped files and parallel memory-aligned matrix-matrix multiplications during the unfolding and SVD steps [S2].

## Subtopics

### [Tensor Approximation](5362-tensor-approximation.md)
Interprets volume data as a multi-dimensional tensor and factorizes it into a core tensor and basis matrices. Unlike standard LOD, it removes small-scale features while highlighting structural patterns. It builds on HOSVD by applying rank-truncation to learn rank-reducible bases.

### [Tensor-Specific Quantization](6775-tensor-specific-quantization.md)
A fixed bit-length encoding strategy for Tucker coefficients. It uses uniform linear quantization for factor matrices and logarithmic quantization for the core tensor to reduce bandwidth.

### Truncated Tensor Decomposition
A rank-reduced approximation where the multilinear rank $(R_1, \dots, R_N)$ is explicitly chosen smaller than the input dimensions $(I_1, \dots, I_N)$ to achieve compression [S1, unofficial]. Unlike CP decomposition, which cannot be truncated ex post, Tucker decomposition allows progressive truncation due to the all-orthogonality of the core tensor [S1, unofficial]. This can be done by either starting with the desired lower rank or subsequently truncating the full decomposition [S1, unofficial]. The goal is to minimize the least-squares error $\|\mathcal{A} - \tilde{\mathcal{A}}\|^2$ by eliminating principal components with negligible coefficients [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the SVD computations on the unfolded matrices. For an N-dimensional tensor of size $I^N$ and rank $R$, computing the SVD for each mode takes $O(N \cdot I^N \cdot R^2)$ or similar depending on implementation [S1, unofficial][S2].
*   **Memory Complexity:** Requires storing the input tensor, the unfolded matrices, and the resulting factor matrices and core tensor. The core tensor size is $\prod R_n$, and factor matrices are $\sum I_n R_n$. If $R_n \ll I_n$, this is significantly smaller than the original data [S1, unofficial].
*   **Performance:** In volume rendering, using global bases with multiresolution octrees allows reducing the multilinear rank to half the block size ($R_n = B/2$), keeping storage costs low independent of initial resolution [S2].
*   **Strengths:** Captures multi-linear structure; allows efficient rank-truncation; global bases reduce storage compared to per-block bases [S2][S3].
*   **Limitations:** Not unique; requires careful handling of large data via memory mapping; quantization artifacts if not optimized [S1, unofficial][S2][S3].

## Background
The Higher-Order SVD (HOSVD), also known as multilinear SVD, was introduced by De Lathauwer, De Moor, and Vandewalle in 2000 [S1, unofficial]. It is a higher-order generalization of the matrix SVD/PCA. In the context of volume rendering, it was adapted for Tensor Approximation (TA) by Suter et al. (2013) to create global multiresolution bases for direct volume rendering [S2][S3].

## Sources
[S1] Interactive Multiresolution and Multiscale Visualization of Large Volume Data, https://doi.org/10.2312/diss.20138322, fetched 2026-10-26 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
