# Principal Component Analysis (PCA)

## Overview
Principal Component Analysis (PCA) is a dimensionality reduction technique that finds a linear subspace maximizing the variance of projected data points [S1]. It serves as the foundational statistical method for the Karhunen-Loeve Transform (KLT) [Parent], providing the eigenvectors required to decorrelate volume data for compression [S1]. Unlike its sibling [KLT Decompression with VQ](5374-klt-decompression-with-vq.md), which focuses on quantization strategies for reconstruction, PCA defines the optimal orthogonal basis for the data representation itself.

## How it works
**Inputs:** A dataset of $N$ sample points $\{X_i\}_{i=1}^N$, where each $X_i \in \mathbb{R}^D$. The target dimensionality $d$ (where $d < D$).
**Outputs:** A set of $d$ orthonormal eigenvectors $\{\hat{u}_j\}_{j=1}^d$ forming the projection basis, and the low-dimensional representations $Y_i \in \mathbb{R}^d$ for each input point.

**Data Structures:**
1.  **Data Matrix $X$**: A $D \times N$ matrix where columns are the sample vectors $X_i$.
2.  **Covariance Matrix $C$**: A $D \times D$ symmetric matrix.
3.  **Eigenvectors $U$**: A $D \times d$ matrix containing the selected top eigenvectors.
4.  **Projected Data $Y$**: A $d \times N$ matrix where columns are $Y_i$.

**Steps:**
1.  **Center the Data**: Compute the mean vector $\mu = \frac{1}{N} \sum_{i=1}^N X_i$. Subtract the mean from each sample: $X'_i = X_i - \mu$. [G: Standard preprocessing required for PCA to maximize variance around the origin; not explicitly detailed in [S1] but implied by the covariance definition].
2.  **Compute Covariance Matrix**: Calculate the covariance matrix $C$ of the centered sample points:
    $$ C = \frac{1}{N} \sum_{i=1}^N X'_i (X'_i)^T $$
    [S1]
3.  **Solve Eigenvalue Equation**: Compute the eigenvectors and eigenvalues of $C$. The sources specify using Singular Value Decomposition (SVD) to solve the eigenvalue equation:
    $$ C \hat{u} = \lambda \hat{u} $$
    [S1]
4.  **Select Basis Vectors**: Sort the eigenvectors by their corresponding eigenvalues in descending order. Select the top $d$ eigenvectors $\{\hat{u}_j\}_{j=1}^d$ that yield the largest eigenvalues. These vectors define the subspace that maximizes the variance of the projected data. [S1]
5.  **Project Data**: Compute the $j$-th component of the low-dimensional representation $Y_i$ for each sample $X_i$ by taking the inner product with the selected eigenvectors:
    $$ Y_{ij} = X_i \cdot \hat{u}_j $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def pca(X, d):
    """
    Perform Principal Component Analysis.
    
    Args:
        X: numpy array of shape (N, D), N samples, D dimensions.
        d: int, target number of principal components.
        
    Returns:
        Y: numpy array of shape (N, d), projected data.
        U: numpy array of shape (D, d), selected eigenvectors.
    """
    # 1. Center the data
    mean = np.mean(X, axis=0)
    X_centered = X - mean
    
    # 2. Compute covariance matrix C
    # C = (1/N) * X_centered.T @ X_centered
    # Note: Using (N-1) is standard for unbiased estimation, 
    # but [S1] implies the standard covariance definition. 
    # The scaling factor does not affect eigenvectors.
    N, D = X_centered.shape
    C = np.dot(X_centered.T, X_centered) / N
    
    # 3. Solve eigenvalue equation using SVD or eigendecomposition
    # [S1] specifies SVD. For a symmetric matrix C, 
    # eigendecomposition is equivalent and often preferred numerically.
    # We compute eigenvalues and eigenvectors.
    eigenvalues, eigenvectors = np.linalg.eigh(C)
    
    # 4. Select top d eigenvectors
    # eigh returns eigenvalues in ascending order.
    # We need the largest eigenvalues, so we take the last d.
    indices = np.argsort(eigenvalues)[::-1]
    top_indices = indices[:d]
    
    U = eigenvectors[:, top_indices]
    
    # 5. Project data
    # Y_ij = X_i . u_j
    # Matrix multiplication: Y = X @ U
    Y = np.dot(X, U)
    
    return Y, U
```

**Edge Cases and Pitfalls:**
*   **Nonlinear Structures**: PCA assumes sample points lie on or near a linear subspace. If the data exhibits nonlinear structures, PCA fails to preserve the shape faithfully [S1].
*   **Computational Cost**: The computation is expensive due to the SVD/eigendecomposition step, particularly for high-dimensional data where $D$ is large [S1].

## Subtopics

### [Clustered Principal Component Analysis (CPCA)](10896-clustered-principal-component-analysis-cpca.md)
A method to compress visibility data by partitioning the domain into angular tiles and applying PCA to clusters of tiles covering the same subset of the domain, allowing early culling of visible/occluded tiles.

## Operating characteristics
*   **Complexity**: The method is computationally expensive due to the SVD step [S1]. The naive construction of the covariance matrix and SVD scales poorly with dimensionality $D$.
*   **Optimality**: The solution provided by PCA is a global minimum for the variance maximization objective [S1].
*   **Strengths**: Simplicity of the algorithm and widespread applicability [S1].
*   **Limitations**: Strict assumption of linearity; failure on nonlinear data manifolds [S1].

## Background
PCA is a widely used dimensionality reduction algorithm in many fields [S1]. It is closely related to the Karhunen-Loeve Transform (KLT), which uses PCA to decorrelate data for compression [Parent]. The method was discussed in the context of multi-dimensional transfer function reduction in Information Visualization [S1].

## Sources
[S1] Dimensionality reduction on multi-dimensional transfer functions for ..., https://cseweb.ucsd.edu/~hskim/files/IVS_ivs20106.pdf, fetched 2026-10-27
