# Sparse Representation (Dictionary Based)

Also called: Dictionary based sparse coding

## Overview
Sparse Representation (Dictionary Based) encodes volume data blocks by finding a sparse linear combination of atoms from a pre-computed dictionary, solving an optimization problem to minimize reconstruction error under a sparsity constraint [S1][S2]. It differs from Vector Quantization (VQ) by allowing each input block to be represented by a linear combination of multiple dictionary elements rather than a single nearest-neighbor prototype, offering higher fidelity at the cost of significantly more complex encoding [S1][S2]. It builds on [Sparse Coding (K-SVD)](1018-sparse-coding-k-svd.md).

## How it works
**Inputs and Outputs**
*   **Input:** A volume dataset partitioned into cubic blocks of size $M \times M \times M$ [S1][S2]. A pre-computed dictionary $\mathbf{D} \in \mathbb{R}^{m \times K}$, where $m = M^3$ is the dimension of a flattened block and $K$ is the number of atoms in the dictionary [S1][S2].
*   **Output:** For each block, a sparse coefficient vector $\boldsymbol{\lambda}_i \in \mathbb{R}^K$ containing at most $S$ non-zero entries, stored as the indices of these non-zero entries and their corresponding magnitudes [S1][S2].

**Data Structures**
*   **Dictionary $\mathbf{D}$:** A matrix of shape $(m, K)$, where each column $\mathbf{d}_j$ is an atom of size $m$ [S1][S2].
*   **Sparse Representation $\boldsymbol{\lambda}_i$:** A vector of size $K$, stored efficiently as a list of pairs $(index, value)$ for non-zero entries [S1][S2].

**Algorithm Steps**
1.  **Vectorization:** For each volume block $i$, extract the voxel values and map them to a column vector $\mathbf{y}_i \in \mathbb{R}^m$ [S1][S2].
2.  **Optimization:** Solve for $\boldsymbol{\lambda}_i$ to minimize the reconstruction error subject to a sparsity constraint:
    $$ \min_{\boldsymbol{\lambda}_i} \left\{ \| \mathbf{y}_i - \mathbf{D}\boldsymbol{\lambda}_i \|^2 \right\} \quad \text{subject to} \quad \| \boldsymbol{\lambda}_i \|_0 \leq S $$
    where $\| \boldsymbol{\lambda}_i \|_0$ denotes the number of non-zero entries in $\boldsymbol{\lambda}_i$ [S1][S2].
3.  **Approximation:** Since the exact solution is NP-hard, use a greedy pursuit approximation algorithm [S1][S2]. Specifically, Orthogonal Regularized Matching Pursuit (ORMP) via Cholesky decomposition is employed for its simplicity and efficiency [S1][S2].
4.  **Compression:** Store the sparse representation $\boldsymbol{\lambda}_i$ by recording the indices of the non-zero elements and their corresponding magnitudes [S1][S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_block_dictionary_based(y_i, D, S):
    """
    Encodes a single volume block using sparse representation.
    Args:
        y_i: list of float, flattened volume block of length m.
        D: list of lists of float, dictionary matrix of shape (m, K). D[j] is column j.
        S: int, maximum number of non-zero coefficients.
    Returns:
        indices: list of int, indices of non-zero coefficients.
        values: list of float, magnitudes of non-zero coefficients.
    """
    m = len(y_i)
    K = len(D)
    
    # Initialize coefficients to zero
    lambda_i = [0.0] * K
    selected_indices = []
    residual = list(y_i)
    
    for step in range(S):
        # 1. Select atom most correlated with residual
        correlations = [abs(dot(D[j], residual)) for j in range(K)]
        best_idx = max(range(K), key=lambda j: correlations[j])
        
        if best_idx in selected_indices:
            break
            
        selected_indices.append(best_idx)
        
        # 2. Update coefficients using least-squares on selected atoms
        D_selected = [D[j] for j in selected_indices]
        
        # Solve min || y_i - D_selected * lambda_selected ||^2
        # Using Cholesky decomposition: (D_selected^T D_selected) lambda = D_selected^T y_i
        A = matmul(transpose(D_selected), D_selected)
        b = matmul(transpose(D_selected), y_i)
        
        # Cholesky decomposition of A
        L = cholesky(A)
        
        # Solve L z = b
        z = solve_triangular(L, b)
        
        # Solve L^T lambda_selected = z
        lambda_selected = solve_triangular(transpose(L), z)
        
        # Update residual
        residual = subtract(y_i, matmul(D_selected, lambda_selected))
        
        # Store coefficients in full vector
        for idx, val in zip(selected_indices, lambda_selected):
            lambda_i[idx] = val
            
    # Extract non-zero entries
    indices = [i for i, val in enumerate(lambda_i) if val != 0.0]
    values = [lambda_i[i] for i in indices]
    
    return indices, values

# Helper functions for standard linear algebra operations
def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def transpose(M):
    return [[M[i][j] for i in range(len(M))] for j in range(len(M[0]))]

def matmul(A, B):
    if isinstance(B[0], list):
        return [[sum(A[i][k] * B[k][j] for k in range(len(B))) for j in range(len(B[0]))] for i in range(len(A))]
    else:
        return [sum(A[i][k] * B[k] for k in range(len(B))) for i in range(len(A))]

def subtract(a, b):
    return [x - y for x, y in zip(a, b)]

def cholesky(A):
    n = len(A)
    L = [[0.0] * n for _ in range(n)]
    for i in range(n):
        for j in range(i + 1):
            s = sum(L[i][k] * L[j][k] for k in range(j))
            if i == j:
                L[i][j] = (A[i][i] - s) ** 0.5
            else:
                L[i][j] = (A[i][j] - s) / L[j][j]
    return L

def solve_triangular(L, b):
    n = len(b)
    x = [0.0] * n
    for i in range(n):
        x[i] = (b[i] - sum(L[i][j] * x[j] for j in range(i))) / L[i][i]
    return x
```

**Parameters**
*   $M$: Block side length. Not given in the sources.
*   $K$: Dictionary size. Not given in the sources.
*   $S$: Sparsity level (max non-zero entries). Not given in the sources.

**Edge Cases and Pitfalls**
*   **NP-Hardness:** The optimization problem is NP-hard; greedy approximations do not guarantee global optimality [S1][S2].
*   **Precision:** Floating-point errors in Cholesky decomposition may require regularization if the matrix is ill-conditioned [G].
*   **Boundaries:** Blocks at the volume boundary must be handled (e.g., padding or smaller blocks). Not given in the sources.

## Subtopics

### [Sparse Representation Modeling](5392-sparse-representation-modeling.md)
Represents a signal as a sparse linear combination of atoms from an overcomplete dictionary matrix. It solves the problem of compactly encoding volumetric data by allowing each input signal to be represented by a linear combination of dictionary elements rather than a single codeword, generalizing Vector Quantization (VQ). Unlike VQ, which is limited by a hard limit on achievable quality and compression rate, this method allows arbitrary coefficient values and multiple nonzero entries, offering increased flexibility and adaptation to specific signal data.

### [Sparse Coding](5842-sparse-coding.md)
Decompresses volume data by reconstructing voxel blocks as linear combinations of codewords from a dictionary. It distinguishes itself from its parent method [Sparse Representation (Dictionary Based)](5366-sparse-representation-dictionary-based.md) by explicitly utilizing a linear combination of multiple codewords rather than a single best match, and from its sibling [Sparse Representation Modeling](5392-sparse-representation-modeling.md) by focusing on the decomposition and reconstruction mechanics of voxel blocks rather than statistical coefficient modeling [G].

## Operating characteristics
*   **Complexity:** Encoding is computationally expensive due to the need for iterative greedy pursuit algorithms (e.g., ORMP) and matrix decompositions (Cholesky), unlike the trivial nearest-neighbor search in VQ [S1][S2].
*   **Memory:** Compression is achieved by storing only the indices and magnitudes of the sparse coefficients, reducing memory occupancy compared to storing full block data [S1][S2].
*   **Performance:** Not given in the sources.
*   **Strengths:** Higher reconstruction fidelity than VQ by using linear combinations of multiple atoms [S1][S2].
*   **Limitations:** High encoding complexity; NP-hard optimization requires approximations [S1][S2].

## Background
Sparse representation for volume data compression was introduced in the context of compressed GPU-based direct volume rendering [S1][S2]. The method was notably employed by Gobbetti et al. [GIM12] using ORMP via Cholesky decomposition [CARKD99] [S1][S2]. It builds on the general concept of sparse coding in signal processing [Ela10] [S1][S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
