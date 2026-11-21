# Quasi-Newton Methods (BFGS/LBFGS)
Also called: BFGS, BFGS/LBFGS Optimization, LBFGS

## Overview
Quasi-Newton methods utilize second-order information for optimization by approximating the Hessian matrix using zero- or first-order information [S1]. Unlike [Newton's Method](6442-newton-s-method.md), which requires explicit computation of the Hessian, these methods build an approximation iteratively. In the context of differentiable rendering, they are distinguished by their sensitivity to noise; while they can converge on smooth tasks, they often fail on noisy derivative estimations common in rendering without specific damping or line-search modifications [S1].

## How it works
**Inputs and Outputs:**
- Input: Objective function $f(\theta)$, initial parameter vector $\theta_0$, initial Hessian approximation $H_0$ (or inverse $B_0$), and step size parameters.
- Output: Optimized parameter vector $\theta^*$.

**Data Structures:**
- $\theta$: Parameter vector of size $d$ (often large, e.g., neural network weights) [S1].
- $H$ or $B$: $d \times d$ matrix approximating the Hessian or its inverse. Storing this explicitly is $O(d^2)$, which is prohibitive for large $d$, leading to the use of Limited Memory variants [S1].

**Steps:**
1. **Initialization:** Set $\theta_0$, $H_0 = I$ (identity matrix), and iteration counter $k=0$.
2. **Gradient Evaluation:** Compute the gradient $g_k = \nabla f(\theta_k)$. In rendering, this may be sampled (stochastic) or analytic [S1].
3. **Search Direction:** Compute search direction $p_k$.
   - For BFGS (Inverse Hessian Approximation $B_k$): $p_k = -B_k g_k$.
   - For BFGS (Hessian Approximation $H_k$): Solve $H_k p_k = -g_k$.
4. **Line Search:** Determine step size $\alpha_k$ satisfying Wolfe conditions (e.g., Armijo-Wolfe) [S1].
   - Update $\theta_{k+1} = \theta_k + \alpha_k p_k$.
5. **Curvature Update:** Compute difference vectors:
   - $s_k = \theta_{k+1} - \theta_k$
   - $y_k = g_{k+1} - g_k$
6. **Hessian Update:** Update the approximation $B_{k+1}$ (or $H_{k+1}$) using the BFGS rank-2 update formula [S1]:
   $$ B_{k+1} = B_k + \frac{y_k y_k^T}{y_k^T s_k} - \frac{B_k s_k s_k^T B_k}{s_k^T B_k s_k} $$
   *Note: If $y_k^T s_k \leq 0$, the update is skipped to maintain positive definiteness [G].*
7. **Iteration:** Increment $k$. If convergence criteria are not met, return to Step 2.

**Parameters:**
- Line search tolerance: Standard Wolfe conditions [S1].
- Damping: Used in "Damped BFGS" to handle non-positive semi-definite (non-PSD) Hessians or noisy gradients [S1].

**Edge Cases and Pitfalls:**
- **Noisy Derivatives:** Vanilla BFGS, BFGS with Armijo-Wolfe line search, and LBFGS only converge for smooth tasks (e.g., QUAD task) but fail on noisy derivative estimations typical in rendering [S1].
- **Non-PSD Hessians:** The method assumes the Hessian is positive semi-definite. If the estimated Hessian is not PSD, convergence may fail unless damping is applied [S1].
- **Memory:** Standard BFGS requires $O(d^2)$ storage. For large-scale problems (e.g., neural networks), this is impractical [S1].

**Pseudocode (BFGS with Inverse Hessian Approximation):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bfgs_update(B, s, y):
    """
    Update inverse Hessian approximation B using BFGS formula.
    B: d x d matrix
    s: d x 1 vector (step)
    y: d x 1 vector (gradient difference)
    """
    sy = dot(s, y)
    if sy <= 1e-10: # Guard against non-positive curvature
        return B
    
    Bs = mat_vec(B, s)
    yBs = dot(y, Bs)
    
    # BFGS Rank-2 Update
    term1 = outer(y, y) / sy
    term2 = outer(Bs, Bs) / yBs
    
    return B + term1 - term2

def solve_bfgs(f, grad_f, theta_0, max_iter=1000, tol=1e-6):
    d = len(theta_0)
    theta = theta_0.copy()
    B = identity(d) # Initial inverse Hessian approximation
    
    for k in range(max_iter):
        g = grad_f(theta)
        
        # Check convergence
        if norm(g) < tol:
            break
            
        # Search direction
        p = -mat_vec(B, g)
        
        # Line search (simplified backtracking for brevity, 
        # sources specify Armijo-Wolfe [S1])
        alpha = line_search_armijo_wolfe(f, theta, p, g)
        
        # Update parameters
        theta_new = theta + alpha * p
        
        # Update Hessian approximation
        s = theta_new - theta
        g_new = grad_f(theta_new)
        y = g_new - g
        
        B = bfgs_update(B, s, y)
        theta = theta_new
        
    return theta
```

## Subtopics

### Damped BFGS
A variant of BFGS that modifies the update step to handle cases where the Hessian is not positive semi-definite or gradients are noisy. It was tested in differentiable rendering contexts but did not improve convergence over vanilla BFGS for tasks beyond the smooth QUAD task [S1].

### Adaptive Finite Difference BFGS
A variant that uses adaptive finite differences to estimate gradients or Hessians within the BFGS framework. Like Damped BFGS, it was tested for noisy derivative estimation in rendering but failed to change convergence behavior for other tasks compared to standard methods [S1].

### SLSQP (Sequential Least Squares Quadratic Programming)
An optimization algorithm used to solve configuration selection objectives by approximating gradients and Hessians via least squares [S2]. It generates a search direction from these approximations and solves a system of linear equations to update variables [S2]. It is prone to producing unreasonable resource allocation on complex scenes due to poor initial assumption values and inherent approximation errors in the least-squares estimation [S2].

## Operating characteristics
- **Time Complexity:** Each iteration requires $O(d^2)$ operations for the matrix-vector product and update in standard BFGS. LBFGS reduces this to $O(m \cdot d)$ where $m$ is the memory limit [S1].
- **Memory Complexity:** Standard BFGS is $O(d^2)$, which is prohibitive for large neural networks [S1]. LBFGS is $O(m \cdot d)$.
- **Convergence:** Superlinear convergence for smooth, convex problems [S1].
- **Limitations:** Highly sensitive to noise in gradient estimation. Vanilla variants fail on noisy rendering derivatives [S1]. Requires careful handling of non-PSD Hessians [S1].
- **When to use:** Smooth optimization problems where second-order information provides significant acceleration and memory constraints allow $O(d^2)$ or limited memory storage.
- **When not to use:** Noisy derivative estimation (e.g., stochastic rendering gradients) without damping or specialized line searches [S1].

## Background
Quasi-Newton methods, specifically the BFGS family, are well-established optimization algorithms [S1]. They were tested in the context of differentiable rendering to leverage second-order information while avoiding the explicit computation of the Hessian [S1]. The family includes vanilla BFGS, BFGS with Armijo-Wolfe line search, and LBFGS [S1]. Variants like Damped BFGS and Adaptive Finite Difference BFGS were also explored in this context [S1].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
[S2] NeRFlex: Resource-aware Real-time High-quality Rendering of Complex Scenes on Mobile Devices, https://doi.org/10.1109/icdcs63083.2025.00067, fetched 2026-10-29
