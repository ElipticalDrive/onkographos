# Principal Component Analysis (PCA)
Also called: PCA Compression, Principal Component Analysis for Residuals

## Overview
Principal Component Analysis (PCA) is a linear dimensionality reduction technique that finds a subspace maximizing the variance of projected data, assuming sample points lie on or near a linear subspace [S2]. It is used to compress high-dimensional data (such as residual grids in volume rendering) by projecting it onto a lower-dimensional basis derived from the data's covariance structure [S1, S2]. Unlike its sibling [KLT Decompression with VQ](5374-klt-decompression-with-vq.md), which uses vector quantization for discrete representation, standard PCA provides a continuous linear projection that yields a global minimum for the reconstruction error under the linear assumption [S2].

## How it works
**Inputs:**
- A set of $N$ input vectors $\{X_i\}_{i=1}^N$, where each $X_i \in \mathbb{R}^D$ [S2]. In the context of volume rendering residual compression, these are typically reshaped from a feature grid $f_1$ and residual grid $r_t$ into an $m \times n$ matrix, where $m$ is the number of non-empty feature voxels and $n$ is the number of feature channels [S1].

**Outputs:**
- A low-dimensional representation $\{Y_i\}_{i=1}^N$, where $Y_i \in \mathbb{R}^d$ and $d < D$ [S2].
- The principal directions (eigenvectors) used for projection [S1, S2].

**Data Structures:**
- **Covariance Matrix ($C$):** A $D \times D$ symmetric matrix [S2].
- **Eigenvectors ($\hat{u}_j$):** The top $d$ eigenvectors of $C$, forming a $D \times d$ matrix [S2].
- **SVD Components:** For residual matrices, the decomposition yields matrices $U$ ($m \times q$), $S$ ($q$-vector), and $V$ ($n \times q$), where $q = \min(m, n)$ [S1].

**Steps:**
1. **Compute Covariance:** Calculate the covariance matrix $C$ of the input sample points $\{X_i\}$ [S2].
2. **Solve Eigenvalue Equation:** Solve $C\hat{u} = \lambda\hat{u}$ to find eigenvectors and eigenvalues [S2]. This is typically done using Singular Value Decomposition (SVD) [S2].
3. **Select Directions:** Select the top $d$ eigenvectors $\hat{u}_j$ (for $j=1, \dots, d$) corresponding to the largest eigenvalues. These directions maximize the variance of the projected data [S2].
4. **Project Data:** Compute the $j$-th component of the low-dimensional representation $Y_i$ by taking the inner product of the input $X_i$ and the eigenvector $\hat{u}_j$:
   $$ Y_{ij} = X_i \cdot \hat{u}_j $$
   [S2]

**Alternative Formulation for Residual Matrices:**
When applied to a residual matrix $r_t$ of size $m \times n$:
1. Perform SVD on $r_t$:
   $$ r_t(m,n) = U \cdot \text{diag}(S) \cdot V^T $$
   where $V$ is an $n \times q$ matrix of principal directions, $S$ is a $q$-vector of singular values, and $U$ is an $m \times q$ matrix [S1].
2. Project the residual matrix onto the principal directions $V$ to obtain the compressed residual $r_t'$:
   $$ r_t' = r_t \cdot V $$
   [S1]

**Parameters:**
- $d$: The target dimensionality (number of principal components). Not given in the sources.

**Edge Cases and Pitfalls:**
- **Nonlinearity:** PCA assumes sample points are on or near a linear subspace. If the data has nonlinear structures, PCA fails to preserve the shape faithfully [S2].
- **Computational Cost:** The computation is expensive due to the SVD step, although the solution is a global minimum [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def pca_compress(X, d):
    """
    Compress data X using PCA.
    
    Args:
        X: numpy array of shape (N, D), N samples, D dimensions.
        d: int, target dimensionality.
        
    Returns:
        Y: numpy array of shape (N, d), compressed data.
        U: numpy array of shape (D, d), principal directions (eigenvectors).
    """
    # Step 1: Compute covariance matrix
    # C = (1/N) * X^T * X (assuming X is centered)
    # Note: Standard PCA implementation usually centers X first.
    # [G] Centering is required for covariance to represent variance around the mean.
    X_centered = X - np.mean(X, axis=0)
    C = np.cov(X_centered, rowvar=False)
    
    # Step 2: Solve eigenvalue equation using SVD or eigendecomposition
    # [S2] SVD results in the eigenvector with the largest eigenvalue.
    # Using numpy.linalg.eigh for symmetric matrix C
    eigenvalues, eigenvectors = np.linalg.eigh(C)
    
    # Step 3: Select top d eigenvectors
    # Sort eigenvalues in descending order
    sorted_indices = np.argsort(eigenvalues)[::-1]
    top_d_indices = sorted_indices[:d]
    
    # Get the corresponding eigenvectors (columns)
    U = eigenvectors[:, top_d_indices]
    
    # Step 4: Project data
    # Y_ij = X_i . u_j
    Y = X_centered @ U
    
    return Y, U

def pca_compress_residual_matrix(rt, d):
    """
    Compress a residual matrix rt using PCA via SVD.
    
    Args:
        rt: numpy array of shape (m, n).
        d: int, number of principal directions to keep (q in source, but usually d <= min(m,n)).
           Note: Source [S1] uses q for the rank of the decomposition. 
           If d < q, we truncate V.
           
    Returns:
        rt_prime: numpy array of shape (m, d), compressed residual.
        V: numpy array of shape (n, d), principal directions.
    """
    # Step 1: SVD of rt
    # rt(m,n) = U * diag(S) * V^T
    U, S, Vt = np.linalg.svd(rt, full_matrices=False)
    
    # V is the transpose of Vt
    V = Vt.T
    
    # Step 2: Select top d columns of V
    # [S1] V is n x q. We select the first d columns corresponding to largest singular values.
    V_selected = V[:, :d]
    
    # Step 3: Project rt onto principal directions
    # rt' = rt * V
    rt_prime = rt @ V_selected
    
    return rt_prime, V_selected
```

## Subtopics

### [Clustered Principal Component Analysis (CPCA)](10896-clustered-principal-component-analysis-cpca.md)
CPCA partitions the angular domain into tiles and applies PCA to clusters of tiles covering the same subset of the domain. It is chosen when global linearity does not hold, exploiting spatial and angular locality to compress basis coefficients where standard PCA fails.

## Operating characteristics
- **Time Complexity:** Computationally expensive due to the SVD step [S2]. A naive implementation of related neighborhood steps (in broader context) can be $O(N^2 D)$, but SVD itself is typically $O(\min(mn^2, m^2n))$ for an $m \times n$ matrix [S2].
- **Memory Complexity:** Requires storing the covariance matrix ($D^2$) or the full SVD components ($O(mn)$) [S2, S1].
- **Performance:** The solution is a global minimum for the variance maximization problem [S2].
- **Strengths:** Simplicity of algorithm; widely used; global minimum solution [S2].
- **Limitations:** Assumes linearity; fails on nonlinear structures [S2].
- **When to use:** When data lies near a linear subspace and computational resources allow for SVD [S2].
- **When not to use:** When data has significant nonlinear structures [S2].

## Background
PCA was introduced as a method for dimensionality reduction by finding a subspace that maximizes variance [S2]. It is widely used in many fields due to its simplicity [S2]. In the context of volume rendering and visualization, it is applied to compress feature grids and residuals, such as in Neural Residual Radiance Fields [S1]. The technique is closely related to the Karhunen-Loeve Transform (KLT), which decorrelates data by estimating the covariance matrix and computing eigenvectors [Parent].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
[S2] PDF Dimensionality reduction on multi-dimensional transfer functions for ..., https://cseweb.ucsd.edu/~hskim/files/IVS_ivs20106.pdf, fetched 2026-10-27
