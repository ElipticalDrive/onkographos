# Partial Decoding

Also called: On-the-fly partial decoding, Partial TT Decoding for Interactive Rendering

## Overview
Partial Decoding is a technique for visualizing 4D Tensor Train (TT) volumes that enables interactive time access and low memory usage by avoiding the reconstruction of the full 4D tensor [S1]. It differs from naive reconstruction, which recreates the entire 4D volume in memory, and per-time-step reconstruction, which is too slow for interactive frame rates on large datasets [S1]. By pre-computing a reduced base tensor, it reduces the per-time-step cost to a single tensor-vector multiplication [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Four TT cores $C_1, C_2, C_3, C_4$ representing a 4D volume with spatial dimensions $N_1, N_2, N_3$ and time dimension $N_4$. The TT ranks are $R_1, R_2, R_3, R_4$ (where $R_4=1$ for standard TT format).
*   **Output:** A reduced base tensor $T_c$ (pre-computation) and, for each requested time step $t$, a 3D spatial volume slice $V_t$ (runtime).

**Data Structures:**
*   **TT Cores:** $C_k$ is a tensor of shape $R_{k-1} \times N_k \times R_k$ (with $R_0=R_4=1$).
*   **Reduced Base Tensor ($T_c$):** A 4D tensor of shape $N_1 \times N_2 \times N_3 \times R_3$. This structure is significantly smaller than the full 4D volume because $R_3 \ll N_4$ [S1].

**Steps:**

1.  **Pre-computation (One-time startup cost):**
    Multiply the first three TT cores $C_1, C_2, C_3$ together to form the reduced base tensor $T_c$. This operation involves two tensor-tensor multiplications [S1].
    $$ T_c(i, j, k, r) = \sum_{a=1}^{R_1} \sum_{b=1}^{R_2} C_1(i, a) \cdot C_2(a, j, b) \cdot C_3(b, k, r) $$
    where $i \in [1, N_1], j \in [1, N_2], k \in [1, N_3], r \in [1, R_3]$ [S1].

2.  **Runtime (Per time step $t$):**
    For a required time step $t$ (where $t \in [1, N_4]$), extract the $t$-th column vector $v_t$ from the last TT core $C_4$. The shape of $v_t$ is $R_3 \times 1$ (effectively a vector of length $R_3$).
    Multiply the base tensor $T_c$ with this vector $v_t$ to obtain the 3D volume slice $V_t$:
    $$ V_t(i, j, k) = \sum_{r=1}^{R_3} T_c(i, j, k, r) \cdot v_t(r) $$
    This requires a single tensor-vector multiplication [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def partial_decode_precompute(C1, C2, C3):
    """
    Pre-computes the reduced base tensor T_c.
    C1: shape (1, N1, R1)
    C2: shape (R1, N2, R2)
    C3: shape (R2, N3, R3)
    Returns: T_c of shape (N1, N2, N3, R3)
    """
    # Step 1: Multiply C1 and C2
    # Result M12 has shape (N1, N2, R1*R2) if flattened, 
    # but we treat it as a tensor contraction.
    # M12[i, j, a, b] = C1[i, a] * C2[a, j, b]
    # We contract over R1.
    
    # Efficient implementation using einsum or nested loops:
    # M12 shape: (N1, N2, R2) after contracting R1? 
    # Actually, C1 is (1, N1, R1), C2 is (R1, N2, R2).
    # Contraction over index 2 of C1 and index 0 of C2.
    # Result shape: (N1, N2, R2)
    
    # Let's define a generic tensor contraction function for clarity
    # Contract C1 (dim 2) with C2 (dim 0)
    M12 = contract(C1, [2], C2, [0]) 
    # M12 shape: (N1, N2, R2)
    
    # Step 2: Multiply M12 with C3
    # M12 is (N1, N2, R2), C3 is (R2, N3, R3)
    # Contract M12 (dim 2) with C3 (dim 0)
    # Result T_c shape: (N1, N2, N3, R3)
    T_c = contract(M12, [2], C3, [0])
    
    return T_c

def partial_decode_runtime(T_c, C4, time_step_index):
    """
    Reconstructs a single time step volume.
    T_c: shape (N1, N2, N3, R3)
    C4: shape (R3, N4, 1)
    time_step_index: integer t in [0, N4-1]
    Returns: V_t of shape (N1, N2, N3)
    """
    # Extract the t-th column vector from C4
    # C4 has shape (R3, N4, 1). The time dimension is index 1.
    # We want the vector of length R3 corresponding to time t.
    # v_t[r] = C4[r, t, 0]
    
    v_t = C4[:, time_step_index, 0] 
    # v_t shape: (R3,)
    
    # Multiply T_c with v_t
    # T_c[i, j, k, r] * v_t[r] summed over r
    # Result V_t[i, j, k]
    
    # Using einsum notation: 'ijk r, r -> ijk'
    V_t = contract(T_c, [3], v_t, [0])
    
    return V_t

def contract(A, dims_A, B, dims_B):
    """
    Generic tensor contraction.
    Contracts dimensions dims_A of A with dims_B of B.
    Returns the resulting tensor with contracted dimensions removed.
    """
    # Implementation detail: This performs the summation over the specified indices.
    # For example, if A is (N1, N2, R2) and B is (R2, N3, R3),
    # and we contract A[2] with B[0], the result is (N1, N2, N3, R3).
    # This is a standard tensor network operation.
    # In Python/NumPy, this can be done via einsum or matmul reshaping.
    # For the purpose of this article, we assume a standard tensor contraction primitive exists.
    pass
```

**Parameters:**
*   $R_3$: The third TT rank. Typical values depend on the data compressibility, but must satisfy $R_3 \ll N_4$ for the method to be effective [S1].

**Edge Cases and Pitfalls:**
*   **Naive Reconstruction:** Reconstructing the full 4D tensor defeats the purpose of limiting memory usage [S1].
*   **Performance:** Per-time-step full reconstruction (without pre-computation) is too slow for interactive time access with large datasets due to the high number of multiplications required [S1].
*   **Memory:** The base tensor $T_c$ must fit in memory. If $R_3$ is large, $T_c$ may become too large. However, since $R_3 \ll N_4$, it is significantly smaller than the full volume [S1].

## Operating characteristics
*   **Time Complexity:**
    *   Pre-computation: $O(N_1 N_2 N_3 R_1 R_2 R_3)$ (two tensor-tensor multiplications) [S1]. This is a one-time startup cost [S2, unofficial].
    *   Per-time-step: $O(N_1 N_2 N_3 R_3)$ (one tensor-vector multiplication) [S1].
*   **Memory Complexity:**
    *   Storage: $O(N_1 N_2 N_3 R_3 + \sum R_i N_i)$ for the base tensor and TT cores [S1].
    *   The base tensor $T_c$ is significantly smaller than the full 4D volume $N_1 N_2 N_3 N_4$ because $R_3 \ll N_4$ [S1].
*   **Measured Performance:**
    *   Partial decoding reduces reconstruction time per time step significantly compared to naive reconstruction [S2, unofficial].
    *   It is much faster than other compression methods like ZFP for full-resolution reconstruction of individual time steps [S2, unofficial].
*   **Strengths:**
    *   Enables interactive time access and random access to time steps [S1].
    *   Allows rendering of data larger than available memory [S1].
*   **Limitations:**
    *   Requires a one-time startup cost for pre-computation [S2, unofficial].
    *   Assumes the first three dimensions are spatial and the last is time [S1].

## Background
Partial Decoding was introduced in the context of TT4D, a Tensor-Train-based 4D time-dependent volume rendering method [S1]. It addresses the challenge of visualizing large 4D datasets interactively by leveraging the structure of the Tensor Train decomposition [S1]. The method was presented in the paper "TT4D: Tensor‐Train‐based 4D Time‐Dependent Volume Rendering" [S1].

## Sources
[S1] TT4D: Tensor‐Train‐based 4D Time‐Dependent Volume Rendering, https://doi.org/10.1111/cgf.70471, fetched 2026-10-27
[S2] Tensor-Train-based 4D Time‐Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
