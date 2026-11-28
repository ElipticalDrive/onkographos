# Bilateral gradient filtering for meshes

Also called: Generalized bilateral filter for meshes

## Overview
This method replaces the standard Laplacian postfilter in inverse mesh recovery with a cross-bilateral filter that preserves sharp edges and silhouettes. It sets itself apart from its sibling [Inverse Mesh Recovery with Spatiotemporal Bilateral Gradient Filtering](13136-inverse-mesh-recovery-with-spatiotemporal-bilateral-gradient.md) by using vertex normals to guide the smoothing, preventing the over-smoothing of thin structures and leading to faster convergence [S1].

## How it works
**Inputs:**
- $h$: The vector of values to be filtered (e.g., vertex position updates or displacements).
- $\hat{n}$: The vector of vertex normals for the mesh.
- $L$: The cotangent Laplacian matrix of the mesh.
- $\lambda$: The regularization parameter for the Laplacian term.
- $\sigma_d$: The data term bandwidth parameter.
- $\sigma_\phi$: The partition of unity bandwidth parameter.
- $N$: The number of sampling directions (typically 32).

**Outputs:**
- $\tilde{h}$: The filtered vector.

**Data Structures:**
- $W(\hat{n}, s_i)$: A diagonal matrix where the $j$-th diagonal element is $W_j = \exp(\hat{n}_j \cdot s_i / \sigma_d)$.
- $\Phi(\hat{n}, s_i)$: A diagonal matrix where the $j$-th diagonal element is $\Phi_j = \exp(\hat{n}_j \cdot s_i / \sigma_\phi)$.
- $M$: A matrix of size $V \times 2N$, where $V$ is the number of vertices. The columns are formed by concatenating the vectors $W(\hat{n}, s_i)h$ and $W(\hat{n}, s_i)\mathbf{1}$ for all $i=1 \dots N$.

**Steps:**
1. Sample $N=32$ directions $s_i$ uniformly on the unit sphere $S^2$ using the Sobol sequence [Sobol’ 1967].
2. For each direction $s_i$, compute the diagonal data matrix $W_i = W(\hat{n}, s_i)$ and the diagonal partition of unity matrix $\Phi_i = \Phi(\hat{n}, s_i)$.
3. Construct the matrix $M$ by concatenating the vectors $W_i h$ and $W_i \mathbf{1}$ as columns:
   $$ M = \left[ \begin{array}{c|c} W_1 h & W_1 \mathbf{1} \\ \vdots & \vdots \\ W_N h & W_N \mathbf{1} \end{array} \right] $$
4. Solve the linear system $(I + \lambda L) X = M$ for $X$ using a single matrix solve (e.g., Cholesky decomposition). Let $X = [X_h | X_\phi]$, where $X_h$ contains the solutions for the $W_i h$ columns and $X_\phi$ contains the solutions for the $W_i \mathbf{1}$ columns.
5. Compute the filtered vector $\tilde{h}$ using component-wise operations:
   $$ \tilde{h} = \frac{\sum_{i=1}^N \Phi_i X_{h,i}}{\sum_{i=1}^N \Phi_i X_{\phi,i}} $$
   where $X_{h,i}$ is the $i$-th column of $X_h$ and $X_{\phi,i}$ is the $i$-th column of $X_\phi$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilateral_gradient_filter_mesh(h, n_hat, L, lambda_reg, sigma_d, sigma_phi, N=32):
    """
    Filter vector h using bilateral gradient filtering for meshes.
    
    Args:
        h: (V,) array, values to filter.
        n_hat: (V, 3) array, vertex normals.
        L: (V, V) sparse matrix, cotangent Laplacian.
        lambda_reg: float, regularization parameter.
        sigma_d: float, data term bandwidth.
        sigma_phi: float, partition of unity bandwidth.
        N: int, number of sampling directions.
        
    Returns:
        h_tilde: (V,) array, filtered values.
    """
    import numpy as np
    from scipy.sparse import eye, diags
    from scipy.sparse.linalg import splu
    
    # 1. Sample directions using Sobol sequence
    # Note: In practice, use a Sobol sequence generator. 
    # Here we assume a function sobol_sample(N) returns (N, 3) unit vectors.
    s = sobol_sample(N)  # Shape: (N, 3)
    
    V = h.shape[0]
    ones = np.ones(V)
    
    # Precompute dot products: (V, N)
    # n_hat is (V, 3), s is (N, 3). We need (V, N) where entry (v, i) is n_hat[v] . s[i]
    dot_products = np.dot(n_hat, s.T)  # Shape: (V, N)
    
    # 2. Compute diagonal weights
    # W_i diagonal elements: exp(n_hat . s_i / sigma_d)
    # Phi_i diagonal elements: exp(n_hat . s_i / sigma_phi)
    W_weights = np.exp(dot_products / sigma_d)  # Shape: (V, N)
    Phi_weights = np.exp(dot_products / sigma_phi)  # Shape: (V, N)
    
    # 3. Construct matrix M
    # M columns: [W_1 h, ..., W_N h, W_1 1, ..., W_N 1]
    # W_i h is element-wise multiplication of W_weights[:, i] and h
    # W_i 1 is element-wise multiplication of W_weights[:, i] and ones
    
    # Compute W_i h for all i
    # W_weights is (V, N), h is (V,) -> broadcast to (V, N)
    W_h = W_weights * h[:, np.newaxis]  # Shape: (V, N)
    
    # Compute W_i 1 for all i
    W_1 = W_weights * ones[:, np.newaxis]  # Shape: (V, N)
    
    # Concatenate horizontally
    M = np.hstack([W_h, W_1])  # Shape: (V, 2N)
    
    # 4. Solve (I + lambda L) X = M
    # I + lambda L
    I = eye(V, format='csr')
    A = I + lambda_reg * L
    
    # Solve for X
    # Using sparse LU or Cholesky if symmetric positive definite
    # Assuming L is symmetric positive semi-definite, A is SPD
    try:
        cholesky = splu(A.tocsc()) # Or use a Cholesky solver if available
        X = cholesky.solve(M)
    except:
        # Fallback to general sparse solver if Cholesky fails
        from scipy.sparse.linalg import spsolve
        X = spsolve(A, M)
        
    # X is (V, 2N)
    # Split X into X_h and X_phi
    X_h = X[:, :N]  # Shape: (V, N)
    X_phi = X[:, N:]  # Shape: (V, N)
    
    # 5. Compute h_tilde
    # Numerator: sum_i Phi_i * (W_i h filtered) = sum_i Phi_weights[:, i] * X_h[:, i]
    # Denominator: sum_i Phi_i * (W_i 1 filtered) = sum_i Phi_weights[:, i] * X_phi[:, i]
    
    # Element-wise multiplication and sum over axis 1 (the N dimension)
    numerator = np.sum(Phi_weights * X_h, axis=1)  # Shape: (V,)
    denominator = np.sum(Phi_weights * X_phi, axis=1)  # Shape: (V,)
    
    # Avoid division by zero
    denominator = np.maximum(denominator, 1e-8)
    
    h_tilde = numerator / denominator
    
    return h_tilde
```

**Parameters:**
- $N=32$: Number of sampling directions [S1].
- $\sigma_d, \sigma_\phi$: Bandwidth parameters for the von Mises-Fisher kernel. Not given in the sources.
- $\lambda$: Regularization parameter for the Laplacian. Not given in the sources.

**Edge cases and pitfalls:**
- The Laplacian matrix $L$ must be positive semi-definite, and $I + \lambda L$ must be positive definite for the Cholesky solver to work.
- Division by zero in the denominator should be guarded against [G].
- The Sobol sequence sampling ensures uniform coverage of the sphere, which is critical for the filter's isotropy [S1].

## Subtopics

### [Gradient-Based Mesh Optimization via Spherical Harmonics](23290-gradient-based-mesh-optimization-via-spherical-harmonics.md)
This method optimizes high-frequency surface details of a mesh by backpropagating image-space gradients into spherical harmonic (SH) coefficients within a TetWeave representation.

## Operating characteristics
- **Time Complexity:** The dominant cost is the single matrix solve $(I + \lambda L)^{-1} M$, which is $O(V)$ for sparse matrices if using iterative solvers or $O(V^{1.5})$ to $O(V^2)$ for direct solvers depending on the fill-in. The overhead over standard Laplacian smoothing is minor (<10%) [S1].
- **Memory Complexity:** $O(V)$ for storing the mesh and $O(V \cdot N)$ for the intermediate matrices $M$ and $X$.
- **Performance:** Implemented on NVIDIA GeForce RTX 3080 GPU [S1].
- **Strengths:** Preserves sharp edges and silhouettes better than Laplacian smoothing, leading to faster convergence on thin structures [S1].
- **Limitations:** More complex to implement than Laplacian smoothing. Requires vertex normals.

## Background
This method is based on the generalized bilateral filter from Solomon et al. [2014] and is used in the context of inverse mesh recovery [S1]. It was introduced to improve the quality of reconstructed meshes by preserving geometric details that are lost with standard smoothing techniques.

## Sources
[S1] Spatiotemporal Bilateral Gradient Filtering for Inverse Rendering, https://doi.org/10.1145/3680528.3687606, fetched 2026-10-29
