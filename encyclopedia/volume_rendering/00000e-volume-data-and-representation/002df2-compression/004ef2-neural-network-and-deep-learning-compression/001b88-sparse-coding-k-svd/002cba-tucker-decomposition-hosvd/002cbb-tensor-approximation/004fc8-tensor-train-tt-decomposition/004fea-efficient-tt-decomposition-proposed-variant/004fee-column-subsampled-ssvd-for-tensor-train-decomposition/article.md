# Column Subsampled SSVD for Tensor-Train Decomposition
Also called: Improved SSVD, SSVD with column subsampling

## Overview
This method computes the first step of the Tensor-Train (TT) decomposition by avoiding the explicit construction of the full mode-1 unfolded matrix. It exploits the fact that the column space of the unfolded tensor is low-dimensional to compute the required right singular vectors via a subsampled SVD, thereby reducing memory usage and computational cost compared to standard SSVD. It addresses the memory and time intensity of standard SSVD by working only on parts of the full data [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
- Input: A 4D tensor $\mathcal{T} \in \mathbb{R}^{N_1 \times N_2 \times N_3 \times N_4}$ and target TT ranks $R_1, R_2, R_3$.
- Output: The first TT core $C_1 \in \mathbb{R}^{1 \times N_1 \times R_1}$ and the intermediate matrix $V^\top \in \mathbb{R}^{R_1 \times (N_2 N_3 N_4)}$ which serves as the starting point for subsequent cores.

**Data Structures**
The TT cores $C_i$ have shapes $\mathbb{R}^{R_{i-1} \times N_i \times R_i}$ with boundary ranks $R_0 = 1$ and $R_4 = 1$ [S1][S2, unofficial].
- $C_1$ is a matrix of shape $1 \times N_1 \times R_1$.
- $C_2$ is a 3D tensor of shape $R_1 \times N_2 \times R_2$.
- $C_3$ is a 3D tensor of shape $R_2 \times N_3 \times R_3$.
- $C_4$ is a matrix of shape $R_3 \times N_4 \times 1$ [S1][S2, unofficial].

**Algorithm**
1. **Mode-1 Unfolding**: Unfold $\mathcal{T}$ into a matrix $A \in \mathbb{R}^{N_1 \times (N_2 N_3 N_4)}$. The first dimension corresponds to mode 1, and the second dimension is the product of the remaining modes [S1].
2. **Column Subsampling**: Recognize that $A$ is highly imbalanced with $N_1 \ll (N_2 N_3 N_4)$. The linear column space of $A$ is spanned by at most $N_1$ linearly independent columns [S1][S2, unofficial]. Instead of computing the full SVD of $A$, select a subset of $R_1$ columns from $A$ to form a small matrix $A_{sub} \in \mathbb{R}^{N_1 \times R_1}$ [S1][S2, unofficial].
3. **Subsampled SVD**: Compute the SVD of the subsampled matrix: $A_{sub} = U_{sub} \Sigma_{sub} W_{sub}^\top$, where $U_{sub} \in \mathbb{R}^{N_1 \times R_1}$ contains the left singular vectors [S1][S2, unofficial].
4. **Recover Right Singular Vectors**: Compute $V^\top$ using the relation $V^\top = U_{sub}^\top A$ [S1][S2, unofficial]. This avoids the expensive SVD computation on the full wide matrix $A$ [S2, unofficial].
5. **Form First Core**: Reshape $U_{sub}$ into the first TT core $C_1$ of shape $1 \times N_1 \times R_1$ [S1][S2, unofficial].
6. **Iterate**: Use $V^\top$ as the input for the next step to compute $C_2$, continuing until all cores are obtained [S1][S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def column_subsampled_ssvd(T, R1):
    """
    T: 4D tensor of shape (N1, N2, N3, N4)
    R1: Target rank for the first core
    Returns: C1 (shape 1 x N1 x R1), Vt (shape R1 x N2*N3*N4)
    """
    N1, N2, N3, N4 = T.shape
    
    # Step 1: Mode-1 unfolding
    # Reshape to (N1, N2*N3*N4)
    A = T.reshape(N1, -1)
    
    # Step 2: Column Subsampling
    # Select R1 columns from A. 
    # Note: The sources do not specify the exact selection strategy (e.g., random, 
    # deterministic, or based on column norms). [G] We assume random selection 
    # for implementation completeness, but this is a critical implementation detail 
    # not specified in the provided notes.
    indices = random.sample(range(A.shape[1]), R1)
    A_sub = A[:, indices]
    
    # Step 3: Subsampled SVD
    # Compute SVD of the small matrix A_sub
    U_sub, S_sub, Vt_sub = svd(A_sub, full_matrices=False)
    # U_sub is (N1, R1)
    
    # Step 4: Recover Right Singular Vectors
    # Vt = U_sub^T * A
    Vt = U_sub.T @ A
    
    # Step 5: Form First Core
    # Reshape U_sub into (1, N1, R1)
    C1 = U_sub.reshape(1, N1, R1)
    
    return C1, Vt
```

**Parameters**
- $R_1$: The target rank for the first TT core. Typical values depend on the desired compression ratio and accuracy. Not given in the sources.

**Edge Cases and Pitfalls**
- **Imbalanced Matrix**: The matrix $A$ is highly imbalanced ($N_1 \ll N_2 N_3 N_4$), leading to linearly dependent columns. The method relies on the assumption that the subsampled columns capture the essential column space [S1][S2, unofficial].
- **Memory Efficiency**: The method avoids explicit reshaping of the large matrix $V^\top$ by exploiting column-major storage, reducing memory overhead [S2, unofficial].
- **Selection Strategy**: The specific method for selecting the $R_1$ columns is not detailed in the sources [S1][S2, unofficial].

## Operating characteristics
- **Time Complexity**: Reduced compared to full SVD on $A$. The dominant cost is the SVD on the small matrix $A_{sub}$ ($O(N_1 R_1^2)$) and the matrix multiplication $U_{sub}^\top A$ ($O(N_1 R_1 (N_2 N_3 N_4))$) [S1][S2, unofficial].
- **Memory Complexity**: Significantly reduced as it avoids storing the full $N_1 \times (N_2 N_3 N_4)$ matrix explicitly in some implementations by leveraging storage layout [S2, unofficial].
- **Strengths**: Memory-efficient, avoids expensive SVD on large matrices [S1][S2, unofficial].
- **Limitations**: Accuracy depends on the column selection strategy [S1][S2, unofficial].

## Background
This method is proposed in the context of Tensor-Train-based 4D time-dependent volume rendering [S1][S2, unofficial]. It is an improvement over the standard SSVD approach to reduce memory and time consumption [S1][S2, unofficial].

## Sources
[S1] TT4D: Tensor‐Train‐based 4D Time‐Dependent Volume Rendering, https://doi.org/10.1111/cgf.70471, fetched 2026-10-27
[S2] Tensor-Train-based 4D Time-Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
