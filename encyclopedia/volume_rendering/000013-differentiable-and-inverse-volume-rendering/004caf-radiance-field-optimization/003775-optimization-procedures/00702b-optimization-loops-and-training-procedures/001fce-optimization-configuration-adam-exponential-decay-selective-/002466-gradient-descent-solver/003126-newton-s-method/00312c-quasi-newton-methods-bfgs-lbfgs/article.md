# Quasi-Newton Methods (BFGS/LBFGS)
Also called: BFGS, BFGS/LBFGS Optimization, LBFGS

## Overview
Quasi-Newton methods utilize second-order information for optimization by approximating the Hessian matrix using zero- or first-order information [S1]. Unlike [Newton's Method](6442-newton-s-method.md), which requires explicit computation of the Hessian, these methods build an approximation iteratively [G]. In differentiable rendering, they are distinguished by their sensitivity to noise; while they can converge on smooth tasks, they often fail on noisy derivative estimations common in rendering without specific damping or line-search modifications [S1].

## How it works
**Inputs and Outputs:**
- Input: Objective function $f(\theta)$, initial parameter vector $\theta_0$, and iteration parameters [G].
- Output: Optimized parameter vector $\theta^*$ [G].

**Data Structures:**
- $\theta$: Parameter vector of size $d$ (problem sizes are typically large, e.g., neural network weights) [S1].
- $B$: $d \times d$ matrix approximating the inverse Hessian [G].

**Steps:**
1. **Initialization:** Set $\theta_0$, $B_0 = I$ (identity matrix), and iteration counter $k=0$ [G].
2. **Gradient Evaluation:** Compute the gradient $g_k = \nabla f(\theta_k)$ [G].
3. **Search Direction:** Compute search direction $p_k = -B_k g_k$ [G].
4. **Line Search:** Determine step size $\alpha_k$ satisfying Wolfe conditions (e.g., Armijo-Wolfe) [S1]. Update $\theta_{k+1} = \theta_k + \alpha_k p_k$ [G].
5. **Curvature Update:** Compute difference vectors $s_k = \theta_{k+1} - \theta_k$ and $y_k = g_{k+1} - g_k$ [G].
6. **Hessian Update:** Update the approximation $B_{k+1}$ using the BFGS rank-2 update formula [G]:
   $$ B_{k+1} = B_k + \frac{y_k y_k^T}{y_k^T s_k} - \frac{B_k s_k s_k^T B_k}{s_k^T B_k s_k} $$
   *Note: If $y_k^T s_k \leq 0$, the update is skipped to maintain positive definiteness [G].*
7. **Iteration:** Increment $k$. If convergence criteria are not met, return to Step 2 [G].

**Parameters:**
- Line search tolerance: Standard Wolfe conditions [S1].
- Damping: Used in "Damped BFGS" to handle non-positive semi-definite (non-PSD) Hessians or noisy gradients [S1].

**Edge Cases and Pitfalls:**
- **Noisy Derivatives:** Vanilla BFGS, BFGS with Armijo-Wolfe line search, and LBFGS only converge for smooth tasks (e.g., the QUAD task) but fail on noisy derivative estimations typical in rendering [S1].
- **Non-PSD Hessians:** Non-PSD Hessians are not necessarily a problem if the right combination of estimator and optimizer is used, as demonstrated in sphere's Phong BRDF optimization [S1].
- **Scale:** Problem sizes are typically large (e.g., neural network weights) compared to the number of physical rendering parameters [S1].

**Pseudocode (BFGS with Inverse Hessian Approximation):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def outer(a, b):
    return [[x * y for y in b] for x in a]

def mat_vec(M, v):
    return [sum(row[j] * v[j] for j in range(len(v))) for row in M]

def norm(v):
    return dot(v, v) ** 0.5

def identity(n):
    return [[1 if i == j else 0 for j in range(n)] for i in range(n)]

def bfgs_update(B, s, y):
    sy = dot(s, y)
    if sy <= 1e-10:
        return B
    Bs = mat_vec(B, s)
    yBs = dot(y, Bs)
    term1 = outer(y, y)
    term2 = outer(Bs, Bs)
    return [[B[i][j] + (term1[i][j] / sy) - (term2[i][j] / yBs) for j in range(len(B))] for i in range(len(B))]

def solve_bfgs(f, grad_f, theta_0, max_iter=1000, tol=1e-6):
    d = len(theta_0)
    theta = theta_0[:]
    B = identity(d)
    
    for k in range(max_iter):
        g = grad_f(theta)
        if norm(g) < tol:
            break
        p = mat_vec(B, g)
        p = [-x for x in p]
        
        # Sources specify Armijo-Wolfe line search [S1], but do not provide implementation details.
        # A standard backtracking line search is used here as a [G] placeholder.
        alpha = 1.0
        while f(theta + [alpha * x for x in p]) > f(theta) + 1e-4 * alpha * dot(g, p):
            alpha *= 0.5
            
        theta_new = [theta[i] + alpha * p[i] for i in range(d)]
        s = [theta_new[i] - theta[i] for i in range(d)]
        g_new = grad_f(theta_new)
        y = [g_new[i] - g[i] for i in range(d)]
        
        B = bfgs_update(B, s, y)
        theta = theta_new
        
    return theta
```

## Subtopics

### Damped BFGS
A variant of BFGS that modifies the update step to handle cases where the Hessian is not positive semi-definite or gradients are noisy [S1]. It was tested in differentiable rendering contexts but did not improve convergence over vanilla BFGS for tasks beyond the smooth QUAD task [S1].

### Adaptive Finite Difference BFGS
A variant that uses adaptive finite differences to estimate gradients or Hessians within the BFGS framework [S1]. Like Damped BFGS, it was tested for noisy derivative estimation in rendering but failed to change convergence behavior for other tasks compared to standard methods [S1].

### SLSQP (Sequential Least Squares Quadratic Programming)
An optimization algorithm used to solve configuration selection objectives by approximating gradients and Hessians via least squares [S2]. It generates a search direction from these approximations and solves a system of linear equations to update variables [S2]. It is prone to producing unreasonable resource allocation on complex scenes due to poor initial assumption values and inherent approximation errors in the least-squares estimation [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Convergence:** Converges on smooth tasks like the QUAD task, but fails on noisy derivative estimations typical in rendering [S1].
- **Limitations:** Highly sensitive to noise in gradient estimation; vanilla variants fail on noisy rendering derivatives without damping or specialized line searches [S1].
- **When to use:** Smooth optimization problems where second-order information provides acceleration [G].
- **When not to use:** Noisy derivative estimation (e.g., stochastic rendering gradients) without damping or specialized line searches [S1].

## Background
Quasi-Newton methods, specifically the BFGS family, are well-established optimization algorithms [S1]. They were tested in the context of differentiable rendering to leverage second-order information while avoiding the explicit computation of the Hessian [S1]. The family includes vanilla BFGS, BFGS with Armijo-Wolfe line search, and LBFGS [S1]. Variants like Damped BFGS and Adaptive Finite Difference BFGS were also explored in this context [S1].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
[S2] NeRFlex: Resource-aware Real-time High-quality Rendering of Complex Scenes on Mobile Devices, https://doi.org/10.1109/icdcs63083.2025.00067, fetched 2026-10-29
