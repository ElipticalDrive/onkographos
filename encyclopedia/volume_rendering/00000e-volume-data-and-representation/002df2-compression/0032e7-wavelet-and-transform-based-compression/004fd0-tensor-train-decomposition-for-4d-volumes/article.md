# Tensor-Train Decomposition for 4D Volumes

Also called: TT decomposition with efficient subsampling, TT4D, TT4D Compression, TT4D Decomposition, Tensor-Train-based 4D Time-Dependent Volume Rendering

## Overview
TT4D is a memory- and time-efficient lossy compression technique for four-dimensional (4D) time-varying volume data that utilizes Tensor-Train (TT) decomposition with efficient subsampling [S1, unofficial]. It solves the problem of high memory consumption and computational cost associated with standard TT construction on large datasets, enabling the processing of volumes up to 64GB [S1, unofficial]. Unlike sibling methods like [Value compression](6-value-compression.md) or [Blockwise Wavelet Compression](6100-blockwise-wavelet-compression.md) which may rely on variable-bit encoding or block-wise processing, TT4D guarantees fixed-rate compression by avoiding vector quantization, thereby enabling random-access patterns and interactive rendering of individual time steps [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D tensor $T$ of size $N_1 \times N_2 \times N_3 \times N_4$, representing spatial dimensions $(x, y, z)$ and time $t$.
*   **Output:** A set of TT cores $\{C_1, C_2, C_3, C_4\}$ and the ranks $\{R_1, R_2, R_3\}$.

**Data Structures:**
The TT decomposition factorizes $T$ into a sequence of 3D tensors (cores) $C_i$. For $i=1 \dots d$ (where $d=4$), $C_i \in \mathbb{R}^{R_{i-1} \times N_i \times R_i}$, with boundary ranks $R_0 = R_4 = 1$.
*   $C_1 \in \mathbb{R}^{1 \times N_1 \times R_1}$
*   $C_2 \in \mathbb{R}^{R_1 \times N_2 \times R_2}$
*   $C_3 \in \mathbb{R}^{R_2 \times N_3 \times R_3}$
*   $C_4 \in \mathbb{R}^{R_3 \times N_4 \times 1}$

The reconstruction of the original tensor is defined by the contraction:
$$ T_{i_1, i_2, i_3, i_4} = \sum_{r_1=1}^{R_1} \sum_{r_2=1}^{R_2} \sum_{r_3=1}^{R_3} (C_1)_{1, i_1, r_1} (C_2)_{r_1, i_2, r_2} (C_3)_{r_2, i_3, r_3} (C_4)_{r_3, i_4, 1} $$

**Algorithm: TT4D Construction**
The method is based on Sequential SVD (SSVD) but replaces full SVD computations on large matricized tensors with efficient subsampling to reduce memory and time [S1, unofficial].

1.  **Initialization:** Set $R_0 = 1$. Let $T^{(1)} = T$.
2.  **Iterative Core Extraction (for $k = 1$ to $3$):**
    a.  **Matricization:** Reshape the current tensor $T^{(k)}$ into a matrix $M_k$ of size $(R_{k-1} \prod_{j=1}^{k-1} N_j) \times (\prod_{j=k}^{4} N_j)$. *Note: Standard SSVD requires full reshaping. TT4D avoids this by subsampling.*
    b.  **Subsampling:** Instead of computing SVD on the full $M_k$, select a subset of columns using either **Random Column Sampling** or **Maximum Volume (MaxVol) Approximation**.
        *   *Random Sampling:* Select $p$ columns randomly.
        *   *MaxVol Sampling:* Iteratively select columns that maximize the volume of the parallelepiped spanned by selected columns. This requires auxiliary matrices and up to 15 iterations with an oversampling parameter $p=10$ [S1, unofficial].
    c.  **SVD on Subsampled Matrix:** Compute the SVD of the subsampled matrix $M_k^{sub}$.
    d.  **Rank Determination:** Determine the rank $R_k$ based on the desired compression ratio or error threshold. Truncate the SVD to keep the top $R_k$ singular values/vectors.
    e.  **Core Extraction:** Extract the left singular vectors to form the core $C_k$. The size of $C_k$ is $R_{k-1} \times N_k \times R_k$.
    f.  **Update Tensor:** Compute the residual tensor $T^{(k+1)}$ by multiplying the right singular vectors with the remaining part of the tensor. *Implementation Note: Perform operations in-place or on small submatrices to minimize memory consumption [S1, unofficial].*
3.  **Final Core:** The last core $C_4$ is derived from the remaining residual $T^{(4)}$ with rank $R_3$. Since $R_4=1$, $C_4$ has shape $R_3 \times N_4 \times 1$.

**Parameters:**
*   **MaxVol Iterations:** 15 [S1, unofficial].
*   **Oversampling Parameter ($p$):** 10 [S1, unofficial].
*   **Ranks ($R_i$):** Determined by the target compression ratio. For example, a 1:128 compression on the channel dataset used specific ranks resulting in a 1:8 subsampled time slice decoding time of 20.8ms [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tt4d_decompose(T, target_ranks=None, sampling_method='random', max_iterations=15, oversampling_p=10):
    """
    T: 4D numpy array of shape (N1, N2, N3, N4)
    target_ranks: List of [R1, R2, R3] or None (auto-determine)
    sampling_method: 'random' or 'maxvol'
    """
    d = 4
    ranks = [1] * (d + 1) # R0=1, R4=1
    cores = []
    
    current_tensor = T
    
    for k in range(1, d):
        # 1. Matricize along dimension k
        # Shape: (R_{k-1} * N_1 * ... * N_{k-1}) x (N_k * ... * N_d)
        # Note: R_{k-1} is 1 for k=1
        shape_left = 1
        for i in range(1, k):
            shape_left *= current_tensor.shape[i-1] # Adjust for 0-indexed logic if needed
        # Simplified matricization logic for TT:
        # Reshape to (prod(N_1..N_{k-1})*R_{k-1}, prod(N_k..N_d))
        # Since R_{k-1} is stored in previous core, we treat it as part of the left index
        # For k=1, left size is N1. For k>1, left size is R_{k-1} * N1 * ... * N_{k-1}
        
        # Get current rank from previous core
        if k == 1:
            r_prev = 1
        else:
            r_prev = ranks[k-1]
            
        # Calculate dimensions
        n_left = r_prev
        for i in range(k-1): # N_1 to N_{k-1}
            n_left *= current_tensor.shape[i]
            
        n_right = 1
        for i in range(k-1, d): # N_k to N_d
            n_right *= current_tensor.shape[i]
            
        # Reshape T to matrix M
        # This is the expensive step TT4D avoids via subsampling
        # M shape: (n_left, n_right)
        M = current_tensor.reshape(n_left, n_right)
        
        # 2. Subsample columns
        if sampling_method == 'random':
            # Select p columns randomly
            num_cols = oversampling_p # or based on target rank
            indices = random.sample(range(n_right), min(num_cols, n_right))
            M_sub = M[:, indices]
        elif sampling_method == 'maxvol':
            # Iterative maxvol selection
            # Requires auxiliary matrices and up to max_iterations
            # Returns indices of selected columns
            indices = maxvol_approximation(M, max_iterations, oversampling_p)
            M_sub = M[:, indices]
            
        # 3. SVD on subsampled matrix
        U, S, Vt = svd(M_sub, full_matrices=False)
        
        # 4. Determine Rank R_k
        # If target_ranks is provided, use it. Otherwise, use all or threshold.
        if target_ranks is not None:
            R_k = target_ranks[k-1]
        else:
            R_k = len(S) # Keep all for now, or apply threshold
            
        # Truncate
        U_k = U[:, :R_k]
        S_k = S[:R_k]
        Vt_k = Vt[:R_k, :]
        
        # 5. Form Core C_k
        # C_k shape: (R_{k-1}, N_k, R_k)
        # We need to reshape U_k back to the spatial dimensions
        # U_k corresponds to the left part of the tensor
        # The left part has size R_{k-1} * N_1 * ... * N_{k-1}
        # We need to separate R_{k-1} from the spatial dimensions
        
        # Reshape U_k to (R_{k-1}, N_1, ..., N_{k-1}, R_k)
        # Note: The first dimension of M was R_{k-1} * prod(N_1..N_{k-1})
        # We need to split this first dimension.
        
        # Construct the shape for the core
        core_shape = [r_prev]
        for i in range(k-1):
            core_shape.append(current_tensor.shape[i])
        core_shape.append(R_k)
        
        # Reshape U_k to core_shape
        # U_k is (n_left, R_k). n_left = r_prev * prod(N_1..N_{k-1})
        # We need to reshape the first dimension of U_k into [r_prev, N_1, ..., N_{k-1}]
        # This requires knowing the exact layout. Assuming row-major:
        U_reshaped = U_k.reshape(core_shape)
        
        cores.append(U_reshaped)
        
        # 6. Update Tensor for next iteration
        # T_new = M^T * V_k^T ? 
        # Standard SSVD update: T_{k+1} = (I \otimes ... \otimes V_k^T) * T_k
        # Efficiently: Multiply the right singular vectors with the right part of the tensor
        # V_k is (R_k, n_right)
        # We need to contract the last R_k dimensions of the tensor with V_k
        
        # Reshape V_k to match the right dimensions of the tensor
        # Right dimensions are N_k, ..., N_d
        # V_k shape: (R_k, N_k * ... * N_d)
        # Reshape V_k to (R_k, N_k, ..., N_d)
        right_shape = [current_tensor.shape[i] for i in range(k-1, d)]
        V_k_reshaped = Vt_k.T.reshape([R_k] + right_shape)
        
        # Contract: T_{k+1} = sum_{r_k} C_k[..., r_k] * V_k_reshaped[r_k, ...]
        # This is a tensor contraction. 
        # In practice, this is done by reshaping the tensor and multiplying
        current_tensor = contract_tensor_with_core(current_tensor, V_k_reshaped, k)
        
    # 7. Last Core C_4
    # Current tensor should be of shape (R_3, N_4)
    # Reshape to (R_3, N_4, 1)
    cores.append(current_tensor.reshape(ranks[3], current_tensor.shape[-1], 1))
    
    return cores
```

## Subtopics

### TT Reconstruction for Time-Dependent Rendering
A method to reconstruct a specific time step $t$ from the TT cores for volume rendering, optimized for GPU-based on-the-fly decoding [S1, unofficial]. It utilizes preprocessed cores to reduce computational load.
1.  **Preprocessing:** Multiply the first 3 TT cores ($C_1, C_2, C_3$) together to form a preprocessed tensor $P$. $P \in \mathbb{R}^{N_1 \times N_2 \times N_3 \times R_3}$. This step is performed once during initialization.
2.  **Extraction:** For a specific time step $t$, extract the $t$-th column vector from the last TT core ($C_4$). $C_4 \in \mathbb{R}^{R_3 \times N_4 \times 1}$. The column vector $v_t \in \mathbb{R}^{R_3}$ is obtained by indexing $C_4[:, t, 0]$.
3.  **Reconstruction:** Multiply the preprocessed tensor $P$ with the extracted column vector $v_t$. The result is the 3D volume for time step $t$: $V_t = P \times_4 v_t$ (tensor-vector multiplication along the 4th mode). This yields a 3D tensor of size $N_1 \times N_2 \times N_3$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Decomposition is faster than standard SSVD, TTHRESH, and tntorch due to subsampling, though MaxVol sampling is slower than Random sampling due to iterative processing [S1, unofficial]. Reconstruction of a single time step is significantly faster than ZFP fixed-rate (which requires decoding 4 time steps) [S1, unofficial].
*   **Memory Complexity:** Linear growth with dimensionality. Memory usage is significantly lower than TTHRESH, tntorch, and default SSVD because it avoids large intermediate matrices and performs operations in-place [S1, unofficial].
*   **Performance:** For the channel dataset compressed at 1:128, a 1:8 subsampled time slice can be decoded in an average time of 20.8ms [S1, unofficial]. Full reconstruction for large datasets (turbulence, channel) can exceed 100ms, necessitating subsampled reconstruction for fluent interaction [S1, unofficial].
*   **Strengths:** Fixed-rate compression, random access, low memory footprint, support for interactive filtering (e.g., Difference of Gaussian filter in <2ms) [S1, unofficial].
*   **Limitations:** Not designed for maximal compression (TTHRESH achieves higher ratios); limited to 1:128 compression ratio for 32-bit float data if compared to ZFP's fixed-rate limit, but TT4D can go higher with lower accuracy [S1, unofficial]. Reconstruction time for full resolution can be high (>100ms) for very large datasets [S1, unofficial].

## Background
The method was introduced by C. Hartmann, R. Ballester-Ripoll, and R. Pajarola in the paper 'TT4D' published in Computer Graphics Forum by Eurographics and John Wiley & Sons Ltd [S1, unofficial]. It addresses the limitations of integrating tensor decompositions into time-dependent interactive visualization pipelines, which previously suffered from high memory and time costs [S1, unofficial].

## Sources
[S1] Tensor-Train-based 4D Time-Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
