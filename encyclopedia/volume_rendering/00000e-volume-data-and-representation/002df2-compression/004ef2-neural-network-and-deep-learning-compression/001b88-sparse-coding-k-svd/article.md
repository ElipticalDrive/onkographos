# Sparse Coding (K-SVD)

Also called: Compressive Volume Rendering via Dictionary Learning, Dictionary Learning for Volume Rendering, K-SVD, K-SVD Dictionary Learning

## Overview
Sparse Coding (K-SVD) compresses volume rendering data by learning an overcomplete dictionary from training images and representing image blocks as sparse linear combinations of dictionary atoms [S2, unofficial]. It differs from fixed-basis methods like DCT by adapting the basis to the specific statistical structure of the volume data, yielding higher reconstruction fidelity (PSNR) for the same redundancy level [S2, unofficial]. It builds on [Sparse Representation (Dictionary Based)](5366-sparse-representation-dictionary-based.md) by jointly optimizing the dictionary and the sparse coefficients [S2, unofficial].

## How it works
**Inputs:** A set of training images rendered from the volume dataset (e.g., chest CT from multiple viewpoints) [S2, unofficial].
**Outputs:** A learned dictionary $\mathbf{D}$ and sparse coefficient matrices $\mathbf{X}$ for new input blocks [S2, unofficial].

**Data Structures:**
*   **Dictionary $\mathbf{D}$:** Matrix of size $M \times N$, where $M$ is the dimension of a block (e.g., $16 \times 16 = 256$ for grayscale, $16 \times 16 \times 3 = 768$ for RGB) and $N$ is the number of atoms (e.g., 1024 or 2304) [S2, unofficial]. Columns are normalized to unit 2-norm [S2, unofficial].
*   **Training Matrix $\mathbf{Y}$:** Matrix where each column is a flattened training block [S2, unofficial].
*   **Coefficient Matrix $\mathbf{X}$:** Matrix where each column $\mathbf{x}_i$ contains the sparse coefficients for the corresponding block in $\mathbf{Y}$ [S2, unofficial].

**Algorithm Steps:**
The K-SVD algorithm alternates between two steps to minimize the reconstruction error subject to a sparsity constraint [S2, unofficial]:

1.  **Sparse Coding Step:** Given the current dictionary $\mathbf{D}$, compute the sparse coefficients $\mathbf{X}$ for all training blocks in $\mathbf{Y}$ using a pursuit algorithm such as Orthogonal Matching Pursuit (OMP) [S2, unofficial]. For each column $\mathbf{y}_i$ of $\mathbf{Y}$, find $\mathbf{x}_i$ such that:
    $$ \min_{\mathbf{x}_i} \|\mathbf{y}_i - \mathbf{D}\mathbf{x}_i\|_2^2 \quad \text{subject to} \quad \|\mathbf{x}_i\|_0 \leq L $$
    where $L$ is the maximum number of non-zero coefficients (sparsity level) [S2, unofficial].

2.  **Dictionary Update Step:** With $\mathbf{X}$ fixed, update the dictionary $\mathbf{D}$ to reduce the reconstruction error [S2, unofficial]. The algorithm updates atoms one by one to better fit the training vectors, ensuring a reduction or no change in the Mean Squared Error (MSE) without violating the sparsity constraint [S2, unofficial].

**Parameters:**
*   **Block Size:** $16 \times 16$ pixels for grayscale; $16 \times 16 \times 3$ for RGB [S2, unofficial].
*   **Dictionary Size ($N$):** 1024 atoms for grayscale ($M=256$); 2304 atoms for RGB ($M=768$) [S2, unofficial].
*   **Sparsity Level ($L$ or $K$):** 10 for grayscale training [S2, unofficial]; 6 for Chameleon/Supernova datasets, 4 for Turbulence dataset in compression contexts [S1, unofficial].
*   **Initialization:** The dictionary can be initialized randomly or using a prescribed dictionary (e.g., DCT) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Convergence:** The algorithm converges if the sparse coding step provides the best approximation with no more than $L$ non-zero entries [S2, unofficial].
*   **Color Correlation:** Training separate dictionaries for RGB channels ignores inter-channel correlations. Joint training on 3D RGB blocks captures these correlations but increases dictionary size and training time [S2, unofficial].
*   **Missing Pixels:** For recovery from missing data, the dictionary must be decimated by the subsampling matrix $S$ ($\tilde{\mathbf{D}} = S\mathbf{D}$) before solving the sparse coding problem [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def k_svd(Y, D, L, max_iterations):
    """
    Y: Training data matrix (M x NumSamples)
    D: Dictionary matrix (M x N)
    L: Sparsity level (max non-zeros per coefficient vector)
    max_iterations: Number of alternating steps
    """
    M = len(Y[0])
    N = len(D)
    NumSamples = len(Y)
    X = [[0.0] * NumSamples for _ in range(N)]
    
    for iteration in range(max_iterations):
        # Step 1: Sparse Coding
        X = sparse_coding(Y, D, L)
        
        # Step 2: Dictionary Update
        for k in range(N):
            indices = [i for i in range(NumSamples) if X[k][i] != 0]
            if not indices:
                continue
                
            # Compute residual excluding k-th atom
            E_k = [row[:] for row in Y]
            for j in range(N):
                if j != k:
                    for i in range(NumSamples):
                        for m in range(M):
                            E_k[m][i] -= D[m][j] * X[j][i]
            
            # Extract columns corresponding to non-zero coefficients
            E_k_sub = [[E_k[m][i] for i in indices] for m in range(M)]
            
            # Standard linear algebra operation: SVD
            U, Sigma, Vt = svd(E_k_sub)
            
            # Update atom and coefficients
            d_new = [U[m][0] for m in range(M)]
            r_new = [Sigma[0] * Vt[0][i] for i in range(len(indices))]
            
            for m in range(M):
                D[m][k] = d_new[m]
            for idx, i in enumerate(indices):
                X[k][i] = r_new[idx]
                
            # Normalize atom to unit 2-norm
            norm_d = sum(v**2 for v in d_new)**0.5
            for m in range(M):
                D[m][k] /= norm_d
            for idx in range(len(indices)):
                X[k][indices[idx]] *= norm_d
                
    return D, X

def sparse_coding(Y, D, L):
    M = len(Y[0])
    N = len(D)
    NumSamples = len(Y)
    X = [[0.0] * NumSamples for _ in range(N)]
    
    for i in range(NumSamples):
        y = [Y[m][i] for m in range(M)]
        residual = y[:]
        support = []
        coeffs = []
        
        for _ in range(L):
            correlations = [sum(D[m][j] * residual[m] for m in range(M)) for j in range(N)]
            atom_idx = max(range(N), key=lambda j: abs(correlations[j]))
            
            if atom_idx in support:
                break
                
            support.append(atom_idx)
            
            # Standard linear algebra operation: Least Squares solve
            coeffs = lstsq(D, support, y)
            
            residual = [y[m] - sum(D[m][support[s]] * coeffs[s] for s in range(len(support))) for m in range(M)]
            
        for idx, atom_idx in enumerate(support):
            X[atom_idx][i] = coeffs[idx]
            
    return X

def svd(A):
    # Standard linear algebra routine to compute U, Sigma, Vt such that A = U @ diag(Sigma) @ Vt
    pass

def lstsq(D, support, y):
    # Standard linear algebra routine to solve min ||D[:, support] @ x - y||_2
    pass
```

## Subtopics

### [Orthogonal Matching Pursuit (OMP)](1999-orthogonal-matching-pursuit-omp.md)
Greedy algorithm for sparse coding, used in K-SVD's coding step.

### [CP Decomposition](2255-cp-decomposition.md)
Tensor factorization alternative for volume compression.

### [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md)
Tensor factorization alternative for volume compression.

### [Karhunen-Loeve Transform (KLT)](5365-karhunen-loeve-transform-klt.md)
Linear decorrelation transform, related to dictionary learning but with fixed data-derived basis.

### [Sparse Representation (Dictionary Based)](5366-sparse-representation-dictionary-based.md)
General framework for sparse coding with fixed or learned dictionaries.

### Compressive Volume Rendering with Dictionary Learning
A specific application of K-SVD for volume rendering that treats rendered images as collections of $16 \times 16$ pixel blocks [S2, unofficial]. It trains dictionaries on grayscale images first, then extends to RGB by training on 3D blocks ($16 \times 16 \times 3$) to capture color correlations [S2, unofficial]. It is used for recovering images with missing pixels and compression [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is iterative and depends on the number of samples, dictionary size, and sparsity [S2, unofficial]. Training a $256 \times 1024$ dictionary on grayscale images took 7.1 minutes on a single-threaded Intel Core i7-4770K [S2, unofficial]. RGB training ($768 \times 2304$) took 47.3 minutes [S2, unofficial].
*   **Memory Complexity:** Requires storing the dictionary ($M \times N$) and the coefficient matrix ($N \times \text{NumSamples}$) during training [G]. For compression, only the dictionary and sparse coefficients for each block need to be stored [G].
*   **Performance:** Achieves bits-per-sample (bps) of 0.25 (Chameleon), 0.07 (Supernova), and 0.20 (Turbulence) [S1, unofficial]. Deblocked SSIM scores are 0.72 (Chameleon), 0.76 (Supernova), and 0.84 (Turbulence) [S1, unofficial]. For missing pixel recovery (70% missing), trained dictionary PSNR is 31.2 dB vs. 22.97 dB for DCT [S2, unofficial].
*   **Strengths:** Adapts to data statistics, better reconstruction quality than fixed bases (DCT) for the same redundancy, handles color correlations effectively [S2, unofficial].
*   **Limitations:** High training time, requires careful parameter tuning (sparsity, dictionary size), encoding is slower than fixed-basis methods due to iterative sparse coding [S2, unofficial].

## Background
K-SVD was introduced by Elad and Aharon [MAB06] as a generalization of K-Means for dictionary learning [S2, unofficial]. It has been applied to volume rendering compression and recovery by treating rendered images as collections of pixel blocks [S2, unofficial]. The method is used in research contexts for real-time deblocked GPU rendering of compressed volumes [S1, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
