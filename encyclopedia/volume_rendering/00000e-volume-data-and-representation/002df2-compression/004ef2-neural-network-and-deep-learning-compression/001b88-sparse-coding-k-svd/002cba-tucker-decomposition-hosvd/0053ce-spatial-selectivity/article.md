# Spatial Selectivity

Also called: Subregion Reconstruction via Factor Selection

## Overview
Spatial Selectivity is a property of Tensor Approximation (TA) that allows the reconstruction of a specific spatial subvolume by selecting a subset of rows from the factor matrices corresponding to the spatial dimensions [S1, S2, unofficial]. It enables efficient access to spatially restricted data without reconstructing the full volume, which is distinct from sibling methods like [Truncated Tensor Decomposition](7540-truncated-tensor-decomposition.md) that reduce rank globally.

## How it works
**Inputs:**
- The Tucker core tensor $B$ of size $R_1 \times R_2 \times R_3$.
- The factor matrices $U^{(1)}$ ($I_1 \times R_1$), $U^{(2)}$ ($I_2 \times R_2$), and $U^{(3)}$ ($I_3 \times R_3$) [S1, unofficial].
- Row-index subranges $J_n \subseteq [0 \dots I_n]$ for each mode $n$ corresponding to the desired spatial subvolume [S1, unofficial].

**Outputs:**
- The reconstructed subvolume tensor $A_f^{J_1 \times J_2 \times J_3}$ [S1, unofficial].

**Data Structures:**
- $U^{(n)}$: Matrix where rows correspond to spatial indices and columns correspond to basis vectors (rank).
- $U_{J_n}^{(n)}$: Submatrix of $U^{(n)}$ containing only rows indexed by $J_n$.

**Steps:**
1. Identify the spatial bounds of the target subvolume, defined by index ranges $J_1, J_2, J_3$ for modes 1, 2, and 3 respectively [S1, unofficial].
2. Extract the row-block submatrices $U_{J_1}^{(1)}$, $U_{J_2}^{(2)}$, and $U_{J_3}^{(3)}$ from the full factor matrices [S1, unofficial].
3. Compute the subvolume reconstruction using the mode-n product:
   $$ A_f^{J_1 \times J_2 \times J_3} = B \times_1 U_{J_1}^{(1)} \times_2 U_{J_2}^{(2)} \times_3 U_{J_3}^{(3)} $$
   where $\times_n$ denotes the mode-n product [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_subvolume(core_tensor_B, factor_matrices, j1_range, j2_range, j3_range):
    """
    Reconstructs a spatial subvolume using Spatial Selectivity.
    
    Args:
        core_tensor_B: Tensor of shape (R1, R2, R3)
        factor_matrices: List of matrices [U1, U2, U3]
        j1_range: List of row indices for mode 1
        j2_range: List of row indices for mode 2
        j3_range: List of row indices for mode 3
        
    Returns:
        Reconstructed subvolume tensor
    """
    U1, U2, U3 = factor_matrices
    
    # Select specific rows from factor matrices
    U1_sub = U1[j1_range, :]
    U2_sub = U2[j2_range, :]
    U3_sub = U3[j3_range, :]
    
    # Compute mode-n products
    # Step 1: B x1 U1_sub
    # Result shape: (len(j1_range), R2, R3)
    temp1 = mode_n_product(core_tensor_B, U1_sub, mode=1)
    
    # Step 2: temp1 x2 U2_sub
    # Result shape: (len(j1_range), len(j2_range), R3)
    temp2 = mode_n_product(temp1, U2_sub, mode=2)
    
    # Step 3: temp2 x3 U3_sub
    # Result shape: (len(j1_range), len(j2_range), len(j3_range))
    subvolume = mode_n_product(temp2, U3_sub, mode=3)
    
    return subvolume

def mode_n_product(tensor, matrix, mode):
    """
    Computes the mode-n product of a tensor and a matrix.
    For a 3D tensor T and matrix M, mode=1: result[i,j,k] = sum_l T[i,j,l] * M[i,l]
    """
    # Implementation depends on tensor library, but conceptually:
    # Reshape tensor to matrix along mode n, multiply, reshape back.
    # Here we assume a standard implementation exists.
    pass
```

**Edge Cases:**
- The reconstruction maintains the original spatial resolution for the selected indices, not a lower resolution [S1, unofficial].
- If $J_n$ is empty, the resulting subvolume dimension is 0.

## Operating characteristics
- **Time Complexity:** Proportional to the size of the selected subvolume and the ranks $R_n$. It avoids computing the full volume reconstruction.
- **Memory Complexity:** Requires storing the full factor matrices and core tensor, but only allocates memory for the output subvolume.
- **Strengths:** Enables efficient view-frustum culling and adaptive brick selection in interactive multiresolution volume visualization [S1, unofficial].
- **Limitations:** Requires random access to rows of factor matrices. Not suitable for reducing the overall storage size (that is handled by rank reduction) [S1, unofficial].

## Background
Spatial Selectivity exploits the property that in Tensor Approximation (TA), the rows of the factor matrices along the spatial axes directly correspond to the spatial distribution of the data [S1, unofficial]. This allows for subvolume reconstruction by simply selecting row subsets, a feature not previously fully exploited in TA-based visualization systems [S1, unofficial]. It is used in conjunction with [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md) to provide multiresolution and multiscale volume visualization capabilities [S1, unofficial].

## Sources
[S1] Tensor Approximation Properties for Multiresolution and ..., https://www.ifi.uzh.ch/dam/jcr:00000000-4c65-90f4-0000-00006dc77a03/TA_Poster.pdf, fetched 2026-10-29 (unofficial)
[S2] Analysis of Tensor Approximation for Compression-Domain Volume ..., https://www.ifi.uzh.ch/vmml/publications/analysistensorapproximationtalk/tensorapproximation.pdf, fetched 2026-10-29 (unofficial)
