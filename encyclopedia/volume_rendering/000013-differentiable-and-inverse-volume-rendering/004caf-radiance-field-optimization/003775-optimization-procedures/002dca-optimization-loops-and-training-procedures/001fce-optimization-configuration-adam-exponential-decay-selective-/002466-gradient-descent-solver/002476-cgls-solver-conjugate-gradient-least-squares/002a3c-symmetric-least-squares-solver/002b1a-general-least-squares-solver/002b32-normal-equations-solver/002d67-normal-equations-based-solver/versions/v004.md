# Normal Equations-based Solver

Also called: A^T A Solver, Constrained Least Squares, Least Squares via Normal Equations, Quadratic Programming Solver (Box-Constrained)

## Overview
The Normal Equations-based Solver addresses the box-constrained least squares problem $\forall i \in [1,...,n] : 0 \le x_i \le 1$ by formulating it as a quadratic programming (QP) problem [S1, unofficial]. It differs from direct unconstrained solvers by explicitly handling the box constraints via the Alternating Direction Method of Multipliers (ADMM) [S1, unofficial]. Unlike iterative least-squares solvers that avoid forming the normal equations, this method explicitly constructs the Hessian $Q = A^T A$, which shares the drawback of quadratic condition number amplification [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Matrix $A$ (size $m \times n$), vector $b$ (size $m$), and constraint bounds $[0, 1]$ for each element of $x$ [S1, unofficial].
- Output: Vector $x$ (size $n$) minimizing the constrained objective [S1, unofficial].

**Data Structures:**
- Exact layouts and bit packing are not given in the sources [S1, unofficial]. The method stores $Q = A^T A$ and $c = -A^T b$ as sparse structures to exploit sparsity [S1, unofficial].

**Steps:**
1. Construct the quadratic programming parameters by computing $Q = A^T A$ and $c = -A^T b$ [S1, unofficial].
2. Formulate the objective as $\min_x \frac{1}{2} x^T Q x + c^T x$ subject to $0 \le x_i \le 1$ for all $i \in [1,...,n]$ [S1, unofficial].
3. Solve the constrained QP problem using an ADMM solver (e.g., OSQP) [S1, unofficial].
4. The ADMM method splits the objective into a data term and a prior term, alternating between subproblems and maintaining a dual variable $u$ to enforce consensus [S1, unofficial].

**Formulas:**
- Objective: $\min_x \frac{1}{2} x^T Q x + c^T x$, where $x \in \mathbb{R}^n$, $Q \in \mathbb{R}^{n \times n}$, and $c \in \mathbb{R}^n$ [S1, unofficial].
- Hessian: $Q = A^T A$, where $A \in \mathbb{R}^{m \times n}$ [S1, unofficial].
- Linear term: $c = -A^T b$, where $b \in \mathbb{R}^m$ [S1, unofficial].
- Constraints: $0 \le x_i \le 1$ for all $i \in [1,...,n]$ [S1, unofficial].

**Parameters:**
- Typical values are not given in the sources [S1, unofficial].
- Solver: OSQP [S1, unofficial].
- Hardware: CPU implementation is standard; a GPU implementation exists but offers no significant speed-up for the small matrix sizes resulting from normal equations [S1, unofficial].

**Edge Cases and Pitfalls:**
- Ill-conditioning: The condition number of $Q$ is the square of the condition number of $A$, making the method prone to stability issues for large, ill-conditioned systems [S1, unofficial].
- Convexity: The ordinary least squares problem is convex, as the Hessian $H(x) = 2 A^T A$ is positive semi-definite [S1, unofficial].
- Truncation fallback: If constraints are not enforced explicitly by the solver, values are truncated via $x'_i = \max\{\min\{x_i, 1\}, 0\}$ [S1, unofficial].

## Subtopics

### [ADMM Solver](10107-admm-solver.md)
The ADMM Solver is an optimization method for solving separable minimization problems of the form $\min_x f(x) + g(x)$ by splitting the objective into a data term $f(x)$ and a prior term $g(x)$. It differs from direct solvers by decoupling these terms into alternating subproblems, allowing specialized solvers for each component, and differs from primal-dual methods by explicitly maintaining a dual variable $u$ to enforce consensus between the split variables.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial]. The solver exploits sparsity in the problem definition [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Performance:** For the small size of $A^T A$ resulting from normal equations, no significant speed-up was observed with GPU implementations compared to CPU implementations [S1, unofficial].
- **Strengths:** Handles box constraints explicitly; leverages sparse solvers [S1, unofficial].
- **Limitations:** Prone to stability issues if the matrix is ill-conditioned due to the quadratic condition number [S1, unofficial].
- **When to use:** When box constraints are required and the system size is small enough to avoid severe ill-conditioning [S1, unofficial].
- **When not to use:** For very large, ill-conditioned systems where avoiding $A^T A$ is preferable [S1, unofficial].

## Background
The method is based on formulating box-constrained least squares as a quadratic programming problem, a technique used in opacity optimization approaches [GRT13, AZD17] [S1, unofficial]. The ADMM method was first introduced by Boyd et al. [BPC∗ 11] for solving convex optimization problems [S1, unofficial]. The OSQP solver library [SBG∗ 20] is commonly used for this purpose [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
