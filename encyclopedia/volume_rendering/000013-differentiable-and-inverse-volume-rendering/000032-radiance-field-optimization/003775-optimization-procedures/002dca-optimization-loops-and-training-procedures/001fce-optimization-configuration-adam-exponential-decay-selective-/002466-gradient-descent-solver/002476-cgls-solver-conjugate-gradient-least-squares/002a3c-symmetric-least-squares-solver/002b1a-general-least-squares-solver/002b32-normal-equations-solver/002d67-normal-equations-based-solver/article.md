# Normal Equations-based Solver

Also called: A^T A Solver, Constrained Least Squares, Least Squares via Normal Equations, Quadratic Programming Solver (Box-Constrained)

## Overview
The Normal Equations-based Solver addresses the box-constrained least squares problem $\forall i \in [1,...,n] : 0 \le x_i \le 1$ by formulating it as a quadratic programming (QP) problem [S1, unofficial]. It is distinguished from direct normal equation solvers by its explicit handling of bound constraints via the Alternating Direction Method of Multipliers (ADMM), whereas standard normal equation solvers typically ignore constraints or apply post-hoc truncation [S1, unofficial]. It serves as a robust alternative when the solution space is strictly bounded, such as in opacity or transfer function optimization, though it incurs higher computational overhead than unconstrained methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Matrix $A \in \mathbb{R}^{m \times n}$ and vector $b \in \mathbb{R}^m$ from the linear system $Ax \approx b$.
- Output: Vector $x \in \mathbb{R}^n$ such that $0 \le x_i \le 1$ for all $i$, minimizing the constrained least squares error.

**Data Structures:**
- The core data structure is the quadratic programming formulation defined by the Hessian matrix $Q$ and linear term vector $c$.
- $Q = A^T A \in \mathbb{R}^{n \times n}$.
- $c = -A^T b \in \mathbb{R}^n$.
- The solver exploits the sparsity of $Q$ if $A$ is sparse [S1, unofficial].

**Steps:**
1. **Formulate QP Problem:** Compute $Q = A^T A$ and $c = -A^T b$. The objective is to minimize:
   $$ \min_x \frac{1}{2} x^T Q x + c^T x $$
   subject to $0 \le x_i \le 1$ for all $i \in [1, \dots, n]$ [S1, unofficial].
2. **Solve via ADMM:** Use the Alternating Direction Method of Multipliers (ADMM) to solve the convex optimization problem. ADMM decomposes the problem into subproblems that are easier to solve, handling the separable constraint structure [S1, unofficial].
3. **Implementation:** Utilize the OSQP (Operator Splitting Quadratic Program) solver library, which supports ADMM and exploits sparsity [S1, unofficial].
4. **Constraint Enforcement:** If an unconstrained solver is used instead of ADMM, enforce constraints via truncation:
   $$ x'_i = \max(\min(x_i, 1), 0) $$
   However, the primary method described uses explicit constrained solving [S1, unofficial].

**Parameters:**
- Solver: OSQP [S1, unofficial].
- Algorithm: ADMM [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Ill-conditioning:** The condition number of $Q$ is the square of the condition number of $A$. For large, ill-conditioned systems, this formulation is numerically unstable [S1, unofficial].
- **Performance:** While a GPU implementation of OSQP exists, no significant speed-up was observed for the small matrix sizes resulting from normal equations compared to CPU implementations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normal_equations_constrained_solver(A, b):
    """
    Solves box-constrained least squares using Normal Equations and ADMM.
    A: Sparse matrix (m x n)
    b: Vector (m,)
    Returns: x (n,), constrained to [0, 1]
    """
    # Step 1: Formulate QP problem
    # Q = A^T A
    Q = A.T @ A
    # c = -A^T b
    c = -A.T @ b
    
    # Step 2: Solve using OSQP with ADMM
    # OSQP expects the problem in the form:
    # min 0.5 * x^T P x + q^T x
    # subject to l <= Ax <= u
    # Here, P = Q, q = c
    # Constraints: 0 <= x <= 1
    
    import osqp
    import scipy.sparse as sp
    
    # Ensure Q is sparse if A is sparse
    if sp.issparse(Q):
        P = Q
    else:
        P = sp.csc_matrix(Q)
        
    q = c
    
    # Define constraints
    l = np.zeros(len(c))
    u = np.ones(len(c))
    
    # Construct constraint matrix (identity for simple bounds)
    M = len(c)
    A_con = sp.eye(M, format='csc')
    
    # Setup OSQP problem
    prob = osqp.OSQP()
    prob.setup(P=P, q=q, A=A_con, l=l, u=u, verbose=False)
    
    # Solve
    res = prob.solve()
    
    if res.info.status != 'solved':
        raise ValueError("OSQP did not converge to a solution")
        
    return res.x
```

## Subtopics

### [ADMM Solver](10107-admm-solver.md)
A technique to solve minimization problems of the form $\min_x f(x) + g(x)$ with variable $x$ and objectives $f(x)$ and $g(x)$ that are separable. $f(x)$ is a data term measuring goodness of fit, $g(x)$ is a prior term measuring deviation from expectations. It is chosen here for its ability to handle the box constraints efficiently via operator splitting.

## Operating characteristics
- **Time Complexity:** Dominated by the formation of $Q = A^T A$ ($O(mn^2)$ or less if sparse) and the iterative ADMM convergence. Convergence speed depends on the condition number of $Q$, which is poor for ill-conditioned $A$ [S1, unofficial].
- **Memory Complexity:** Requires storing $Q$ ($n \times n$). If $A$ is sparse, $Q$ may be denser, increasing memory usage [S1, unofficial].
- **Measured Performance:** For the small matrix sizes resulting from normal equations in typical volume rendering applications, no significant speed-up was observed using a GPU implementation of OSQP compared to the CPU implementation [S1, unofficial].
- **Strengths:** Explicitly handles box constraints ($0 \le x_i \le 1$) without post-hoc truncation errors; convex formulation guarantees a global minimum [S1, unofficial].
- **Limitations:** Numerically unstable for ill-conditioned systems due to squaring of the condition number; potentially slower than unconstrained solvers due to iterative nature [S1, unofficial].
- **When to use:** When strict adherence to bounds is required and the system size is small enough that the quadratic condition number does not cause numerical failure.
- **When not to use:** For large, ill-conditioned systems where CGLS or other preconditioned methods are preferred [S1, unofficial].

## Background
The formulation of least squares as a quadratic programming problem was introduced in the context of transfer function optimization for volume rendering [S1, unofficial]. The use of ADMM for such problems traces back to Boyd et al. [BPC* 11], and the specific implementation relies on the OSQP solver library [SBG* 20]. The approach was evaluated as a reference for handling constraints in RGB*α optimization, noting that while more general QP solvers exist, they share the stability drawbacks of normal equations [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
