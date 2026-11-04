# Normal Equations Solver

Also called: A^T A Solver, Least Squares via Normal Equations

## Overview
The Normal Equations Solver addresses the overdetermined linear system $Ax = b$ arising in radiance-field optimization by forming and solving the symmetric positive semi-definite system $A^T A x = A^T b$ [S1, unofficial]. It is distinguished from sibling methods like the General Least Squares Solver by explicitly squaring the condition number to gain computational speed, making it the fastest alternative for well-conditioned systems but prone to numerical instability when the matrix is ill-conditioned [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Sparse system matrix $A \in \mathbb{R}^{m \times n}$ (where $m \gg n$) and vector $b \in \mathbb{R}^m$.
- Output: Solution vector $x \in \mathbb{R}^n$.

**Data Structures:**
- Matrix $A$ is stored in Compressed Sparse Row (CSR) format to allow sequential querying of non-zero entries for the $i$-th voxel (row) [S1, unofficial].
- Matrix $A^T A$ is a band matrix. Due to the separate handling of RGB*$\alpha$ channels, non-zero entries exist only on the main diagonal and the two diagonals four rows/columns away from the main diagonal [S1, unofficial].
- Only the upper triangular part of $A^T A$ is stored due to symmetry [S1, unofficial].

**Steps:**
1. **Assemble $A^T A$ and $A^T b$:** Compute the sparse matrix-matrix product $A^T A$ and sparse matrix-vector product $A^T b$. This can be done using linear algebra libraries (e.g., Eigen on CPU, cuSPARSE on GPU) or custom matrix-free compute shaders [S1, unofficial].
2. **Matrix-Free Assembly (Vulkan Compute Example):** Iterate over all voxels $i$. For each voxel, determine the column indices $j(D_i)$ and $j(D_i)+1$ and the interpolation weight $w(i)$ [S1, unofficial]. Atomically add the following values to $A^T A$:
   - $(1-w(i))^2$ to entry $(j(D_i), j(D_i))$
   - $w(i)^2$ to entry $(j(D_i)+1, j(D_i)+1)$
   - $(1-w(i))w(i)$ to entry $(j(D_i)+1, j(D_i))$ [S1, unofficial].
3. **Solve the System:** Use a dense or sparse linear least squares solver (e.g., QR, LLT, LDLT) on the assembled $A^T A$ and $A^T b$ [S1, unofficial].
4. **Constraint Handling:** The ordinary least squares problem is convex with Hessian $H(x) = 2A^T A$ [S1, unofficial]. If the solver does not enforce box constraints $0 \le x_i \le 1$, apply truncation: $x'_i = \max\{\min\{x_i, 1\}, 0\}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_normal_equations(A_csr, b, num_voxels, get_voxel_indices_and_weight):
    """
    Solves Ax = b via normal equations A^T A x = A^T b.
    A_csr: CSR format matrix A.
    b: Right-hand side vector.
    num_voxels: Number of voxels (rows in A).
    get_voxel_indices_and_weight: Function returning (j, j_plus_1, w) for voxel i.
    """
    n = A_csr.num_columns
    # Initialize symmetric matrix A^T A and vector A^T b
    # Using a dense array for A^T A is feasible as it is typically small
    ATA = zeros((n, n))
    ATb = zeros(n)

    # Step 1: Assemble A^T A and A^T b
    # In a matrix-free GPU implementation, this loop runs in parallel over voxels
    # with atomic additions for ATA. Here we simulate the accumulation.
    for i in range(num_voxels):
        j, j_plus_1, w = get_voxel_indices_and_weight(i)
        
        # Compute contributions to A^T A
        # Entry (j, j)
        val_jj = (1 - w) ** 2
        ATA[j, j] += val_jj
        
        # Entry (j+1, j+1)
        val_jp1_jp1 = w ** 2
        ATA[j_plus_1, j_plus_1] += val_jp1_jp1
        
        # Entry (j+1, j) - symmetric, so also affects (j, j+1)
        val_jp1_j = (1 - w) * w
        ATA[j_plus_1, j] += val_jp1_j
        ATA[j, j_plus_1] += val_jp1_j  # Maintain symmetry explicitly in dense storage
        
        # Compute contribution to A^T b
        # Assuming A is tall and skinny, and b corresponds to the voxel values
        # The specific row of A for voxel i is needed to compute (A^T b)_k
        # However, the source notes imply A^T b is computed similarly or via library.
        # For completeness, if A is sparse CSR:
        # row_start = A_csr.row_ptr[i]
        # row_end = A_csr.row_ptr[i+1]
        # for k in range(row_start, row_end):
        #     col = A_csr.col_ind[k]
        #     val = A_csr.values[k]
        #     ATb[col] += val * b[i]
        # Note: The source emphasizes the atomic assembly of ATA for the specific
        # interpolation structure. A^T b assembly follows standard sparse mat-vec.
        pass

    # Step 2: Solve the symmetric system ATA x = ATb
    # Using a dense solver like LLT or LDLT
    try:
        # LLT requires positive definite; LDLT handles semi-definite better
        # QR is also an option but pivoting may be needed for stability
        x = solve_linear_system(ATA, ATb, method='LDLT')
    except SingularMatrixError:
        # Fallback or error handling for rank deficiency
        x = solve_linear_system(ATA, ATb, method='QR')

    # Step 3: Apply box constraints if not enforced by solver
    x = clip(x, min=0.0, max=1.0)
    
    return x
```

**Parameters:**
- Memory usage for $A$: For a dataset with $1.76 \cdot 10^6$ voxels using 32-bit floating point precision, the system matrix requires almost 7 GiB of memory [S1, unofficial].
- $A^T A$ is usually small enough to be stored in dense format [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Ill-conditioning:** The condition number of the normal equations system is the square of the original matrix's condition number: $\text{cond}(A^T A) = \text{cond}(A)^2$ [S1, unofficial]. This leads to numerical instability for ill-conditioned matrices [S1, unofficial].
- **Rank Deficiency:** $A^T A$ is often nearly rank deficient because certain transfer function values are hardly referenced by voxel values, or rank deficient if no values fall within a certain range [S1, unofficial]. Solvers like QR may require pivoting for stability [S1, unofficial].

## Subtopics

### Normal Equations-based Solver
A method to solve the box-constrained least squares problem $\forall i \in [1,...,n] : 0 \le x_i \le 1$ by formulating it as a quadratic programming problem [S1, unofficial]. It uses the Alternating Direction Method of Multipliers (ADMM) via the OSQP solver library, exploiting sparsity [S1, unofficial]. The quadratic programming problem is $\min_x \frac{1}{2} x^T Q x + c^T x$, where $Q = A^T A$ and $c = -A^T b$ [S1, unofficial]. While a GPU implementation of OSQP exists, no significant speed-up was observed compared to CPU for the small size of $A^T A$ resulting from normal equations [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Generally the fastest alternative for solving the least squares problem due to the efficiency of forming and solving the symmetric system [S1, unofficial].
- **Memory Complexity:** $A$ requires significant memory (e.g., ~7 GiB for $1.76 \cdot 10^6$ voxels with 32-bit floats), but $A^T A$ is typically small and fits in memory [S1, unofficial].
- **Strengths:** Fast assembly and solution for well-conditioned systems; leverages sparse matrix-matrix products efficiently [S1, unofficial].
- **Limitations:** Prone to stability issues if the matrix is ill-conditioned due to the squaring of the condition number [S1, unofficial]. May require pivoting or alternative solvers (like QR) to handle near-rank deficiency [S1, unofficial].
- **When to use:** When computational speed is prioritized and the system matrix $A$ is well-conditioned.
- **When not to use:** When the matrix is ill-conditioned or nearly rank deficient, as numerical errors may be amplified [S1, unofficial].

## Background
The normal equations approach is a standard technique in linear algebra for solving least squares problems. In the context of volume rendering, it was applied to transfer function optimization, where the goal is to optimize RGB*$\alpha$ values subject to box constraints [S1, unofficial]. The method was evaluated against other solvers like CGLS, noting that while normal equations are faster, they suffer from condition number squaring [S1, unofficial]. The specific implementation details, including matrix-free Vulkan compute shaders for assembly, were described in recent work on transfer function optimization for comparative volume rendering [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
