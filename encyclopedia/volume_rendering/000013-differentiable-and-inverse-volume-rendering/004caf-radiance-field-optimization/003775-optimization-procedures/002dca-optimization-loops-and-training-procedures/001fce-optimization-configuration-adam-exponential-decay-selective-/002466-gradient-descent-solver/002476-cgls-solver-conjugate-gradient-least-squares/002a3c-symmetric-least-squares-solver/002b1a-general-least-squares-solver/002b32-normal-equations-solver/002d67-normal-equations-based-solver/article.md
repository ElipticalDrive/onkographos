# Normal Equations-based Solver

Also called: A^T A Solver, Constrained Least Squares, Least Squares via Normal Equations, Quadratic Programming Solver (Box-Constrained)

## Overview
The Normal Equations-based Solver addresses the box-constrained least squares problem $\forall i \in [1,...,n] : 0 \le x_i \le 1$ by formulating it as a quadratic programming (QP) problem [S1, unofficial]. It differs from direct unconstrained solvers by explicitly handling the box constraints, often via the Alternating Direction Method of Multipliers (ADMM), and differs from iterative least-squares solvers (like CGLS) by relying on the explicit construction of the Hessian matrix $Q = A^T A$ [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Matrix $A$ (size $m \times n$), vector $b$ (size $m$), and constraint bounds $[0, 1]$ for each element of $x$.
- Output: Vector $x$ (size $n$) minimizing the constrained objective.

**Data Structures:**
- The core data structure is the sparse matrix $Q = A^T A$ and vector $c = -A^T b$. The solver exploits the sparsity of $Q$ [S1, unofficial].

**Steps:**
1. Construct the quadratic programming parameters:
   - Compute $Q = A^T A$.
   - Compute $c = -A^T b$.
   - The objective is $\min_x \frac{1}{2} x^T Q x + c^T x$ [S1, unofficial].
2. Solve the constrained QP problem using an ADMM solver (e.g., OSQP) [S1, unofficial].
   - The ADMM method splits the objective into a data term and a prior term, alternating between subproblems and maintaining a dual variable $u$ to enforce consensus [S1, unofficial].
   - If the solver returns an unconstrained solution $x'$, enforce constraints via truncation: $x_i = \max(\min(x'_i, 1), 0)$ [S1, unofficial].

**Formulas:**
- Objective: $\min_x \frac{1}{2} x^T Q x + c^T x$ [S1, unofficial].
- Hessian: $Q = A^T A$ [S1, unofficial].
- Linear term: $c = -A^T b$ [S1, unofficial].
- Constraint enforcement: $x_i = \text{clip}(x'_i, 0, 1)$ [S1, unofficial].

**Parameters:**
- Solver: OSQP [S1, unofficial].
- Hardware: CPU implementation is standard; GPU implementation exists but offers no significant speed-up for the small matrix sizes resulting from normal equations [S1, unofficial].

**Edge Cases and Pitfalls:**
- Ill-conditioning: The condition number of $Q$ is the square of the condition number of $A$. This makes the method prone to stability issues for large, ill-conditioned systems [S1, unofficial].
- Convexity: The ordinary least squares problem is convex, as the Hessian $H(x) = 2 A^T A$ is positive semi-definite [S1, unofficial].

## Subtopics

### [ADMM Solver](10107-admm-solver.md)
The ADMM Solver is an optimization method for solving separable minimization problems of the form $\min_x f(x) + g(x)$ by splitting the objective into a data term $f(x)$ and a prior term $g(x). It differs from direct solvers by decoupling these terms into alternating subproblems, allowing specialized solvers for each component, and differs from primal-dual methods by explicitly maintaining a dual variable $u$ to enforce consensus between the split variables.

## Operating characteristics
- **Time Complexity:** Depends on the sparsity of $A$ and the efficiency of the ADMM solver (OSQP). Construction of $Q = A^T A$ is $O(m n^2)$ or less if sparse.
- **Memory Complexity:** Requires storage for $Q$ ($n \times n$) and $c$ ($n$).
- **Performance:** For the small size of $A^T A$ resulting from normal equations, no significant speed-up was observed with GPU implementations compared to CPU implementations [S1, unofficial].
- **Strengths:** Handles box constraints explicitly; leverages sparse solvers.
- **Limitations:** Prone to stability issues if the matrix is ill-conditioned due to the quadratic condition number [S1, unofficial].
- **When to use:** When box constraints are required and the system size is small enough to avoid severe ill-conditioning.
- **When not to use:** For very large, ill-conditioned systems where avoiding $A^T A$ is preferable [S1, unofficial].

## Background
The method is based on formulating box-constrained least squares as a quadratic programming problem, a technique used in opacity optimization approaches [GRT13, AZD17] [S1, unofficial]. The ADMM method was first introduced by Boyd et al. [BPC∗ 11] for solving convex optimization problems [S1, unofficial]. The OSQP solver library [SBG∗ 20] is commonly used for this purpose [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
