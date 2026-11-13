# Sparse Coding (K-SVD)

Also called: Compressive Volume Rendering via Dictionary Learning, Dictionary Learning for Volume Rendering, K-SVD, K-SVD Dictionary Learning

## Overview
Sparse Coding (K-SVD) compresses volume rendering data by learning an overcomplete dictionary from training images and representing image blocks as sparse linear combinations of dictionary atoms [S2, unofficial]. It differs from fixed-basis methods like [Karhunen-Loeve Transform (KLT)](5365-karhunen-loeve-transform-klt.md) or [Convolutional Autoencoder for Volume Compression](5718-convolutional-autoencoder-for-volume-compression.md) by adapting the basis to the specific statistical structure of the volume renderings, allowing for higher compression ratios and better artifact reduction in scenarios with missing or sparse data [S2, unofficial].

## How it works
**Inputs:** A dataset of volume rendered images (grayscale or RGB) used for training; a block size $B \times B$ (or $B \times B \times 3$ for RGB); target dictionary size $D_{rows} \times D_{cols}$; target sparsity level $L$.
**Outputs:** A learned dictionary matrix $\mathbf{D}$ and, for a given test image, sparse coefficient matrices $\mathbf{X}$.

**Data Structures:**
*   **Dictionary $\mathbf{D}$:** A matrix of size $D_{rows} \times D_{cols}$. Each column is a dictionary atom (basis vector). Columns are normalized to unit 2-norm.
*   **Coefficient Matrix $\mathbf{X}$:** A matrix of size $D_{cols} \times N$, where $N$ is the number of training blocks. Each column $\mathbf{x}_i$ contains the sparse coefficients for the $i$-th block.
*   **Training Data Matrix $\mathbf{Y}$:** A matrix of size $D_{rows} \times N$, where each column $\mathbf{y}_i$ is a vectorized training block.

**Algorithm Steps:**
The K-SVD algorithm alternates between two steps to minimize the reconstruction error subject to a sparsity constraint [S2, unofficial]:

1.  **Sparse Coding Step:** Given the current dictionary $\mathbf{D}$, compute the sparse coefficients $\mathbf{X}$ for all training blocks $\mathbf{Y}$. This is typically done using Orthogonal Matching Pursuit (OMP) [S2, unofficial]. For each block $\mathbf{y}_i$, solve:
    $$ \min_{\mathbf{x}_i} \|\mathbf{y}_i - \mathbf{D}\mathbf{x}_i\|_2^2 \quad \text{subject to} \quad \|\mathbf{x}_i\|_0 \leq L $$
    where $L$ is the maximum number of non-zero coefficients [S2, unofficial].

2.  **Dictionary Update Step:** With the sparse coefficients $\mathbf{X}$ fixed, update the dictionary $\mathbf{D}$ to minimize the reconstruction error. This is done atom by atom. For the $k$-th atom $\mathbf{d}_k$ and its corresponding coefficient row $\mathbf{r}_k^T$ (from $\mathbf{X}$):
    *   Identify the set of indices $\Omega_k$ where $\mathbf{x}_i$ has a non-zero $k$-th coefficient.
    *   Define the error matrix $\mathbf{E}_k = \mathbf{Y} - \sum_{j \neq k} \mathbf{d}_j \mathbf{r}_j^T$.
    *   Compute the updated atom $\mathbf{d}_k'$ and coefficient row $\mathbf{r}_k'^T$ using Singular Value Decomposition (SVD) on the sub-matrix of $\mathbf{E}_k$ restricted to $\Omega_k$. Specifically, if $\mathbf{E}_k^{(\Omega_k)}$ is the matrix of errors for the active blocks, let $\mathbf{E}_k^{(\Omega_k)} = \mathbf{U} \mathbf{\Sigma} \mathbf{V}^T$. Then $\mathbf{d}_k' = \mathbf{u}_1$ and $\mathbf{r}_k'^T = \sigma_1 \mathbf{v}_1^T$ [S2, implied by standard K-SVD, unofficial].
    *   Normalize $\mathbf{d}_k'$ to unit 2-norm and scale $\mathbf{r}_k'$ accordingly.

**Formulas:**
*   **Objective Function:**
    $$ \min_{\mathbf{D}, \mathbf{X}} \{ \|\mathbf{Y} - \mathbf{D}\mathbf{X}\|_F^2 \} \quad \text{subject to} \quad \forall_i \|\mathbf{x}_i\|_0 \leq L $$
    where $\|\cdot\|_F$ is the Frobenius norm [S2, unofficial].
*   **Missing Pixel Recovery:** If pixels are missing, define a subsampling matrix $\mathbf{S}$ such that $\tilde{\mathbf{y}} = \mathbf{S}\mathbf{y}$. The problem becomes:
    $$ \min_{\mathbf{x}} \|\mathbf{x}\|_0 \quad \text{s.t.} \quad \tilde{\mathbf{y}} = \tilde{\mathbf{D}}\mathbf{x} $$
    where $\tilde{\mathbf{D}} = \mathbf{S}\mathbf{D}$ is the decimated dictionary [S2, unofficial].

**Parameters:**
*   **Block Size:** $16 \times 16$ pixels for grayscale; $16 \times 16 \times 3$ for RGB [S2, unofficial].
*   **Dictionary Size:** $256 \times 1024$ for grayscale; $768 \times 2304$ for RGB [S2, unofficial]. In other contexts, $D=1024$ atoms with block size $B=8$ voxels is used [S1, unofficial].
*   **Sparsity Level ($L$ or $K$):** $L=10$ for grayscale training [S2, unofficial]; $K=6$ for Chameleon/Supernova, $K=4$ for Turbulence [S1, unofficial].
*   **Training Data:** Rendered from multiple viewpoints (e.g., 35 views for training, 1 for testing) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def k_svd_train(Y, D_rows, D_cols, L, max_iters=100):
    """
    Y: Matrix of training vectors (D_rows x N)
    D_rows: Number of rows in dictionary (atom dimension)
    D_cols: Number of columns in dictionary (number of atoms)
    L: Maximum sparsity level
    max_iters: Maximum number of alternating optimization iterations
    Returns: Dictionary D (D_rows x D_cols)
    """
    # Initialize dictionary D randomly or using PCA
    D = initialize_dictionary(D_rows, D_cols)
    
    for iteration in range(max_iters):
        # Step 1: Sparse Coding
        X = sparse_coding(Y, D, L)
        
        # Step 2: Dictionary Update
        for k in range(D_cols):
            # Find indices where k-th coefficient is non-zero
            active_indices = find_nonzero_indices(X[k, :])
            
            if len(active_indices) == 0:
                # Re-initialize atom if not used
                D[:, k] = random_unit_vector(D_rows)
                continue
                
            # Compute error matrix excluding the k-th atom's contribution
            # E = Y - D * X + D[:, k] * X[k, :]
            # We only need the error for active indices
            E = Y[:, active_indices] - D @ X[:, active_indices] + D[:, k] * X[k, active_indices]
            
            # Perform SVD on the error matrix restricted to active indices
            U, S, Vt = svd(E, full_matrices=False)
            
            # Update atom and coefficients
            D[:, k] = U[:, 0]
            X[k, active_indices] = S[0] * Vt[0, :]
            
            # Normalize atom to unit 2-norm
            norm = l2_norm(D[:, k])
            D[:, k] = D[:, k] / norm
            X[k, active_indices] = X[k, active_indices] * norm
            
    return D

def sparse_coding(Y, D, L):
    """
    Y: Matrix of training vectors (D_rows x N)
    D: Dictionary (D_rows x D_cols)
    L: Sparsity level
    Returns: Coefficient matrix X (D_cols x N)
    """
    N = Y.shape[1]
    X = np.zeros((D.shape[1], N))
    
    for i in range(N):
        y = Y[:, i]
        # Use OMP to find sparse representation
        x = omp(y, D, L)
        X[:, i] = x
        
    return X

def omp(y, D, L):
    """
    Orthogonal Matching Pursuit
    y: Signal vector
    D: Dictionary
    L: Sparsity level
    Returns: Sparse coefficient vector x
    """
    N_atoms = D.shape[1]
    x = np.zeros(N_atoms)
    residual = y.copy()
    indices = []
    
    for _ in range(L):
        # Find atom most correlated with residual
        correlations = np.abs(D.T @ residual)
        idx = np.argmax(correlations)
        
        if idx in indices:
            break
            
        indices.append(idx)
        
        # Solve least squares problem to update coefficients
        D_sub = D[:, indices]
        coeffs, _, _, _ = lstsq(D_sub, y)
        
        # Update residual
        residual = y - D_sub @ coeffs
        
    # Set coefficients for selected indices
    for i, idx in enumerate(indices):
        x[idx] = coeffs[i]
        
    return x
```

## Subtopics

### [Orthogonal Matching Pursuit (OMP)](1999-orthogonal-matching-pursuit-omp.md)
A greedy algorithm for sparse coding that selects dictionary atoms sequentially to find an approximate sparse solution $\mathbf{x}$ for the problem $\min \|\mathbf{x}\|_0$ subject to $\|\mathbf{y} - \mathbf{D}\mathbf{x}\|_2 \leq \epsilon$. It is used in the sparse coding step of K-SVD [1999-orthogonal-matching-pursuit-omp.md].

### Compressive Volume Rendering with Dictionary Learning
A variant that treats rendered images as collections of square pixel blocks to apply sparse coding and dictionary learning for recovery or compression. It specifically trains dictionaries for grayscale images first, then extends to color images using $16 \times 16 \times 3$ blocks to capture channel correlations [S2, unofficial].

## Operating characteristics
**Time Complexity:** The training time is significant. For a $256 \times 1024$ dictionary on grayscale images, training took 7.1 minutes on a single-threaded Intel Core i7-4770K CPU [S2, unofficial]. RGB training took 47.3 minutes [S2, unofficial]. Encoding (sparse coding) is fast using OMP.
**Memory Complexity:** The dictionary size is $D_{rows} \times D_{cols}$. For grayscale, $256 \times 1024$ floats; for RGB, $768 \times 2304$ floats [S2, unofficial].
**Performance:** Achieves very low bits-per-sample (bps): 0.25 bps for Chameleon ($1024^2$), 0.07 bps for Supernova (1080p), and 0.20 bps for Turbulence ($512^4$) [S1, unofficial].
**Strengths:** Outperforms fixed dictionaries (like DCT) in reconstruction quality (PSNR) for missing pixel recovery and compression [S2, unofficial]. Captures complex structures in volume renderings better than linear transforms [S2, unofficial].
**Limitations:** Requires training data. Encoding is slower than fixed-basis methods due to the iterative sparse coding step. Sensitive to the choice of sparsity level $L$ and dictionary size [S2, unofficial].

## Background
K-SVD was introduced by M. Aharon, M. Elad, and A. Bruckstein in 2006 [MAB06, cited in S2]. It is a generalization of K-Means clustering, where the cluster centers (dictionary atoms) are also learned. In volume rendering, it was applied to compress and recover volume rendered images, treating them as collections of pixel blocks [S2, unofficial]. It is used in scenarios requiring high compression ratios and artifact reduction, such as real-time deblocked GPU rendering [S1, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
