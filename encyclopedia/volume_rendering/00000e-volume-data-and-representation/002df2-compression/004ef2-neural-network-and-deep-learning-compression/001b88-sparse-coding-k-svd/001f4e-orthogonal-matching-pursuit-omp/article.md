# Orthogonal Matching Pursuit (OMP)
Also called: Matching Pursuit (MP)

## Overview
Orthogonal Matching Pursuit (OMP) is a greedy algorithm used in sparse coding to find an approximate sparse solution $\mathbf{x}$ for the problem $\min \|\mathbf{x}\|_0$ subject to $\|\mathbf{y} - \mathbf{D}\mathbf{x}\|_2 \leq \epsilon$ [S1, unofficial]. Because extracting the exact sparsest representation is an NP-hard problem [DMA97], OMP iteratively selects dictionary atoms to find an approximate solution instead [S1, unofficial]. It is distinguished from its sibling Matching Pursuit (MP) by its use of orthogonal projection, which ensures faster convergence and better orthogonality properties [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Signal vector $\mathbf{y} \in \mathbb{R}^M$, overcomplete dictionary $\mathbf{D} \in \mathbb{R}^{M \times N}$ (where $N > M$), and target sparsity $L$ [S1, unofficial].
- Output: Sparse coefficient vector $\mathbf{x} \in \mathbb{R}^N$ with at most $L$ non-zero entries [S1, unofficial].

**Data Structures:**
- $\mathbf{D}$: Matrix where columns $\mathbf{d}_i$ are normalized atoms.
- $\mathbf{r}$: Residual vector, initialized to $\mathbf{y}$ [G].
- $\Lambda$: Set of selected atom indices, initialized to empty [G].

**Steps:**
1. **Correlation:** Compute the inner products between the signal $\mathbf{y}$ and all dictionary columns $\mathbf{d}_i$ [S1, unofficial].
   $$ \mathbf{p} = \mathbf{D}^T \mathbf{y} $$
2. **Selection:** Identify the index $k$ of the atom with the maximum absolute inner product [G].
   $$ k = \arg \max_i |p_i| $$
3. **Update Index Set:** Add $k$ to the set of selected indices $\Lambda$ [G].
4. **Least Squares:** Deploy a least squares solver to update the coefficients $\mathbf{x}_\Lambda$ for the selected atoms, minimizing the residual error [S1, unofficial].
   $$ \mathbf{x}_\Lambda = (\mathbf{D}_\Lambda^T \mathbf{D}_\Lambda)^{-1} \mathbf{D}_\Lambda^T \mathbf{y} $$
   where $\mathbf{D}_\Lambda$ denotes the submatrix of $\mathbf{D}$ containing the columns indexed by $\Lambda$ [G].
5. **Residual Update:** Update the residual $\mathbf{r}$ based on the new coefficients [G].
   $$ \mathbf{r} = \mathbf{y} - \mathbf{D}_\Lambda \mathbf{x}_\Lambda $$
6. **Termination:** Repeat steps 1-5 until $|\Lambda| = L$ or the residual norm $\|\mathbf{r}\|_2$ is sufficiently small [G].

**Parameters:**
- $L$: The target sparsity, representing the maximum number of nonzero entries allowed in the coefficient vector $\mathbf{x}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def orthogonal_matching_pursuit(y, D, L):
    """
    y: signal vector of length M
    D: dictionary matrix of shape (M, N)
    L: target sparsity (maximum number of nonzero entries)
    """
    M, N = len(y), len(D[0])
    r = list(y)
    indices = []
    x = [0.0] * N

    for _ in range(L):
        # 1. Compute inner products between signal y and dictionary columns
        correlations = [sum(D[j][i] * y[j] for j in range(M)) for i in range(N)]

        # 2. Select atom with maximum absolute correlation
        k = max(range(N), key=lambda i: abs(correlations[i]))
        indices.append(k)

        # 3. Solve least squares for selected atoms
        # x_sub = (D_sub^T D_sub)^-1 D_sub^T y
        D_sub = [[D[j][i] for i in indices] for j in range(M)]
        DtD = [[sum(D_sub[j][a] * D_sub[j][b] for j in range(M)) for b in range(len(indices))] for a in range(len(indices))]
        DtY = [sum(D_sub[j][a] * y[j] for j in range(M)) for a in range(len(indices))]
        x_sub = solve_linear_system(DtD, DtY)

        # 4. Update full coefficient vector
        for idx_pos, idx in enumerate(indices):
            x[idx] = x_sub[idx_pos]

        # 5. Update residual
        r = [y[j] - sum(D_sub[j][idx_pos] * x_sub[idx_pos] for idx_pos in range(len(indices))) for j in range(M)]

    return x

def solve_linear_system(A, b):
    """Solves Ax = b using Gaussian elimination with partial pivoting."""
    n = len(b)
    aug = [A[i][:] + [b[i]] for i in range(n)]
    for i in range(n):
        max_row = i + max(range(i + 1, n), key=lambda r: abs(aug[r][i]), default=i)
        aug[i], aug[max_row] = aug[max_row], aug[i]
        pivot = aug[i][i]
        if abs(pivot) < 1e-12:
            continue
        for j in range(i + 1, n):
            factor = aug[j][i] / pivot
            for k in range(i, n + 1):
                aug[j][k] -= factor * aug[i][k]
    x = [0.0] * n
    for i in range(n - 1, -1, -1):
        x[i] = aug[i][n]
        for j in range(i + 1, n):
            x[i] -= aug[i][j] * x[j]
        if abs(aug[i][i]) > 1e-12:
            x[i] /= aug[i][i]
    return x
```

## Subtopics

### [Alternating MaxVol Column Sampling](14048-alternating-maxvol-column-sampling.md)
Alternating MaxVol Column Sampling is an iterative algorithm that selects a subset of columns from a wide matrix $A \in \mathbb{R}^{M \times N}$ (where $M \ll N$) to approximate its column space. It addresses the limitation of random column sampling, which is efficient but does not guarantee a basis that represents the matrix well, by iteratively maximizing the volume of the selected submatrix. Unlike [Orthogonal Matching Pursuit (OMP)](1999-orthogonal-matching-pursuit-omp.md), which selects atoms sequentially based on residual correlation, this method selects a block of columns simultaneously by alternating between row and column selection to find a submatrix with a large determinant. It is primarily used to enable efficient SVD computation on large matrices by reducing them to a smaller submatrix before decomposition.

## Operating characteristics
OMP is preferred over MP because it offers improvements on orthogonality and faster convergence [S1, unofficial]. The algorithm converges under the condition that the sparse coding works perfectly, providing the best approximation to signal $\mathbf{y}_i$ containing no more than $L$ nonzero entries [S1, unofficial]. Each sparse coding step reduces the total representation mean squared error (MSE) $\|\mathbf{Y} - \mathbf{D}\mathbf{X}\|_F^2$ [S1, unofficial]. The sparsity of the coefficient matrix is restricted by the properties of the dictionary $\mathbf{D}$ [S1, unofficial].

## Background
OMP was introduced in [CBL89, DMZ94, PRK93, Tro04] and is a greedy algorithm for sparse coding [S1, unofficial]. It is closely related to the K-SVD dictionary learning algorithm due to its simplicity and effectiveness [S1, unofficial]. The exact sparsest representation is NP-hard [DMA97], making OMP a practical alternative [S1, unofficial].

## Sources
[S1] Compressive Volume Rendering, https://doi.org/10.11575/prism/25391, fetched 2026-10-27 (unofficial)
