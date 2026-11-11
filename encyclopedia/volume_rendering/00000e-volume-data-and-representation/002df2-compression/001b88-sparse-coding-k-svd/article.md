# Sparse Coding (K-SVD)

Also called: Compressive Volume Rendering via Dictionary Learning, Dictionary Learning for Volume Rendering, K-SVD, K-SVD Dictionary Learning

## Overview
Sparse Coding with K-SVD compresses volume rendering data by learning an overcomplete dictionary from rendered images and representing image blocks as sparse linear combinations of dictionary atoms [S2, unofficial]. It differs from prescribed basis methods (like DCT) by adapting the basis to the specific statistical properties of the volume rendering output, yielding higher reconstruction fidelity (PSNR) for the same or lower bitrates, particularly in scenarios involving missing data or high compression ratios [S2, unofficial].

## How it works
**Inputs:** A set of training images rendered from the volume dataset (typically 3D scalar fields rendered via ray-marching) [S2, unofficial].
**Outputs:** A learned dictionary $D$ and, for a target image, a sparse coefficient matrix $X$ [S2, unofficial].

**Data Structures:**
1.  **Training Matrix $Y$:** A matrix where each column is a flattened training block [S2, unofficial].
    *   For grayscale: $Y \in \mathbb{R}^{M \times N_{train}}$, where $M = B^2$ ($B$ is block side length) [S2, unofficial].
    *   For RGB: $Y \in \mathbb{R}^{3M \times N_{train}}$, where blocks are $B \times B \times 3$ and linearized (e.g., R channel first, then G, then B) [S2, unofficial].
2.  **Dictionary $D$:** An overcomplete matrix $D \in \mathbb{R}^{M \times K}$ (grayscale) or $D \in \mathbb{R}^{3M \times K'}$ (RGB), where $K, K'$ is the number of atoms [S2, unofficial]. Columns are normalized to unit 2-norm [S2, unofficial].
3.  **Coefficient Matrix $X$:** $X \in \mathbb{R}^{K \times N_{train}}$ (or $K' \times N_{train}$), where each column $x_i$ has at most $L$ non-zero entries [S2, unofficial].

**Parameters:**
*   $B$: Block size. Typical: $16 \times 16$ pixels [S2, unofficial]. For 3D voxel blocks in some implementations, $B=8$ voxels [S1, unofficial].
*   $L$: Target sparsity (max non-zeros per coefficient vector). Typical: $L=10$ for grayscale [S2, unofficial]; $K=6$ or $4$ for specific datasets [S1, unofficial].
*   $K$: Number of dictionary atoms. Typical: $1024$ [S1, unofficial][S2, unofficial]; $2304$ for RGB [S2, unofficial].

**Algorithm Steps:**

1.  **Training Data Generation:**
    *   Render the volume dataset from multiple viewpoints (e.g., 35 views) using a standard volume renderer [S2, unofficial].
    *   Convert color images to grayscale if training a grayscale dictionary, or keep RGB for color dictionary [S2, unofficial].
    *   Extract non-overlapping blocks of size $B \times B$ (grayscale) or $B \times B \times 3$ (RGB) from the training images [S2, unofficial].
    *   Form matrix $Y$ by stacking these blocks as columns [S2, unofficial].

2.  **Dictionary Learning (K-SVD):**
    Initialize $D$ (e.g., randomly or using DCT atoms) [G]. Iterate until convergence:
    
    *   **Sparse Coding Step:** For each column $y_i$ of $Y$, compute coefficient vector $x_i$ using Orthogonal Matching Pursuit (OMP):
        $$ \min_{x_i} \|x_i\|_0 \quad \text{subject to} \quad \|y_i - D x_i\|_2 \leq \epsilon $$
        where $\epsilon$ is an error threshold [S2, unofficial]. OMP selects atoms greedily based on inner products and updates coefficients via least squares [S2, unofficial].
        
    *   **Dictionary Update Step:** For each atom $d_j$ in $D$:
        1.  Identify the set of indices $I_j$ of training vectors that use atom $j$ (i.e., $x_{i,j} \neq 0$ for $i \in I_j$) [G].
        2.  Compute the error matrix $E_j = Y_{I_j} - D X_{I_j} + d_j x_{j, I_j}^T$, where $Y_{I_j}$ contains the corresponding training vectors, $X_{I_j}$ contains the corresponding coefficients, and $x_{j, I_j}$ is the row of coefficients for atom $j$ [G].
        3.  Perform Singular Value Decomposition (SVD) on $E_j = U \Sigma V^T$ [G].
        4.  Update $d_j = U(:,1)$ and $x_{j, I_j}^T = \Sigma(1,1) V^T(:,1)$ [S2, unofficial]. This minimizes the reconstruction error for the vectors using atom $j$ while maintaining sparsity [S2, unofficial].

3.  **Encoding (Compression):**
    *   For a target image, extract blocks.
    *   For each block $y$, solve $\min \|x\|_0$ s.t. $\|y - Dx\|_2 \leq \epsilon$ using OMP to get $x$ [S2, unofficial].
    *   Store the indices of the non-zero coefficients and their values [G].

4.  **Decoding (Reconstruction):**
    *   Reconstruct block $\hat{y} = D x$.
    *   Assemble blocks into the full image.

**Missing Pixel Recovery Variant:**
If pixels are missing, define a subsampling matrix $S$ such that $\tilde{y} = S y$ [S2, unofficial]. The problem becomes:
$$ \min_{\mathbf{x}} \|\mathbf{x}\|_0 \quad \text{s.t.} \quad \tilde{\mathbf{y}} = \tilde{\mathbf{D}}\mathbf{x} $$
where $\tilde{\mathbf{D}} = S D$ is the decimated dictionary [S2, unofficial]. Solve via OMP to recover $x$, then approximate $y$ as $D x$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from numpy.linalg import svd, lstsq

def k_svd_train(Y, D, L, max_iters=100):
    """
    Train dictionary D using K-SVD on training data Y.
    Y: (M, N) matrix of training vectors.
    D: (M, K) initial dictionary.
    L: max sparsity (number of non-zeros).
    """
    N = Y.shape[1]
    for iteration in range(max_iters):
        # 1. Sparse Coding Step
        X = np.zeros((D.shape[1], N))
        for i in range(N):
            y_i = Y[:, i]
            # OMP implementation
            x_i = omp(y_i, D, L)
            X[:, i] = x_i
            
        # 2. Dictionary Update Step
        for j in range(D.shape[1]):
            # Indices of vectors using atom j
            idx = np.where(X[j, :] != 0)[0]
            if len(idx) == 0:
                continue
                
            # Error matrix excluding atom j's contribution
            # E_j = Y[:, idx] - D @ X[:, idx] + D[:, j] @ X[j, idx]
            # This isolates the error that atom j is responsible for
            E_j = Y[:, idx] - D @ X[:, idx] + D[:, j] @ X[j, idx]
            
            # SVD to update d_j and x_j
            U, S, Vt = svd(E_j, full_matrices=False)
            d_new = U[:, 0]
            x_new = S[0] * Vt[0, :]
            
            D[:, j] = d_new
            X[j, idx] = x_new
            
    return D, X

def omp(y, D, L):
    """
    Orthogonal Matching Pursuit.
    y: signal (M,)
    D: dictionary (M, K)
    L: sparsity level
    """
    K = D.shape[1]
    x = np.zeros(K)
    residual = y.copy()
    indices = []
    
    for _ in range(L):
        # Compute inner products
        corrs = np.abs(D.T @ residual)
        # Select atom with largest correlation
        idx = np.argmax(corrs)
        if idx in indices:
            break
        indices.append(idx)
        
        # Least squares to update coefficients
        D_sub = D[:, indices]
        coeffs, _, _, _ = lstsq(D_sub, y, rcond=None)
        
        # Update residual
        residual = y - D_sub @ coeffs
        
    # Finalize x
    x[indices] = coeffs
    return x
```

## Subtopics

### [CP Decomposition](2255-cp-decomposition.md)
CP Decomposition factorizes a tensor into a sum of rank-one components (outer products of vectors). It offers higher compactness than Tucker Decomposition but requires more components for complex scenes, leading to higher reconstruction costs. Choose when memory footprint is the primary constraint and scene complexity allows for higher rank.

### [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md)
Tucker Decomposition generalizes SVD to tensors, producing a core tensor and factor matrices. It preserves multilinear structure and orthonormality, making it suitable for global basis generation and multiresolution rendering. Choose when preserving structural integrity and enabling multiresolution access is more important than maximum compactness.

### [Karhunen-Loeve Transform (KLT)](5365-karhunen-loeve-transform-klt.md)
KLT is a linear decorrelation technique using data-dependent eigenvectors of the covariance matrix. It provides superior decorrelation compared to DCT but lacks fast transforms, shifting computation to offline preprocessing. Choose when offline preprocessing cost is acceptable and optimal decorrelation is required.

### [Sparse Representation (Dictionary Based)](5366-sparse-representation-dictionary-based.md)
Encodes blocks as sparse linear combinations of dictionary atoms. It differs from Vector Quantization by allowing multiple atoms per block, offering higher fidelity at higher encoding complexity. K-SVD is the primary method for learning such dictionaries.

#### Orthogonal Matching Pursuit (OMP)
A greedy sparse coding algorithm that selects atoms sequentially based on inner products and updates coefficients via least squares [S2, unofficial]. It offers better orthogonality handling and faster convergence than Matching Pursuit (MP) [S2, unofficial]. Use as the solver for the sparse coding step in K-SVD [S2, unofficial].

#### Compressive Volume Rendering with Dictionary Learning
A technique treating rendered images as pixel blocks for sparse coding recovery/compression [S2, unofficial]. It trains dictionaries on grayscale first, then extends to RGB by linearizing $B \times B \times 3$ blocks [S2, unofficial]. Use when recovering missing pixels or compressing rendered images is the goal [S2, unofficial].

## Operating characteristics
**Time Complexity:** Dictionary training is iterative and expensive. Training a $256 \times 1024$ dictionary on grayscale images took 7.1 minutes on a single-core CPU [S2, unofficial]. RGB training took 47.3 minutes [S2, unofficial]. Encoding is dominated by OMP, which is linear in dictionary size per block.
**Memory Complexity:** Dictionary size $D$ (e.g., $1024$ atoms of dimension $256$) is stored in memory. Coefficients are sparse.
**Performance:** Achieves very low bits-per-sample (bps): 0.25 bps for Chameleon, 0.07 bps for Supernova, 0.20 bps for Turbulence [S1, unofficial]. SSIM scores (with deblocking) range from 0.72 to 0.84 [S1, unofficial].
**Strengths:** High compression ratios; superior reconstruction quality (PSNR) compared to DCT dictionaries, especially with missing data (31.2 dB vs 22.97 dB PSNR with 70% missing pixels) [S2, unofficial].
**Limitations:** High training time; encoding is slower than fixed-basis methods; sensitive to training data distribution.
**When to use:** Real-time exploration of time-varying datasets during animation playback with pre-loaded compressed data in GPU memory [S1, unofficial].
**When not to use:** When encoding latency is critical or training data is unavailable.

## Background
Originated from dictionary learning techniques in signal processing, adapted for volume rendering by treating rendered images as collections of blocks [S2, unofficial]. The K-SVD algorithm was introduced by Elad and Aharon [MAB06] as a generalization of K-Means. In volume rendering, it was applied to compress and recover rendered images, demonstrating advantages over prescribed bases like DCT [S2, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
