# Normal Equations Solver

Also called: A^T A Solver, Least Squares via Normal Equations

## Overview
The Normal Equations Solver addresses the overdetermined linear system $Ax = b$ by explicitly forming the symmetric positive semi-definite system $A^T A x = A^T b$ and solving it using a standard linear solver [S1, unofficial]. It is distinguished from iterative methods (like CGLS) and direct factorizations of the original matrix (like QR) by its reliance on the explicit construction of the Hessian matrix $A^T A$ [S1, unofficial]. While generally the fastest alternative for small-to-medium systems, it is prone to numerical instability if the system matrix $A$ is ill-conditioned [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: System matrix $A \in \mathbb{R}^{m \times n}$ (where $m \gg n$) and vector $b \in \mathbb{R}^m$.
- Output: Solution vector $x \in \mathbb{R}^n$.

**Data Structures:**
- Matrix $A$: Stored in Compressed Sparse Row (CSR) format to allow efficient sequential querying of non-zero entries for the $i$-th voxel (row) [S1, unofficial].
- Matrix $A^T A$: A band matrix with non-zero entries only on the main diagonal and two diagonals four rows/columns away from the main diagonal. The distance of four arises from the separate handling of RGB*$\alpha$ channels [S1, unofficial]. Due to symmetry, only the upper triangular part is stored [S1, unofficial].
- Vector $A^T b$: Standard dense or sparse vector representation.

**Steps:**
1. **Assemble $A^T A$ and $A^T b$**: Compute the sparse matrix-matrix product $A^T A$ and the sparse matrix-vector product $A^T b$. This can be done using linear algebra libraries (e.g., Eigen on CPU, cuSPARSE on GPU) or custom matrix-free compute shaders [S1, unofficial].
   - In a matrix-free GPU implementation (e.g., Vulkan compute), iterate over all voxels. For each voxel $i$, compute the column indices $j(D_i)$ and $j(D_i)+1$ and the interpolation weight $w(i)$ [S1, unofficial].
   - Atomically add the following values to $A^T A$:
     - $(1 - w(i))^2$ to entry $(j(D_i), j(D_i))$
     - $w(i)^2$ to entry $(j(D_i)+1, j(D_i)+1)$
     - $(1 - w(i))w(i)$ to entry $(j(D_i)+1, j(D_i))$ [S1, unofficial]
   - Similarly compute $A^T b$ [S1, unofficial].
2. **Solve the Symmetric System**: Solve the ordinary least squares problem $A^T A x = A^T b$ using a dense or sparse linear least squares solver (e.g., QR, LLT, LDLT) [S1, unofficial].
3. **Constraint Handling**: The standard normal equations solver does not enforce box constraints $0 \le x_i \le 1$. If the unconstrained solution violates these constraints, apply truncation: $x'_i = \max\{\min\{x_i, 1\}, 0\}$ [S1, unofficial]. If explicit constraint enforcement is required, a constrained quadratic programming solver (e.g., ADMM via OSQP) should be used instead [S1, unofficial].

**Formulas:**
- Normal Equations: $A^T A x = A^T b$ [S1, unofficial]
- Condition Number Relationship: $\text{cond}(A^T A) = \text{cond}(A)^2$. The source text states "cond(A T A) = cond(A)" but notes the squaring effect on condition number magnitude and instability, implying the squared relationship is the operative numerical concern [S1, unofficial].
- Truncation: $x'_i = \max\{\min\{x_i, 1\}, 0\}$ [S1, unofficial]

**Parameters:**
- Precision: 32-bit floating point is typical. For a dataset with $1.76 \cdot 10^6$ voxels, storing $A$ in CSR format requires almost 7 GiB of memory [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Ill-conditioning**: The condition number of $A^T A$ is the square of the condition number of $A$, leading to potential numerical instability [S1, unofficial].
- **Rank Deficiency**: $A^T A$ is often nearly rank deficient because certain transfer function values are hardly referenced by voxel values, or rank deficient if no values fall within a certain range. Solvers like QR may require pivoting for stability [S1, unofficial].
- **GPU Atomic Contention**: While atomic additions on GPU for assembling $A^T A$ might theoretically cause write conflicts and slowdowns, in practice this method proved to be one of the fastest solutions [S1, unofficial].

## Subtopics

### [Normal Equations-based Solver](5483-normal-equations-based-solver.md)
Addresses the box-constrained least squares problem ($0 \le x_i \le 1$) by formulating it as a quadratic programming (QP) problem. It differs from direct unconstrained solvers by explicitly handling constraints (often via ADMM) and differs from iterative solvers by relying on the explicit construction of the Hessian $Q = A^T A$.

## Operating characteristics
- **Time Complexity**: Generally the fastest alternative for small-to-medium systems where $A^T A$ is small enough to fit in memory and be solved directly [S1, unofficial].
- **Memory Complexity**: Storing $A$ is expensive (e.g., ~7 GiB for $1.76 \cdot 10^6$ voxels with 32-bit floats), but $A^T A$ is usually quite small and can be stored in dense or sparse format [S1, unofficial].
- **Strengths**: Fast assembly and solution for small systems; leverages highly optimized dense linear algebra libraries for the final solve step [S1, unofficial].
- **Limitations**: Prone to stability issues if $A$ is ill-conditioned due to the squaring of the condition number; does not natively support box constraints without post-processing or external QP solvers [S1, unofficial].
- **When to Use**: When $A^T A$ is small enough to be stored and solved directly, and numerical stability is not a primary concern or the matrix is well-conditioned.
- **When Not to Use**: When $A$ is ill-conditioned (use QR or iterative methods); when explicit box constraints are required (use Normal Equations-based Solver with QP).

## Background
The method originates from the formulation of ordinary least squares problems in volume rendering transfer function optimization. It is discussed in the context of comparing solver strategies for optimizing RGB*$\alpha$ values, where the system matrix $A$ is tall and skinny ($m \gg n$) [S1, unofficial]. The approach is noted as the most straightforward solution, with implementations supporting CPU (Eigen), GPU (cuSPARSE), and matrix-free Vulkan compute [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
