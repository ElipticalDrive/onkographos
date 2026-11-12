# Principal Component Analysis (PCA)

Also called: PCA Compression, Principal Component Analysis for Residuals

## Overview
Principal Component Analysis (PCA) is a linear dimensionality reduction technique that finds a subspace maximizing the variance of projected data, assuming sample points lie on or near a linear subspace [S2]. It serves as the foundational statistical method for the Karhunen-Loeve Transform (KLT) [Parent], providing the eigenvectors used to decorrelate volume data, and is distinct from its sibling [KLT Decompression with VQ](5374-klt-decompression-with-vq.md) by focusing on the basis computation rather than the quantization scheme.

## How it works
**Inputs:** A set of $N$ input vectors $\{X_i\}_{i=1}^N$, where each $X_i \in \mathbb{R}^D$. In volume rendering contexts, these vectors are often reshaped from feature grids $f_1$ and residual grids $r_t$ into matrices $f_1(m, n)$ and $r_t(m, n)$, where $m$ is the number of non-empty feature voxels and $n$ is the number of feature channels [S1].

**Outputs:** A low-dimensional representation $\{Y_i\}_{i=1}^N$ where $Y_i \in \mathbb{R}^d$ ($d < D$), and the principal directions (eigenvectors) $\hat{u}_j$ for $j=1, \dots, d$ [S2].

**Data Structures:**
1.  **Input Matrix $X$:** Size $N \times D$. Each row is a sample vector $X_i$.
2.  **Covariance Matrix $C$:** Size $D \times D$.
3.  **Eigenvectors $U$:** Size $D \times d$. Columns are the top $d$ eigenvectors $\hat{u}_j$.
4.  **Projected Data $Y$:** Size $N \times d$.

**Steps:**
1.  **Compute Covariance Matrix:** Calculate the covariance matrix $C$ of the sample points $X_i$.
    $$ C = \frac{1}{N} \sum_{i=1}^N (X_i - \mu)(X_i - \mu)^T $$
    where $\mu$ is the mean vector [S2].
2.  **Solve Eigenvalue Equation:** Solve for the eigenvectors and eigenvalues of $C$:
    $$ C\hat{u} = \lambda\hat{u} $$
    This is typically computed using Singular Value Decomposition (SVD) [S2].
3.  **Select Principal Directions:** Select the top $d$ eigenvectors $\hat{u}_j$ corresponding to the largest eigenvalues $\lambda_j$. These directions maximize the variance of the projected data [S2].
4.  **Project Data:** Compute the $j$-th component of the low-dimensional representation $Y_i$ by taking the inner product of the input $X_i$ and the eigenvector $\hat{u}_j$:
    $$ Y_{ij} = X_i \cdot \hat{u}_j $$
    [S2]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def pca_compress(X, d):
    """
    Compress data X using PCA.
    X: numpy array of shape (N, D)
    d: target dimensionality
    Returns: Y (compressed data), U (principal directions)
    """
    # Step 1: Center the data
    mean = np.mean(X, axis=0)
    X_centered = X - mean
    
    # Step 2: Compute Covariance Matrix
    # C = (1/N) * X_centered.T @ X_centered
    # Note: For numerical stability, SVD is often preferred over explicit covariance
    # However, the source [S2] explicitly mentions solving the eigenvalue equation of C.
    # We follow the source's logic: compute C, then eigendecompose.
    N, D = X_centered.shape
    C = (X_centered.T @ X_centered) / N
    
    # Step 3: Solve Eigenvalue Equation C*u = lambda*u
    # Using numpy's eig which returns eigenvalues and eigenvectors
    eigenvalues, eigenvectors = np.linalg.eigh(C)
    
    # Step 4: Select top d eigenvectors
    # eigenvalues are sorted in ascending order by np.linalg.eigh
    # We need the largest eigenvalues, so we take the last d
    top_indices = np.argsort(eigenvalues)[::-1][:d]
    U = eigenvectors[:, top_indices] # Shape (D, d)
    
    # Step 5: Project data
    # Y_ij = X_i . u_j
    Y = X_centered @ U # Shape (N, d)
    
    return Y, U

def pca_compress_residual_matrix(rt, d):
    """
    Specific implementation for residual matrix rt(m, n) as described in [S1].
    rt: numpy array of shape (m, n)
    d: number of principal directions (columns of V)
    Returns: rt_prime (compressed residual), V (principal directions)
    """
    m, n = rt.shape
    
    # Perform SVD on rt
    # rt(m,n) = U * diag(S) * V^T
    # Note: np.linalg.svd returns U, s, Vt
    U, s, Vt = np.linalg.svd(rt, full_matrices=False)
    
    # V is the transpose of Vt
    V = Vt.T # Shape (n, min(m, n))
    
    # Select top d principal directions (columns of V)
    # These correspond to the largest singular values
    V_principal = V[:, :d] # Shape (n, d)
    
    # Project rt onto principal directions
    # rt' = rt * V
    rt_prime = rt @ V_principal # Shape (m, d)
    
    return rt_prime, V_principal
```

**Parameters:**
*   $d$: The target dimensionality. Typical values depend on the compression ratio required. Not given in the sources.
*   $m, n$: Dimensions of the feature/residual matrix. $m$ is the number of non-empty voxels, $n$ is the number of channels [S1].

**Edge Cases and Pitfalls:**
*   **Nonlinearity:** PCA assumes sample points are on or near a linear subspace. If the data has nonlinear structures, PCA fails to preserve the shape faithfully [S2].
*   **Computational Cost:** Computing the covariance matrix and solving the eigenvalue equation is computationally expensive, particularly for high-dimensional data [S2].

## Subtopics

### [Clustered Principal Component Analysis (CPCA)](10896-clustered-principal-component-analysis-cpca.md)
Applies PCA to clusters of tiles in the angular domain to exploit spatial and angular locality, suitable when global linearity does not hold.

### [PCA-based TSDF Compression](12932-pca-based-tsdf-compression.md)
Applies PCA to Truncated Signed Distance Field voxel blocks using a single global basis to preserve structural information and filter noise.

## Operating characteristics
*   **Time Complexity:** Computationally expensive due to the Singular Value Decomposition (SVD) or eigenvalue decomposition required to compute the covariance matrix and its eigenvectors [S2]. A naive implementation of related nearest-neighbor steps in dimensionality reduction can be $O(N^2 D)$, but PCA itself is dominated by the $O(D^3)$ cost of eigendecomposition or $O(N D^2)$ for SVD on the data matrix [S2].
*   **Memory Complexity:** Requires storing the covariance matrix ($D \times D$) or the full SVD components.
*   **Strengths:** The solution is a global minimum for the variance maximization problem [S2]. It is widely used due to its simplicity and effectiveness for linearly correlated data [S2].
*   **Limitations:** Fails on nonlinear data structures [S2]. Computationally intensive for large $D$ [S2].
*   **When to use:** When data exhibits strong linear correlations and computational resources allow for SVD/eigendecomposition.
*   **When not to use:** When data has complex nonlinear manifolds or when real-time performance is critical and $D$ is very large.

## Background
PCA was introduced as a method for dimensionality reduction in information visualization and other fields. It is closely related to the Karhunen-Loeve Transform (KLT) [Parent]. The method is widely used in many fields due to its simplicity [S2].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] PDF Dimensionality reduction on multi-dimensional transfer functions for ..., https://cseweb.ucsd.edu/~hskim/files/IVS_ivs20106.pdf, fetched 2026-10-27
