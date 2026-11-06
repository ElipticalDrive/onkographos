# Sparse Coding (K-SVD)

Also called: Compressive Volume Rendering via Dictionary Learning, Dictionary Learning for Volume Rendering, K-SVD, K-SVD Dictionary Learning

## Overview
Sparse Coding (K-SVD) compresses volume rendering data by learning an overcomplete dictionary from rendered images and representing image blocks as sparse linear combinations of dictionary atoms [S1, S2, unofficial]. It sets itself apart from prescribed basis methods like [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md) by adapting the basis to the specific statistical structure of the volume data, and from [Vector Quantization for Volumetric Radiance Fields](1870-vector-quantization-for-volumetric-radiance-fields.md) by allowing multiple atoms per block, which reduces reconstruction error at the cost of higher encoding complexity [S2, unofficial].

## How it works
**Inputs:** A set of training images rendered from the volume dataset (e.g., chest CT from multiple viewpoints) [S2, unofficial].
**Outputs:** A learned dictionary $\mathbf{D}$ and sparse coefficient matrices $\mathbf{X}$ for the training data [S2, unofficial].

**Data Structures:**
*   **Dictionary $\mathbf{D}$:** A matrix of size $M \times N$, where $M$ is the dimension of a training block (e.g., $16 \times 16 = 256$ for grayscale, $16 \times 16 \times 3 = 768$ for RGB) and $N$ is the number of atoms (e.g., 1024 or 2048) [S1, S2, unofficial]. Columns are normalized to unit 2-norm.
*   **Coefficient Matrix $\mathbf{X}$:** A matrix of size $N \times K$, where $K$ is the number of training blocks. Each column $\mathbf{x}_i$ contains the sparse coefficients for the $i$-th block.
*   **Training Data Matrix $\mathbf{Y}$:** A matrix of size $M \times K$, where each column is a flattened training block [S2, unofficial].

**Parameters:**
*   **Block Size:** $16 \times 16$ pixels for grayscale; $16 \times 16 \times 3$ for RGB [S2, unofficial].
*   **Sparsity Level ($L$ or $K$):** Maximum number of non-zero coefficients per block. Typical values: $L=10$ for grayscale training [S2, unofficial]; $K=6$ for Chameleon/Supernova datasets, $K=4$ for Turbulence [S1, unofficial].
*   **Dictionary Size ($N$):** Typical values: 1024 [S1, S2, unofficial], 2048 [S1, unofficial], or 2304 for RGB [S2, unofficial].

**Algorithm Steps:**
1.  **Training Data Generation:** Render the volume from multiple viewpoints (e.g., 35 views) using a transfer function. Convert to grayscale if training a grayscale dictionary. Extract non-overlapping $16 \times 16$ blocks to form $\mathbf{Y}$ [S2, unofficial].
2.  **Initialization:** Initialize $\mathbf{D}$ randomly or using a prescribed basis (e.g., DCT) [S2, unofficial].
3.  **Sparse Coding (OMP):** For each column $\mathbf{y}_i$ in $\mathbf{Y}$, find sparse $\mathbf{x}_i$ using Orthogonal Matching Pursuit (OMP) such that $\|\mathbf{y}_i - \mathbf{D}\mathbf{x}_i\|_2 \leq \epsilon$ and $\|\mathbf{x}_i\|_0 \leq L$ [S2, unofficial].
4.  **Dictionary Update (K-SVD):** Update $\mathbf{D}$ to minimize the reconstruction error $\|\mathbf{Y} - \mathbf{D}\mathbf{X}\|_F^2$ subject to sparsity constraints [S2, unofficial]. This is done atom-by-atom:
    *   For atom $j$, identify the set of indices $\mathcal{I}_j$ where $\mathbf{x}_i$ uses atom $j$ (i.e., $x_{j,i} \neq 0$).
    *   Compute the error matrix $\mathbf{E}_j = \mathbf{Y} - \sum_{k \neq j} \mathbf{d}_k \mathbf{x}_k^T$ (where $\mathbf{d}_k$ is the $k$-th column of $\mathbf{D}$) [S2, unofficial].
    *   Perform SVD on $\mathbf{E}_j$ restricted to columns in $\mathcal{I}_j$: $\mathbf{E}_j \mathbf{1}_{\mathcal{I}_j} = \mathbf{U} \mathbf{\Sigma} \mathbf{V}^T$.
    *   Update $\mathbf{d}_j = \mathbf{u}_1$ and scale the corresponding row of $\mathbf{X}$ by $\sigma_1$ [S2, unofficial].
5.  **Iteration:** Repeat steps 3 and 4 until convergence or a fixed number of iterations [S2, unofficial].

**Missing Pixel Recovery:**
To recover missing pixels in a test image:
1.  Form the subsampled signal $\tilde{\mathbf{y}}$ by removing missing components.
2.  Define subsampling matrix $\mathbf{S}$ such that $\tilde{\mathbf{y}} = \mathbf{S}\mathbf{y}$.
3.  Compute decimated dictionary $\tilde{\mathbf{D}} = \mathbf{S}\mathbf{D}$.
4.  Solve $\min \|\mathbf{x}\|_0 \text{ s.t. } \tilde{\mathbf{y}} = \tilde{\mathbf{D}}\mathbf{x}$ via OMP [S2, unofficial].
5.  Reconstruct $\mathbf{y} \approx \mathbf{D}\mathbf{x}$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def k_svd_train(Y, D, L, max_iter):
    """
    Train dictionary D using K-SVD on data Y.
    Y: (M, K) matrix of training blocks
    D: (M, N) initial dictionary
    L: sparsity level
    max_iter: number of iterations
    """
    M, N = D.shape
    for iteration in range(max_iter):
        # Step 1: Sparse Coding using OMP
        X = np.zeros((N, Y.shape[1]))
        for i in range(Y.shape[1]):
            y_i = Y[:, i]
            x_i = omp(y_i, D, L)
            X[:, i] = x_i
        
        # Step 2: Dictionary Update
        for j in range(N):
            # Find indices where atom j is used
            idx = np.where(X[j, :] != 0)[0]
            if len(idx) == 0:
                continue
            
            # Compute error excluding atom j
            # E_j = Y - sum_{k!=j} d_k * x_k^T
            # This is equivalent to Y - D * X + d_j * x_j^T
            E_j = Y - D @ X + D[:, j:j+1] @ X[j:j+1, :]
            
            # Restrict E_j to columns in idx
            E_j_sub = E_j[:, idx]
            
            # SVD of E_j_sub
            U, S, Vt = np.linalg.svd(E_j_sub, full_matrices=False)
            
            # Update atom d_j
            D[:, j] = U[:, 0]
            
            # Update coefficient x_j for all used blocks
            # x_j_new = x_j_old * sigma_1 / ||d_j|| (but d_j is unit norm)
            # Actually, we scale the row of X corresponding to j
            # The update is: x_j = sigma_1 * v_1^T
            X[j, idx] = S[0] * Vt[0, :]
            
            # Normalize D column j (should already be unit norm from U[:,0])
            D[:, j] /= np.linalg.norm(D[:, j])
            
    return D, X

def omp(y, D, L):
    """
    Orthogonal Matching Pursuit
    y: signal (M,)
    D: dictionary (M, N)
    L: sparsity level
    """
    x = np.zeros(D.shape[1])
    residual = y.copy()
    indices = []
    
    for _ in range(L):
        # Compute inner products
        correlations = np.abs(D.T @ residual)
        # Select atom with max correlation
        idx = np.argmax(correlations)
        if idx in indices:
            break
        indices.append(idx)
        
        # Least squares to update coefficients
        D_sub = D[:, indices]
        coeffs = np.linalg.lstsq(D_sub, y, rcond=None)[0]
        x[indices] = coeffs
        
        # Update residual
        residual = y - D_sub @ coeffs
        
    return x
```

## Subtopics

### [CP Decomposition](2255-cp-decomposition.md)
CP Decomposition is a tensor factorization technique that represents a tensor as a sum of rank-one components, where each component is the outer product of vectors corresponding to each mode of the tensor. In the context of volume rendering, it is used to factorize the 3D (or 4D) radiance field tensor into compact vectors, significantly reducing the memory footprint compared to dense voxel grids. Unlike [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md), which uses a core tensor multiplied by factor matrices, CP uses only vectors, offering higher compactness but requiring more components to model complex scenes, leading to higher computational costs during reconstruction.

### [Tucker Decomposition (HOSVD)](5352-tucker-decomposition-hosvd.md)
Tucker Decomposition (specifically via Higher-Order SVD) approximates an $N$-dimensional input tensor $\mathcal{A}$ by a product of a core tensor $\mathcal{B}$ and $N$ factor matrices $\mathbf{U}^{(n)}$, capturing the multi-linear structure of volume data. It sets itself apart from sibling methods like CP Decomposition by providing a compact model with rank-reducible bases that capture non-axis aligned features, unlike fixed-basis transforms. It serves as the foundational compression step for Tensor Approximation, enabling multiresolution volume rendering by learning global bases.

### [Karhunen-Loeve Transform (KLT)](5365-karhunen-loeve-transform-klt.md)
The Karhunen-Loeve Transform (KLT) is a linear decorrelation technique used in volume data compression to remove redundancy by estimating the covariance matrix and computing its eigenvectors. It is closely related to Principal Component Analysis (PCA) and provides superior decorrelation quality compared to the Discrete Cosine Transform (DCT). Unlike orthogonal basis methods like DCT, KLT bases are data-dependent and not mutually independent, which prevents the use of fast forward and inverse transforms. To mitigate the computational cost of these transforms, KLT is often combined with Vector Quantization (VQ) and block-wise decomposition, shifting the heavy reconstruction work to an offline preprocessing stage.

### [Sparse Representation (Dictionary Based)](5366-sparse-representation-dictionary-based.md)
Sparse Representation (Dictionary Based) encodes volume data blocks by finding a sparse linear combination of atoms from a pre-computed dictionary, solving an optimization problem to minimize reconstruction error under a sparsity constraint. It differs from Vector Quantization (VQ) by allowing each input block to be represented by a linear combination of multiple dictionary elements rather than a single nearest-neighbor prototype, offering higher fidelity at the cost of significantly more complex encoding.

### Orthogonal Matching Pursuit (OMP)
Orthogonal Matching Pursuit (OMP) is a greedy algorithm for sparse coding that selects dictionary atoms sequentially to find an approximate sparse solution $\mathbf{x}$ for the problem $\min \|\mathbf{x}\|_0 \text{ s.t. } \|\mathbf{y} - \mathbf{D}\mathbf{x}\|_2 \leq \epsilon$ [S2, unofficial]. It computes inner products between the signal $\mathbf{y}$ and dictionary columns $\mathbf{D}$, then deploys least squares solvers to update the solution coefficients [S2, unofficial]. The target sparsity $L$ limits the number of non-zero entries in $\mathbf{x}_i$ [S2, unofficial]. OMP is preferred over Matching Pursuit (MP) because it offers improvements on orthogonality and faster convergence [S2, unofficial].

### Compressive Volume Rendering with Dictionary Learning
Compressive Volume Rendering with Dictionary Learning treats rendered images as collections of square pixel blocks to apply sparse coding and dictionary learning for recovery or compression [S2, unofficial]. It typically trains a dictionary for grayscale images first, then extends to color images based on experimental insights [S2, unofficial]. The block size is typically $16 \times 16$ pixels [S2, unofficial].

## Operating characteristics
**Time Complexity:** Dictionary training is computationally expensive, taking 7.1 minutes for grayscale (256x1024 dict) and 47.3 minutes for RGB (768x2304 dict) on a single-threaded Intel Core i7-4770K CPU [S2, unofficial]. Encoding (sparse coding) is faster but still requires iterative pursuit algorithms like OMP [S2, unofficial].

**Memory Complexity:** The dictionary size is $M \times N$. For grayscale, $256 \times 1024$ floats; for RGB, $768 \times 2304$ floats [S2, unofficial]. Coefficients are stored sparsely.

**Measured Performance:**
*   **Bits-per-sample (bps):** 0.25 (Chameleon $1024^2$), 0.07 (Supernova 1080p), 0.20 (Turbulence $512^4$) [S1, unofficial].
*   **Reconstruction Quality:** For 70% missing pixels, trained dictionary PSNR is 31.2 dB vs. 22.97 dB for DCT dictionary [S2, unofficial].
*   **SSIM:** Deblocked SSIM scores are 0.72 (Chameleon), 0.76 (Supernova), 0.84 (Turbulence) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Outperforms prescribed dictionaries (like DCT) in reconstruction quality (PSNR) for missing pixel recovery and compression [S2, unofficial]. Captures color correlations in RGB blocks [S2, unofficial].
*   **Limitations:** High training time. Requires careful parameter tuning (block size, sparsity, dictionary size) [S1, S2, unofficial]. Convergence depends on perfect sparse coding [S2, unofficial].

**When to use:** When high compression ratios are needed and offline training time is acceptable, especially for datasets with repetitive structures or specific color distributions [S1, S2, unofficial].

**When not to use:** When real-time dictionary training is required or for datasets with low redundancy where prescribed bases perform similarly [S1, S2, unofficial].

## Background
The K-SVD algorithm was introduced by Elad and Aharon [MAB06] as a generalization of K-Means for dictionary learning [S2, unofficial]. It was applied to volume rendering by [S2, unofficial] to treat rendered images as collections of pixel blocks for compression and artifact reduction. The method builds on [Compression](4-compression.md) principles by using dictionary-based transformation and sparse representation [S1, S2, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
