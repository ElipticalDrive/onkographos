# Tensor-Train (TT) Decomposition
Also called: 4D Volume to TT Cores, TT Decomposition, TT Format Decomposition, Tensor-Train (TT) Format Decomposition and Reshaping, Tensor-Train Singular Value Decomposition (TT-SVD)

## Overview
Tensor-Train (TT) Decomposition factorizes a high-dimensional input tensor into a sequence of two- and three-dimensional tensors called TT cores, reducing memory complexity from quartic to linear growth with respect to dimensionality [S1][S2, unofficial]. It solves the problem of high-dimensional data storage and computation by avoiding the exponential growth of parameters associated with dense tensor representations [S2, unofficial]. Unlike [Tucker Decomposition (HOSVD)](5363-tucker-decomposition-hosvd.md), where core dimensionality grows with tensor dimensionality, TT cores maintain fixed rank dimensions, making it especially suitable for high-dimensional data such as 4D volumes [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A tensor $T \in \mathbb{R}^{N_1 \times \dots \times N_d}$ (typically a 4D volume $N_1 \times N_2 \times N_3 \times N_4$) [S2, unofficial].
- Parameters: Predefined ranks $R_1, \dots, R_{d-1}$ (for 4D: $R_1, R_2, R_3$) [S1][S2, unofficial].
- Output: A sequence of TT cores $C_1, \dots, C_d$, where $C_i \in \mathbb{R}^{R_{i-1} \times N_i \times R_i}$, with boundary conditions $R_0 = R_d = 1$ [S1][S2, unofficial].

**Data Structures:**
- TT Core $C_i$: A 3D tensor (or 2D matrix if $R_{i-1}=1$ or $R_i=1$) with dimensions $R_{i-1} \times N_i \times R_i$ [S1][S2, unofficial].
- Unfolded Matrix $A$: A 2D matrix obtained by reshaping $T$ along a specific mode [S1][S2, unofficial]. For mode-$k$ unfolding, $A \in \mathbb{R}^{N_k \times (N_1 \dots N_{k-1} N_{k+1} \dots N_d)}$ [S1][S2, unofficial].

**Formula:**
The tensor $T$ is reconstructed from the cores via the outer product $\circ$:
$$ T = \sum_{r_1} \cdots \sum_{r_{d-1}} C_1[:, r_1] \circ C_2[r_1, :, r_2] \circ \cdots \circ C_d[r_{d-1}, :] $$
where $C_i \in \mathbb{R}^{R_{i-1} \times N_i \times R_i}$ and $R_0 = R_d = 1$ [S1][S2, unofficial].

**Algorithm (Sequential SVD / TT-SVD):**
The naive implementation computes the decomposition via Sequential SVD (SSVD) [S1][S2, unofficial].
1. **Mode-1 Unfolding:** Reshape $T$ into a matrix $A \in \mathbb{R}^{N_1 \times (N_2 \dots N_d)}$ by flattening all dimensions except the first [S1][S2, unofficial].
2. **SVD:** Compute the SVD of $A$: $A \approx U \Sigma V^\top$ [G].
3. **Extract First Core:** The first TT core $C_1$ is formed by reshaping the left singular vectors $U \in \mathbb{R}^{N_1 \times R_1}$ into shape $1 \times N_1 \times R_1$ (since $R_0=1$) [S1][S2, unofficial].
4. **Iterate:** For $k = 2$ to $d-1$:
   a. Reshape the remaining factor $V^\top$ (absorbing $\Sigma$) into a tensor of shape $R_{k-1} \times N_k \times \dots \times N_d$ [S1][S2, unofficial].
   b. Unfold this tensor along its first dimension to form matrix $A_k \in \mathbb{R}^{R_{k-1} \times (N_k \dots N_d)}$ [S1][S2, unofficial].
   c. Compute SVD on $A_k$: $A_k \approx U_k \Sigma_k V_k^\top$ [G].
   d. Extract core $C_k$ by reshaping $U_k \in \mathbb{R}^{R_{k-1} \times R_k}$ into shape $R_{k-1} \times N_k \times R_k$ [S1][S2, unofficial].
   e. Update the remainder tensor using $V_k^\top$ (absorbing $\Sigma_k$) for the next iteration [S1][S2, unofficial].
5. **Last Core:** The final core $C_d$ is formed by reshaping the remaining tensor of shape $R_{d-1} \times N_d$ into $R_{d-1} \times N_d \times 1$ [S1][S2, unofficial].

**Pseudocode (Standard TT-SVD):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tt_svd(T, ranks):
    """
    T: Tensor of shape (N1, N2, ..., Nd)
    ranks: List of ranks [R1, R2, ..., R_{d-1}]
    Returns: List of TT cores [C1, C2, ..., Cd]
    """
    d = len(T.shape)
    cores = []
    current_tensor = T  # Initialize current tensor state to T [G]

    for k in range(1, d):
        # Mode-k unfolding: reshape to (N_k, product of others)
        # Permute so mode k is first, then flatten the rest
        perm = [k] + [i for i in range(d) if i != k]
        current_tensor = current_tensor.transpose(perm)
        N_k = current_tensor.shape[0]
        other_dims = current_tensor.shape[1:]
        A = current_tensor.reshape(N_k, -1)

        # Truncated SVD: A ≈ U Σ V^T
        # U: (N_k, R_prev), S: (R_prev,), Vt: (R_prev, prod_others)
        # Note: For k=1, N_k is original N1. For k>1, N_k is the previous rank R_{k-1}.
        R_prev = ranks[k-2] if k > 1 else 1
        R_curr = ranks[k-1]

        U, S, Vt = svd(A, k=R_curr)  # [G] standard truncated SVD routine

        # Form Core C_k
        # C_k shape: (R_{k-1}, N_k, R_k)
        # For k=1, R_prev=1, so U is (N_1, R_1). Reshape to (1, N_1, R_1).
        # For k>1, U is (R_{k-1}, R_k). Reshape to (R_{k-1}, N_k, R_k).
        C_k = U.reshape(R_prev, N_k, R_curr)
        cores.append(C_k)

        # Prepare remainder for next iteration: Vt * diag(S)
        # Vt is (R_curr, prod_others). We absorb S into Vt.
        Vt_scaled = Vt * S.reshape(-1, 1)
        # Reshape remainder to (R_curr, N_{k+1}, ..., N_d)
        remainder_shape = [R_curr] + list(other_dims[1:])
        current_tensor = Vt_scaled.reshape(remainder_shape)

    # Last core C_d
    # After loop, current_tensor is shape (R_{d-1}, N_d)
    C_d = current_tensor.reshape(ranks[d-2], T.shape[d-1], 1)
    cores.append(C_d)

    return cores

def svd(A, k):
    """[G] Standard truncated SVD routine returning top k singular vectors/values."""
    import numpy as np
    U, S, Vt = np.linalg.svd(A, full_matrices=False)
    return U[:, :k], S[:k], Vt[:k, :]
```

**Edge Cases and Pitfalls:**
- **Memory/Time Intensity:** Naive SSVD requires repeatedly unfolding and reshaping the full tensor, which is extremely memory- and time-intensive for large data [S1][S2, unofficial].
- **Imbalanced Matrices:** The unfolded matrices are highly imbalanced (far more columns than rows), leading to numerical instability and high computational cost [S1][S2, unofficial].
- **Rank Selection:** Predefined ranks $R_i$ must be chosen carefully; too low causes poor approximation, too high negates compression benefits [S1][S2, unofficial].

## Subtopics

### [Efficient TT Decomposition (Proposed Variant)](14084-efficient-tt-decomposition-proposed-variant.md)
A memory-optimized variant of the TT-SVD that avoids processing the full unfolded tensor by working only on parts of the full data. It addresses the memory and time intensity of standard SSVD by using column sampling (random or maximum volume) to approximate the left singular vectors from a subset of columns.

## Operating characteristics
- **Time Complexity:** Standard TT-SVD is $O(d \cdot N \cdot R^3)$ or worse depending on unfolding size, where $N$ is total elements. The imbalanced SVDs make it slow for large $N$ [S1][S2, unofficial].
- **Memory Complexity:** Linear growth with dimensionality $d$ for fixed ranks, compared to quartic growth for Tucker on 4D data [S1][S2, unofficial]. Storage is $O(d \cdot N_i \cdot R^2)$ [S1][S2, unofficial].
- **Strengths:** Suitable for high-dimensional data (e.g., 4D volumes) where Tucker decomposition becomes impractical due to memory constraints [S1][S2, unofficial]. Supports interactive reconstruction and rendering by decoding time steps on demand without reconstructing the full 4D tensor [S2, unofficial].
- **Limitations:** Standard TT-SVD is impractical for large datasets due to memory and time requirements [S1][S2, unofficial]. Requires predefined ranks [S1][S2, unofficial].
- **When to use:** When dealing with high-dimensional volume data (e.g., time-varying volumes) and memory is a constraint, and when interactive access to specific time steps is required [S1][S2, unofficial].

## Background
The Tensor-Train decomposition was introduced by Oseledets in 2011 [S1][S2, unofficial]. It is based on factorizing a tensor into a chain of low-rank matrices (cores) [S1][S2, unofficial]. The method has been applied to 4D time-dependent volume rendering to support interactive visualization of individual time steps with random and continuous time access [S1][S2, unofficial]. The naive implementation is known as TT-SVD [S1][S2, unofficial].

## Sources
[S1] TT4D: Tensor‐Train‐based 4D Time‐Dependent Volume Rendering, https://doi.org/10.1111/cgf.70471, fetched 2026-10-27
[S2] Tensor-Train-based 4D Time-Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
