# CGLS with Non-Negativity Projection

Also called: CGLS with projection, Conjugate Gradient Least Squares (CGLS) with Non-Negativity Projection, regularized conjugate gradient

## Overview
This method solves the linear least squares problem $S a = p$ for the coefficient vector $a$ by applying the Conjugate Gradient Least Squares (CGLS) algorithm [S1]. It is specifically designed for tomographic reconstruction where the physical quantity (density) must be non-negative. Unlike the standard [CGLS Solver (Conjugate Gradient Least Squares)](3285-cgls-solver-conjugate-gradient-least-squares.md) which operates in the full space, this variant projects the solution onto the non-negative subspace at every iteration to enforce physical validity, using the number of iterations as an implicit regularization parameter [S1].

## How it works
**Inputs:**
- $S$: The forward projection matrix (basis function contributions to pixels). Columns correspond to basis functions; rows correspond to pixels [S1].
- $p$: The vector of observed pixel values (e.g., from multi-view images) [S1].
- $n$: The number of basis functions (size of $a$).
- $m$: The number of pixels (size of $p$).

**Outputs:**
- $a$: The reconstructed non-negative coefficient vector [S1].

**Data Structures:**
- $a \in \mathbb{R}^n$: Coefficient vector.
- $p \in \mathbb{R}^m$: Observation vector.
- $r \in \mathbb{R}^m$: Residual vector.
- $d \in \mathbb{R}^n$: Search direction vector.
- $S$: Matrix-free or sparse representation. Explicit computation of $S^T S$ is avoided to save memory, as $S^T S$ is typically dense [S1].

**Algorithm Steps:**

1.  **Initialization:**
    Set $k = 0$.
    $$a_0 = \mathbf{0}$$
    $$r_0 = p - S a_0 = p$$
    $$d_0 = S^T r_0$$

2.  **Iteration Loop:** For $k = 1, 2, \dots$:
    a. Compute the step size $\alpha_k$:
    $$\alpha_k = \frac{\|S^T r_{k-1}\|_2^2}{\|S d_{k-1}\|_2^2}$$
    b. Update the solution:
    $$a_k = a_{k-1} + \alpha_k d_{k-1}$$
    c. Update the residual:
    $$r_k = r_{k-1} - \alpha_k S d_{k-1}$$
    d. Compute the next step size $\beta_k$:
    $$\beta_k = \frac{\|S^T r_k\|_2^2}{\|S^T r_{k-1}\|_2^2}$$
    e. Update the search direction:
    $$d_k = S^T r_k + \beta_k d_{k-1}$$
    f. **Non-Negativity Projection:**
    Set any negative entries of $a_k$ to zero:
    $$a_k = \max(a_k, 0)$$
    This ensures the density field remains non-negative, assuming non-negative basis functions are used [S1].

3.  **Termination:**
    The iteration stops when the L-curve criterion indicates the optimal trade-off between solution smoothness and data fidelity [S1].
    - Plot the curve of points $(\log \|a_k\|_2, \log \|S a_k - p\|_2)$ for $k=1, \dots, K_{max}$.
    - Identify the point of highest curvature on this curve.
    - The number of iterations $k$ at this point serves as the regularization parameter [S1].

**Parameters:**
- **Regularization:** The number of iterations acts as the regularization parameter. No explicit regularization term (like Tikhonov) is added to the equations; instead, early stopping prevents overfitting and ghosting artifacts [S1].
- **Basis Functions:** Non-negative basis functions (e.g., box functions) are required to ensure that non-negative coefficients yield a non-negative density field [S1].

**Edge Cases and Pitfalls:**
- **Ghosting Artifacts:** If the number of views is insufficient to constrain the solution, the full system may produce ghosting artifacts. Restricting the linear system to the visual hull (removing columns for basis functions outside the visual hull) mitigates this [S1].
- **Precision:** The method avoids forming $S^T S$, which is crucial for numerical stability and memory efficiency when $n$ is large [S1].
- **Projection Impact:** Projecting $a_k$ to zero affects the subsequent residual and search direction calculations. The standard CGLS recurrence relations are applied *after* the projection in the described variant, or the projection is applied to the solution vector while the residual update uses the unprojected step. The source text implies the projection is applied to $a_k$ after the update step $a_k = a_{k-1} + \alpha_k d_{k-1}$, but the residual $r_k$ and direction $d_k$ updates follow the standard CGLS formulas based on the *current* residual. Note: Standard CGLS does not include projection. The specific variant described applies projection to $a_k$ but continues the CG process on the normal equations. The exact interaction of projection with the conjugacy of $d$ is a deviation from pure CGLS, but the source specifies this sequence [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cgls_non_negative(S, p, max_iter=1000, tol=1e-6):
    """
    Solves S @ a = p with non-negativity constraint using CGLS.
    S is a function or object that computes S @ v and S^T @ w.
    """
    n = len(p) # Number of basis functions (size of a)
    # Note: S is m x n. p is m x 1.
    
    # Initialization
    a = [0.0] * n
    r = list(p) # r_0 = p - S @ a_0 = p
    d = transpose_multiply(S, r) # d_0 = S^T @ r_0
    
    k = 0
    # Store L-curve points for termination
    l_curve_points = []
    
    while k < max_iter:
        # Compute S @ d_{k-1}
        S_d = multiply(S, d)
        
        # Compute norms
        norm_S_T_r_prev_sq = dot_product(transpose_multiply(S, r), transpose_multiply(S, r))
        # Wait, standard CGLS uses ||S^T r_{k-1}||^2 / ||S d_{k-1}||^2 for alpha
        # But we need S^T r_{k-1} for beta later.
        
        # Let's follow the source formula strictly:
        # alpha_k = ||S^T r_{k-1}||_2^2 / ||S d_{k-1}||_2^2
        
        S_T_r_prev = transpose_multiply(S, r)
        norm_S_T_r_prev_sq = dot_product(S_T_r_prev, S_T_r_prev)
        norm_S_d_sq = dot_product(S_d, S_d)
        
        if norm_S_d_sq == 0:
            break
            
        alpha = norm_S_T_r_prev_sq / norm_S_d_sq
        
        # Update solution
        for i in range(n):
            a[i] = a[i] + alpha * d[i]
            
        # Project to non-negative subspace
        for i in range(n):
            if a[i] < 0:
                a[i] = 0.0
                
        # Update residual
        # r_k = r_{k-1} - alpha * S @ d_{k-1}
        for i in range(len(r)):
            r[i] = r[i] - alpha * S_d[i]
            
        # Compute L-curve point
        norm_a = dot_product(a, a) ** 0.5
        norm_r = dot_product(r, r) ** 0.5
        if norm_r > 1e-12:
            l_curve_points.append((norm_a, norm_r))
            
        # Check termination via L-curve curvature (simplified check for implementation)
        # In practice, compute curvature for all points and pick max.
        # For this pseudocode, we assume an external evaluator or loop finishes.
        
        # Compute beta_k
        S_T_r_curr = transpose_multiply(S, r)
        norm_S_T_r_curr_sq = dot_product(S_T_r_curr, S_T_r_curr)
        
        if norm_S_T_r_prev_sq == 0:
            break
            
        beta = norm_S_T_r_curr_sq / norm_S_T_r_prev_sq
        
        # Update search direction
        # d_k = S^T r_k + beta * d_{k-1}
        for i in range(n):
            d[i] = S_T_r_curr[i] + beta * d[i]
            
        k += 1
        
    return a, l_curve_points

def multiply(S, v):
    # Computes S @ v
    pass

def transpose_multiply(S, w):
    # Computes S^T @ w
    pass

def dot_product(u, v):
    return sum(x * y for x, y in zip(u, v))
```

## Subtopics

### [Conjugate Gradient Solver for Compressive Volume Rendering](4270-conjugate-gradient-solver-for-compressive-volume-rendering.md)
Uses CG for compressive sensing; differs by focusing on sparsity-promoting norms rather than non-negativity projection in the linear solver loop.

### [Symmetric Least Squares Solver](4723-symmetric-least-squares-solver.md)
Solves symmetric systems; CGLS is preferred for rectangular $S$ where $S^T S$ is formed implicitly.

### [Adaptive Grid Optical Tomography](13834-adaptive-grid-optical-tomography.md)
Uses adaptive grids for efficiency; CGLS with projection is the solver used within such frameworks for the linear step.

### [Matrix-Free PCG Solver](16777-matrix-free-pcg-solver.md)
Preconditioned CG; CGLS is a specific variant of PCG without preconditioning, tailored for least squares.

## Operating characteristics
- **Time Complexity:** Each iteration requires two matrix-vector multiplications ($S d$ and $S^T r$) and several vector operations. Convergence depends on the condition number of $S^T S$.
- **Memory Complexity:** $O(m + n)$ for storing vectors $a, r, d, p$. Avoids storing $S^T S$ ($O(n^2)$), which is critical for large $n$ [S1].
- **Strengths:** Enforces physical non-negativity constraints directly during iteration; numerically stable for ill-conditioned systems; memory efficient.
- **Limitations:** Convergence can be slow for ill-conditioned problems; the projection step may disrupt the conjugacy of search directions, potentially affecting convergence rate compared to unconstrained CGLS.
- **When to use:** Tomographic reconstruction of density fields (e.g., flames, smoke) where negative values are physically meaningless and the system is underdetermined or noisy [S1].

## Background
The method originates from image-based tomographic reconstruction of flames, specifically addressing the need for non-negative density fields from multi-view video sequences [S1]. It builds on the CGLS variant developed by Hanke [Han98] to solve normal equations without explicit matrix formation. The L-curve criterion for termination is adapted from Åström [Å96] and Hanke [Han98]. The approach is particularly relevant for visual hull-restricted systems to reduce ghosting artifacts caused by insufficient view counts [S1].

## Sources
[S1] Image-based tomographic reconstruction of flames, https://doi.org/10.1145/1028523.1028572, fetched 2026-10-26
