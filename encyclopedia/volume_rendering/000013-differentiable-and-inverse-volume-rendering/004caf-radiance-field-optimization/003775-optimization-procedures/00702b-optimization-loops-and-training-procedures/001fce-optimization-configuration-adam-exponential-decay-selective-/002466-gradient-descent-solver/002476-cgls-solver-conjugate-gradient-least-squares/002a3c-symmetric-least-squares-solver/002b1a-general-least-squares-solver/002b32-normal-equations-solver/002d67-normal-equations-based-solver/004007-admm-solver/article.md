# ADMM Solver
Also called: Alternating Direction Method of Multipliers, Standard ADMM

## Overview
The ADMM Solver is a proximal algorithm used to solve separable minimization problems of the form $\min_x f(x) + g(x)$ or $\min_x f(x) + g(Dx)$, where $f(x)$ is a data fidelity term and $g(x)$ is a prior or regularization term [S1, unofficial]. It sets itself apart from its parent [Normal Equations-based Solver](5483-normal-equations-based-solver.md) by handling composite objectives with non-smooth or complex constraints (like sparsity or total variation) that are difficult to solve directly via normal equations, by splitting the problem into simpler sub-problems [S1, unofficial].

## How it works
**Inputs:**
- Objective functions $f(x)$ and $g(\cdot)$, where $f$ is typically a quadratic data term (e.g., $\|Ax - p\|_2^2$) and $g$ is a convex prior (e.g., $\ell_1$ norm or Total Variation) [S1, unofficial].
- Linear operator $D$ (identity $I$ for standard form, or a transform matrix for generalized form) [S1, unofficial].
- Penalty parameter $\rho > 0$ [S1, unofficial].
- Proximal parameter $\mu$ (required for Linearized ADMM, optional for Standard ADMM depending on implementation) [S2].
- Maximum iterations $T$ or convergence tolerance [S1, unofficial].

**Outputs:**
- Reconstructed volume $x$ [S1, unofficial].

**Data Structures:**
- $x \in \mathbb{R}^n$: Current primal variable (reconstruction).
- $z \in \mathbb{R}^m$: Auxiliary variable for the prior term.
- $u \in \mathbb{R}^m$: Scaled dual variable (Lagrange multiplier).
- $y \in \mathbb{R}^m$: Dual variable (used in Linearized ADMM).

**Numbered Steps (Standard ADMM for $\min_x f(x) + g(Dx)$):**
1. Initialize $x^0, z^0, u^0$.
2. For $k = 0$ to $T-1$:
   a. **x-update:** Solve $x^{k+1} = \operatorname{argmin}_x \left( f(x) + \frac{\rho}{2} \|Dx - z^k + u^k\|_2^2 \right)$.
      - If $f(x) = \frac{1}{2}\|Ax - p\|_2^2$, this is a least-squares problem. In implementations like RTK, this step is solved using a Conjugate Gradient (CG) method [S1, unofficial].
   b. **z-update:** Solve $z^{k+1} = \operatorname{argmin}_z \left( g(z) + \frac{\rho}{2} \|Dx^{k+1} - z + u^k\|_2^2 \right)$.
      - This is equivalent to the proximal operator: $z^{k+1} = \operatorname{prox}_{\frac{1}{\rho}g}(Dx^{k+1} + u^k)$ [S1, unofficial].
   c. **u-update:** $u^{k+1} = u^k + Dx^{k+1} - z^{k+1}$ [S1, unofficial].
3. Return $x^T$.

**Special Case: Standard Form ($D=I$):**
When $D=I$, the updates simplify to proximal operators:
1. $x^{k+1} = \operatorname{prox}_{\frac{1}{\rho}f}(x^k - u^k)$
2. $z^{k+1} = \operatorname{prox}_{\frac{1}{\rho}g}(x^{k+1} + u^k)$
3. $u^{k+1} = u^k + x^{k+1} - z^{k+1}$ [S1, unofficial].

**Pseudocode (Standard ADMM with CG for x-update):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def admm_solver(A, p, prox_f, prox_g, rho, max_iter, tol):
    """
    Solves min_x f(x) + g(x) using Standard ADMM.
    Assumes D = I.
    prox_f and prox_g are functions implementing the proximal operators.
    """
    n = len(p)
    # Initialize variables
    x = zeros(n)
    z = zeros(n)
    u = zeros(n)
    
    for k in range(max_iter):
        # x-update: argmin_x f(x) + (rho/2) * ||x - (z - u)||^2
        # This is equivalent to solving a regularized least squares problem
        # if f(x) is quadratic. Here we assume prox_f handles the specific f.
        # If f(x) = 0.5 * ||Ax - p||^2, the x-update is:
        # x^{k+1} = argmin_x 0.5 * ||Ax - p||^2 + (rho/2) * ||x - (z^k - u^k)||^2
        # This can be solved via Conjugate Gradient (CG) [S1, unofficial].
        
        # Define the effective RHS for the least squares problem
        rhs = z - u
        
        # Solve (A^T A + rho I) x = A^T p + rho * rhs using CG
        # Note: In practice, A^T A is not formed explicitly.
        # CG iteration:
        x = conjugate_gradient_solve(A, rhs, rho, p)
        
        # z-update: proximal operator of g
        z = prox_g(x + u, 1.0/rho)
        
        # u-update: dual ascent
        u = u + x - z
        
        # Check convergence
        # Primal residual: ||x - z||
        # Dual residual: ||z^{k} - z^{k-1}||
        # See [S3] for specific metrics
        if k > 0:
            r_pri = norm(x - z)
            r_dual = norm(z - z_prev)
            if r_pri < tol and r_dual < tol:
                break
        z_prev = z.copy()
        
    return x

def conjugate_gradient_solve(A, rhs, rho, p):
    """
    Solves (A^T A + rho I) x = A^T p + rho * rhs
    using Conjugate Gradient method.
    """
    n = len(rhs)
    x = zeros(n)
    b = mat_vec_mul(A, p) + rho * rhs # A^T p + rho * rhs
    
    # Initial residual
    r = b - mat_vec_mul(A, x) # A^T A x + rho x - b. At x=0, r = b.
    # Actually, standard CG for (H)x=b where H = A^T A + rho I
    # r_0 = b - H x_0 = b
    
    p_vec = r.copy()
    r_dot_r = dot(r, r)
    
    for i in range(100): # Max inner iterations
        Ap = mat_vec_mul(A, p_vec)
        # H * p_vec = A^T (A p_vec) + rho p_vec
        H_p = mat_vec_mul(A, Ap) + rho * p_vec
        
        alpha = r_dot_r / dot(p_vec, H_p)
        x = x + alpha * p_vec
        r = r - alpha * H_p
        r_dot_r_new = dot(r, r)
        
        if r_dot_r_new < 1e-10:
            break
            
        beta = r_dot_r_new / r_dot_r
        p_vec = r + beta * p_vec
        r_dot_r = r_dot_r_new
        
    return x

def mat_vec_mul(A, v):
    # Placeholder for matrix-vector multiplication
    # In tomography, this is Forward Projection and Back Projection
    return A @ v

def dot(v1, v2):
    return sum(v1 * v2)

def norm(v):
    return sqrt(dot(v, v))
```

**Parameters:**
- $\rho$: Augmented Lagrangian penalty parameter. Typical values depend on the condition number of $A$ and $D$ [S1, unofficial].
- $\mu$: Proximal parameter for Linearized ADMM. Constraint: $\mu \rho \|K\|^2 \le 1$ [S2].

**Edge Cases and Pitfalls:**
- **Precision:** The x-update involves solving a linear system. If $A^T A + \rho I$ is ill-conditioned, CG may converge slowly or fail. Regularization or preconditioning is often required [S1, unofficial].
- **Boundaries:** The method assumes the domain of $x$ and $z$ allows the proximal operators to be well-defined. For box constraints ($0 \le x_i \le 1$), the proximal operator includes clamping [S1, unofficial].
- **Convergence:** ADMM convergence is not guaranteed for non-convex problems. For convex problems, it converges, but the rate can be slow if $\rho$ is poorly chosen [S1, unofficial].

## Subtopics

### [L1 Shearlet Regularized CT Reconstruction (ADMM)](10811-l1-shearlet-regularized-ct-reconstruction-admm.md)
This method solves the L1 shearlet regularized computed tomography (CT) reconstruction problem using the Alternating Direction Method of Multipliers (ADMM). It minimizes a composite objective consisting of a weighted L1 norm of shearlet coefficients and a quadratic data fidelity term, subject to non-negativity constraints on the image. It sets itself apart from standard filtered backprojection by enforcing sparsity in the shearlet domain to handle limited-angle artifacts, and from its parent [ADMM Solver](10107-admm-solver.md) by specifically structuring the proximal steps for shearlet shrinkage and non-negativity clamping.

### Linearized Alternating Direction Method of Multipliers (LADMM)
A proximal algorithm used to solve optimization problems of the form $\arg \min_v F(v) + G(Kv)$ [S2]. It modifies the standard ADMM x-update to avoid solving a linear system by linearizing the quadratic term, at the cost of introducing a proximal parameter $\mu$ [S2].
- **Initialization:** $v^{(0)} \in \mathbb{R}^N$, $z^{(0)} \in \mathbb{R}^d$, $y^{(0)} = 0_d$ [S2].
- **Iterations ($t = 1$ to $T_1$):**
  1. $v^{(t+1)} = \text{prox}_{\mu F}(v^{(t)} - \mu \rho K^T (K v^{(t)} - z^{(t)} + y^{(t)}))$ [S2].
  2. $z^{(t+1)} = \text{prox}_{\rho G}(K v^{(t+1)} + y^{(t)})$ [S2].
  3. $y^{(t+1)} = y^{(t)} + K v^{(t+1)} - z^{(t+1)}$ [S2].
- **Parameters:** $\mu$ (proximal parameter) and $\rho$ (penalty parameter) must satisfy $\mu \rho \|K\|^2 \le 1$ [S2]. $T_1$ is the number of iterations [S2]. $z$ and $y$ are additional variables of size $d$ [S2].

### ADMM Convergence Metrics
Metrics used to track the convergence of the ADMM optimization loop by measuring violations of primal and dual feasibility [S3].
- **Primal Residual:** $\text{RMSE}(A x_t, p_t)$ [S3].
- **Dual Residual:** $\text{RMSE}(A x_t, A x_{t-1})$ [S3].
- **Comparison:** Raskar codes yield slightly faster convergence (lower residuals per iteration) compared to boxcar codes [S3].

## Operating characteristics
- **Time Complexity:** Each iteration involves one x-update (solving a linear system, typically $O(n^2)$ or $O(n \log n)$ with FFT/preconditioning) and two proximal updates ($O(n)$). The number of iterations depends on the problem conditioning and tolerance [S1, unofficial].
- **Memory Complexity:** $O(n)$ for storing $x, z, u$ (and $v, y$ for LADMM) [S1, unofficial][S2].
- **Strengths:** Handles non-smooth regularizers (like $\ell_1$) efficiently by splitting them into simple proximal steps. Decomposes complex problems into simpler sub-problems [S1, unofficial].
- **Limitations:** Convergence can be slow for ill-conditioned problems. The x-update may require an iterative solver (like CG), adding inner-loop complexity [S1, unofficial].
- **When to use:** When the objective function is separable into a data term and a prior term, and the prior term has a simple proximal operator (e.g., sparsity, total variation) [S1, unofficial].
- **When not to use:** When the problem is strictly convex and smooth, and a direct solver (like Cholesky) is feasible and faster. Or when the x-update cannot be solved efficiently [S1, unofficial].

## Background
ADMM was introduced as a technique to solve separable minimization problems [S1, unofficial]. It is widely used in image processing and tomography, including implementations in the Reconstruction ToolKit (RTK) for total variation-regularized reconstruction [S1, unofficial]. Linearized ADMM (LADMM) was developed to avoid solving linear systems in the x-update, making it suitable for large-scale problems [S2]. The method has been applied in various contexts, including electron microscopy tilt-series reconstruction [S2] and joint temporal de-blurring and tomographic reconstruction [S3].

## Sources
[S1] Industrial Computed Tomography using Proximal Algorithm, https://doi.org/10.25781/kaust-th10i, fetched 2026-10-27 (unofficial)
[S2] GPU Accelerated 3D Tomographic Reconstruction and Visualization From Noisy Electron Microscopy Tilt-Series., https://doi.org/10.1109/tvcg.2022.3230445, fetched 2026-10-28
[S3] CodEx: A Modular Framework for Joint Temporal De-blurring and Tomographic Reconstruction, https://arxiv.org/abs/2111.06069v3, fetched 2026-10-26
