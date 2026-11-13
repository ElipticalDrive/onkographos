# Sparse Representation Modeling

Also called: Dictionaries for Sparse Representation Modeling, Sparse Representation / Sparse Coding, Sparse Representation via Dictionary Learning

## Overview
Sparse Representation Modeling represents a signal as a sparse linear combination of atoms from an overcomplete dictionary matrix [S1, unofficial]. It solves the problem of compactly encoding volumetric data by allowing each input signal to be represented by a linear combination of dictionary elements rather than a single codeword, generalizing Vector Quantization (VQ) [S2][S3]. Unlike VQ, which is limited by a hard limit on achievable quality and compression rate [S3], this method allows arbitrary coefficient values and multiple nonzero entries, offering increased flexibility and adaptation to specific signal data [S2][S3].

## How it works
**Inputs and Outputs:**
- Input: A signal vector $\mathbf{y} \in \mathbb{R}^n$ (typically a flattened voxel block) [S1, unofficial].
- Input: A dictionary matrix $\mathbf{D} \in \mathbb{R}^{n \times K}$ where $K > n$ (overcomplete) [S1, unofficial].
- Output: A sparse coefficient vector $\mathbf{x} \in \mathbb{R}^K$ [S1, unofficial].

**Data Structures:**
- Dictionary $\mathbf{D}$: A matrix of size $n \times K$ [S1, unofficial]. Each column $\mathbf{d}_j$ is a prototype signal-atom [S1, unofficial]. The dictionary can be a mathematical model (e.g., wavelets) or learned from data [S2][S3].
- Coefficient Vector $\mathbf{x}$: A vector of size $K$ [S1, unofficial].

**Method:**
The goal is to find $\mathbf{x}$ such that $\mathbf{y} \approx \mathbf{D}\mathbf{x}$ with $\mathbf{x}$ having the fewest nonzero coefficients [S1, unofficial]. This is formulated as an optimization problem:

$$
\min_{\mathbf{x}} \|\mathbf{x}\|_0 \quad \text{subject to} \quad \mathbf{y} = \mathbf{D}\mathbf{x}
$$

Or, allowing for an error threshold $\epsilon$:

$$
\min_{\mathbf{x}} \|\mathbf{x}\|_0 \quad \text{subject to} \quad \|\mathbf{y} - \mathbf{D}\mathbf{x}\|_2 \le \epsilon
$$

**Parameters:**
- $n$: Dimension of the signal (e.g., $6^3 = 216$ for a $6 \times 6 \times 6$ voxel block) [S2].
- $K$: Number of atoms in the dictionary (e.g., 1024) [S2].
- Sparsity $S$: The maximum number of nonzero entries in $\mathbf{x}$ [S2].

**Pseudocode:**
The following pseudocode outlines the representation of a single signal block $\mathbf{y}$ given a dictionary $\mathbf{D}$ and sparsity constraint $S$ [G]. Note that solving the $\ell_0$ minimization is NP-hard; practical implementations use greedy algorithms like Orthogonal Matching Pursuit (OMP) [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_representation(y, D, S):
    """
    Computes the sparse representation x of signal y using dictionary D
    with sparsity constraint S.
    Args:
        y: list of floats, length n
        D: list of column vectors, shape (n, K)
        S: int, maximum number of nonzero coefficients
    Returns:
        x: list of floats, length K
    """
    n = len(y)
    K = len(D[0])
    x = [0.0] * K
    selected_indices = []
    residual = list(y)

    for _ in range(S):
        # Compute correlations between residual and all atoms
        correlations = [abs(sum(D[j][i] * residual[i] for i in range(n))) for j in range(K)]
        # Find the atom index with the highest correlation
        idx = max(range(K), key=lambda j: correlations[j])
        if idx in selected_indices:
            break
        selected_indices.append(idx)

        # Solve least squares for selected atoms: min ||y - D_sel * x_sel||^2
        # Using normal equations: (D_sel^T D_sel) x_sel = D_sel^T y
        m = len(selected_indices)
        G = [[sum(D[selected_indices[p]][i] * D[selected_indices[q]][i] for i in range(n)) for q in range(m)] for p in range(m)]
        b = [sum(D[selected_indices[p]][i] * y[i] for i in range(n)) for p in range(m)]
        x_sel = solve_linear_system(G, b)

        # Update full coefficient vector
        for i, ind in enumerate(selected_indices):
            x[ind] = x_sel[i]

        # Update residual
        residual = [y[i] - sum(D[ind][i] * x[ind] for ind in selected_indices) for i in range(n)]

        # Early exit if residual is negligible
        if sum(r * r for r in residual) < 1e-12:
            break

    return x

def solve_linear_system(G, b):
    """Solves Gx = b using Gaussian elimination with partial pivoting."""
    n = len(b)
    A = [G[i] + [b[i]] for i in range(n)]
    for i in range(n):
        max_row = i + max(range(i + 1, n), key=lambda r: abs(A[r][i]), default=i)
        A[i], A[max_row] = A[max_row], A[i]
        pivot = A[i][i]
        if abs(pivot) < 1e-12:
            continue
        for j in range(i + 1, n):
            factor = A[j][i] / pivot
            for k in range(i, n + 1):
                A[j][k] -= factor * A[i][k]
    x = [0.0] * n
    for i in range(n - 1, -1, -1):
        if abs(A[i][i]) < 1e-12:
            x[i] = 0.0
        else:
            x[i] = (A[i][n] - sum(A[i][j] * x[j] for j in range(i + 1, n))) / A[i][i]
    return x
```

**Edge Cases and Pitfalls:**
- **Ill-conditioned Dictionary:** If atoms in $\mathbf{D}$ are highly correlated, the least squares step may become unstable; normalizing atoms to unit length is recommended [G].
- **Sparsity Constraint:** If $S$ is too large, the representation may not be sparse, defeating the purpose of compression; if $S$ is too small, reconstruction error increases [G].

## Subtopics

### [Sparse Coding with K-SVD and Coresets](5748-sparse-coding-with-k-svd-and-coresets.md)
This method addresses the prohibitive computational cost of training data-specific dictionaries on massive volume datasets by combining the K-SVD algorithm with coreset-based data reduction. Unlike uniform subsampling, which fails to capture critical information at material boundaries, this approach uses importance sampling to construct a small, weighted subset of training blocks that approximates the original data distribution. It builds on [Sparse Representation Modeling](5392-sparse-representation-modeling.md) by providing a scalable training pipeline for dictionary-based sparse representations.

## Operating characteristics
- **Time Complexity:** The encoding step (finding $\mathbf{x}$) is NP-hard in general ($\ell_0$ minimization) [G]. Greedy approximations like OMP take $O(S \cdot n \cdot K)$ per signal [G]. Dictionary learning (e.g., K-SVD) is significantly more expensive, requiring iterative updates over the entire training set [G].
- **Memory Complexity:** Storing the dictionary requires $O(n \cdot K)$ space [G]. For each block, storing the indices and coefficients requires $O(S)$ space [G]. Compared to VQ, sparse coding stores multiple coefficients per block, increasing storage per block but enabling higher reconstruction quality [S2][S3].
- **Performance:** Demonstrated state-of-the-art performance on very large static and dynamic volumes [S2][S3].
- **Strengths:** Allows linear combinations of atoms for flexible representation; learned dictionaries adapt to specific signal data [S2][S3].
- **Limitations:** Encoding requires solving an optimization problem per block, and dictionary learning is computationally intensive [G].

## Background
The concept of dictionary-based sparse representation is a generalization of Vector Quantization (VQ) [S2][S3]. While historically signal decomposition used orthogonal or bi-orthogonal bases, there is growing interest in overcomplete dictionaries (containing linearly dependent vectors) for sparse coding [S2][S3]. Recent architectures based on sparse representation of voxel blocks have demonstrated state-of-the-art performance on very large static and dynamic volumes [S2][S3]. The method is closely related to Fractal Compression, which also works on block entities but relies on self-similarity and affine transformations [S2][S3].

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
[S2] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
