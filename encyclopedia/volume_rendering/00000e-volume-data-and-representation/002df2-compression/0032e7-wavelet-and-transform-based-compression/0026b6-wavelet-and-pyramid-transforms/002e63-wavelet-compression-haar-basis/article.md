# Wavelet Compression (Haar Basis)
Also called: Haar Wavelet Linear Algebra Approach, Matrix-based Haar Transform, Multiresolution Image Compression

## Overview
Wavelet Compression (Haar Basis) expresses original volume data using a smaller set of coefficients via a Haar wavelet basis, supporting both lossless and lossy compression based on a user-specified error tolerance [S1, unofficial]. It distinguishes itself from sibling methods by offering a linear algebra formulation where the transform is executed via matrix multiplication, providing a mathematically explicit, albeit computationally less efficient, framework for understanding the decomposition [S2, unofficial]. Unlike [L2 Norm Wavelet Compression](3542-l2-norm-wavelet-compression.md) which optimizes coefficient selection via norm minimization, or [Biorthogonal Wavelet Compression](5381-biorthogonal-wavelet-compression.md) which uses symmetric filters for better phase properties, the Haar basis relies on the simplest orthogonal step-function basis [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset $V$ (or a 1D vector $\tilde{x}_0$ for the 1D case) and a user-specified error tolerance $\epsilon$ [S1, unofficial]. For lossless compression, $\epsilon = 0$ [S1, unofficial].
*   **Output:** A compressed set of coefficients $\tilde{X}_1$ (averages and differences) and, if lossy, a thresholded version of these coefficients.

**Data Structures:**
The method operates on vectors and matrices.
*   **Vector $\tilde{x}_0$:** The input data array of size $N$.
*   **Haar Matrix $A_k$:** An orthogonal matrix of size $2^k \times 2^k$ (or appropriate power of 2) representing the transform at level $k$. The columns are orthogonal with respect to the standard dot product [S2, unofficial].
*   **Combined Matrix $W$:** The product of individual level matrices $W = A_1 * A_2 * \dots * A_n$ [S2, unofficial].

**Steps:**
1.  **1D Transform (Vector):**
    For a single vector $\tilde{x}_0$, the transform is performed by multiplying by the Haar Matrix $A_1$:
    $$ \tilde{X}_1 = A_1 * \tilde{x}_0 $$
    The resulting vector $\tilde{X}_1$ contains components representing averages (low-frequency) and differences (high-frequency) of adjacent values in $\tilde{x}_0$ [S2, unofficial].

2.  **Multi-level Transform:**
    To achieve deeper compression levels, the process is recursive. The resulting vector from level $j$ is multiplied by the next Haar matrix $A_{j+1}$:
    $$ \tilde{X}_{j+1} = A_{j+1} * \tilde{X}_j $$
    Alternatively, a single combined matrix $W$ can be pre-computed as the product of all level matrices:
    $$ W = A_1 * A_2 * \dots * A_n $$
    The final coefficients are obtained by:
    $$ \tilde{X}_1 = W * \tilde{x}_0 $$
    [S2, unofficial]

3.  **2D Transform (Matrix):**
    For a 2D matrix $P$ of size $2^r \times 2^r$, the transform is applied separably.
    *   **Row Transformation:** $Q = P \hat{W}$ (where $\hat{W}$ is the Haar matrix applied to rows).
    *   **Column Transformation:** The result is transposed and multiplied by the Haar matrix again, or equivalently:
        $$ T = W^0 P W $$
        Here, $W^0$ denotes the transpose of $W$ (since $W$ is orthogonal, $W^0 = W^{-1}$ in some notations, but the source specifies $W^0 P W$ for compression) [S2, unofficial].
    *   **Decompression:** To recover $P$ from $T$:
        $$ P = (W^{-1})^0 T W^{-1} $$
        [S2, unofficial]

4.  **3D Transform (Volume):**
    The 2D approach is extended to 3D volumes. The volume is processed by applying the Haar transform along each of the three dimensions. This can be done by treating the volume as a stack of 2D matrices and applying the 2D transform to each, followed by the third dimension, or by applying the 1D transform to vectors along each axis sequentially [S2, unofficial].

5.  **Reconstruction:**
    To recover the original data, the inverse transform is applied. The inverse of the combined matrix is:
    $$ W^{-1} = A_1^{-1} * A_2^{-1} * \dots * A_n^{-1} $$
    Since the Haar matrices are orthogonal, their inverses are their transposes [S2, unofficial]. The original vector is recovered by:
    $$ \tilde{x}_0 = W^{-1} * \tilde{X}_1 $$
    [S2, unofficial]

**Parameters:**
*   **$\epsilon$ (epsilon):** User-specified error tolerance. Determines the threshold for discarding small coefficients in lossy mode. $\epsilon = 0$ for lossless [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def haar_matrix_1d(size):
    """
    Constructs the 1D Haar transform matrix for a given power-of-2 size.
    Returns an orthogonal matrix A.
    """
    if size == 1:
        return [[1.0]]
    
    # Recursive construction or iterative block construction
    # For size N, the matrix consists of averages and differences.
    # Average of x[i], x[i+1] is (x[i] + x[i+1]) / sqrt(2)
    # Difference is (x[i] - x[i+1]) / sqrt(2)
    # Note: Source [S2, unofficial] uses standard matrix multiplication without explicit normalization factors in the text,
    # but orthogonality implies normalization. We assume standard orthogonal Haar basis.
    
    A = [[0.0] * size for _ in range(size)]
    # Level 1: First row is all 1/sqrt(N), subsequent rows are differences
    # However, the source describes A1, A2... as sequential levels.
    # A1 operates on pairs.
    
    # Simplified implementation of the orthogonal Haar matrix
    for i in range(size):
        for j in range(size):
            # Standard Haar basis function evaluation
            # This is a placeholder for the exact matrix construction logic
            # which depends on the specific definition of A1, A2 in [S2, unofficial]
            pass 
    return A

def transform_vector(x, W):
    """
    Applies the Haar transform to vector x using matrix W.
    """
    # Matrix-vector multiplication
    n = len(x)
    result = [0.0] * n
    for i in range(n):
        sum_val = 0.0
        for j in range(n):
            sum_val += W[i][j] * x[j]
        result[i] = sum_val
    return result

def transform_matrix_2d(P, W):
    """
    Applies 2D Haar transform to matrix P.
    T = W^T * P * W
    """
    # Row transform: P * W
    # Column transform: W^T * (P * W)
    
    # Step 1: Row transformation Q = P * W
    rows = len(P)
    cols = len(P[0])
    W_cols = len(W)
    
    Q = [[0.0] * W_cols for _ in range(rows)]
    for i in range(rows):
        for j in range(W_cols):
            sum_val = 0.0
            for k in range(cols):
                sum_val += P[i][k] * W[k][j]
            Q[i][j] = sum_val
            
    # Step 2: Column transformation T = W^T * Q
    # Transpose W
    W_T = [[W[j][i] for j in range(len(W))] for i in range(len(W[0]))]
    
    T = [[0.0] * cols for _ in range(rows)]
    for i in range(rows):
        for j in range(cols):
            sum_val = 0.0
            for k in range(len(W_T)):
                sum_val += W_T[i][k] * Q[k][j]
            T[i][j] = sum_val
            
    return T

def inverse_transform_matrix_2d(T, W):
    """
    Reconstructs P from T.
    P = (W^-1)^T * T * W^-1
    Since W is orthogonal, W^-1 = W^T.
    So P = W * T * W^T
    """
    # W_inv is W^T
    W_inv = [[W[j][i] for j in range(len(W))] for i in range(len(W[0]))]
    
    # Step 1: T * W_inv
    rows = len(T)
    cols = len(T[0])
    W_inv_cols = len(W_inv)
    
    Q = [[0.0] * W_inv_cols for _ in range(rows)]
    for i in range(rows):
        for j in range(W_inv_cols):
            sum_val = 0.0
            for k in range(cols):
                sum_val += T[i][k] * W_inv[k][j]
            Q[i][j] = sum_val
            
    # Step 2: W_inv^T * Q  => W * Q
    W_T = [[W[j][i] for j in range(len(W))] for i in range(len(W[0]))]
    
    P = [[0.0] * cols for _ in range(rows)]
    for i in range(rows):
        for j in range(cols):
            sum_val = 0.0
            for k in range(len(W_T)):
                sum_val += W_T[i][k] * Q[k][j]
            P[i][j] = sum_val
            
    return P
```

## Subtopics

### [3D Non-standard Haar Wavelet Decomposition with Integer Packing](6693-3d-non-standard-haar-wavelet-decomposition-with-integer-pack.md)
Uses unsigned 32-bit integers to store coefficients, replacing floating-point arithmetic to leverage faster integer operations without loss of precision. It differs from standard float-based wavelet compression by utilizing the unused bits of a 32-bit integer word to handle intermediate overflows and underflows, allowing for a fixed-point hierarchy that occupies the same memory footprint as the original volume.

## Operating characteristics
*   **Time Complexity:** The matrix-based approach described in [S2, unofficial] involves $O(N^3)$ operations for an $N \times N$ 2D matrix if implemented as dense matrix multiplication, which is inefficient for large datasets. The source notes that "for large data sets there are much better ways to accomplish the wavelet transform," implying the Fast Wavelet Transform (O(N log N)) is preferred in practice, though the matrix formulation is provided for theoretical clarity [S2, unofficial].
*   **Memory Complexity:** The matrix-based approach requires storing the Haar matrix $W$, which is $O(N^2)$ for 2D. The integer packing variant [S1, unofficial] maintains the same memory footprint as the original volume.
*   **Strengths:** Simple mathematical definition; orthogonal basis ensures stable reconstruction; supports both lossless and lossy compression via $\epsilon$ [S1, unofficial].
*   **Limitations:** The Haar basis has poor frequency localization compared to smoother wavelets (like Daubechies or Biorthogonal); the dense matrix implementation is computationally expensive [S2, unofficial].
*   **When to use:** When a simple, orthogonal basis is required and computational efficiency is not the primary constraint, or for educational purposes to understand the linear algebra of wavelets [S2, unofficial].

## Background
The method is based on work by Stollnitz in computer graphics, applying multiresolution decomposition using wavelets [S1, unofficial]. It was used in the context of direct rendering of multiresolution decomposed volume data [S1, unofficial]. The matrix-based formulation is presented in the context of volumetric medical images visualization on mobile devices, where the simplicity of the Haar transform is leveraged [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
