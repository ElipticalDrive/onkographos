# Newton's Method

Also called: Newton-Raphson method, Newton-Raphson method for inverse rays, Root Finding Based Algorithm

## Overview
Newton's Method is a second-order optimization algorithm derived from the second-order Taylor expansion of the objective function [S2]. It solves for the root of the gradient (where the derivative is zero) to find local minima or maxima, offering quadratic convergence for well-behaved functions [S1, S2]. Unlike [Gradient Descent Solver](3280-gradient-descent-solver.md), which uses only first-order gradient information and a fixed or adaptive step size, Newton's Method incorporates the Hessian matrix to account for local curvature, allowing it to take larger, more informed steps toward the optimum [S2]. In the context of inverse volume rendering, it is also applied as a root-finding algorithm to solve for ray intersections with implicit geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current parameter vector $\theta_t$, objective function $f(\theta)$, gradient function $g(\theta) = \nabla f(\theta)$, Hessian function $H(\theta) = \nabla^2 f(\theta)$ [S2].
*   **Output:** Updated parameter vector $\theta_{t+1}$ [S2].

**Data Structures:**
*   $\theta \in \mathbb{R}^n$: Parameter vector [S2].
*   $g \in \mathbb{R}^n$: Gradient vector [S2].
*   $H \in \mathbb{R}^{n \times n}$: Hessian matrix [S2].
*   $v \in \mathbb{R}^n$: Newton direction vector [S2].
*   $\alpha \in \mathbb{R}$: Step length scalar [S2].

**Numbered Steps:**
1.  **Compute Gradient and Hessian:** Evaluate the gradient $g(\theta_t)$ and the Hessian matrix $H(\theta_t)$ at the current point $\theta_t$ [S2].
2.  **Check Positive Semi-Definiteness (PSD):** Verify if $H(\theta_t)$ is positive semi-definite. If not, apply Hessian modification to ensure PSD, or use a trust region method [S2].
3.  **Compute Newton Direction:** Solve the linear system $H(\theta_t) v = -g(\theta_t)$ for $v$. The update direction is $v = -H^{-1}(\theta_t)g(\theta_t)$ [S2].
    *   *Note:* For root-finding applications (e.g., inverse rays), solve $J_F(x_n)(x_{n+1} - x_n) = -F(x_n)$ for the root approximation $x_{n+1}$, where $J_F$ is the Jacobian of the residual function $F$ [S1].
4.  **Determine Step Length:**
    *   In standard Newton's Method, a fixed step size $\gamma$ is often used, or $\alpha=1$ is assumed if the quadratic approximation is valid [S2].
    *   In Newton Conjugate Gradient, $\alpha$ is derived analytically by minimizing the quadratic approximation along $v$: $\alpha = - \frac{g(\theta_t)^T v}{v^T H(\theta_t) v}$ [S2].
5.  **Apply Trust Region (Optional but Recommended):** To prevent divergence if the Hessian is not well-conditioned, truncate the step: $\alpha = \min(\alpha, \Delta \|v\|^{-1})$, where $\Delta$ is the trust region radius [S2].
6.  **Update Parameters:** $\theta_{t+1} = \theta_t + \alpha v$ [S2].

**Formulas:**
*   **Second-Order Taylor Expansion:**
    $$f(\theta) \approx f(\theta_t) + g(\theta_t)^T (\theta - \theta_t) + \frac{1}{2} (\theta - \theta_t)^T H(\theta_t) (\theta - \theta_t)$$
    [S2]
*   **Optimality Condition (Root of Gradient):**
    $$\frac{d}{d\theta} f(\theta) = 0 \approx g(\theta_t) + H(\theta_t)(\theta - \theta_t)$$
    [S2]
*   **Newton Direction:**
    $$v = -H^{-1}(\theta_t)g(\theta_t)$$
    [S2]
*   **Update Rule:**
    $$\theta_{t+1} = \theta_t - \gamma H^{-1}(\theta_t)g(\theta_t)$$
    [S2]
*   **Newton Conjugate Gradient Step Length:**
    $$\alpha = - \frac{g(\theta)^T v}{v^T H(\theta) v}$$
    [S2]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def norm(v):
    return sum(x**2 for x in v)**0.5

def dot(a, b):
    return sum(x*y for x, y in zip(a, b))

def ensure_psd(H):
    # Shift diagonal until positive semi-definite [G]
    min_eig = np.linalg.eigvalsh(H).min()
    if min_eig < 0:
        H = H + (-min_eig + 1e-6) * np.eye(len(H))
    return H

def solve_linear_system(A, b):
    return np.linalg.solve(A, b)

def hess_vector_product(theta, v, grad, eps=1e-5):
    # Central difference approximation [S2]
    return (grad(theta + eps * v) - grad(theta - eps * v)) / (2 * eps)

def newton_method(theta, f, grad, hess, max_iter=100, tol=1e-6, trust_radius=None):
    for t in range(max_iter):
        g = grad(theta)
        H = hess(theta)
        if norm(g) < tol:
            break
        H = ensure_psd(H)
        v = solve_linear_system(H, -g)
        alpha = 1.0
        if trust_radius is not None:
            if norm(v) > 1e-12:
                alpha = min(alpha, trust_radius / norm(v))
            else:
                alpha = 0.0
        theta = theta + alpha * v
    return theta

def newton_conjugate_gradient(theta, f, grad, hess, max_iter=100, tol=1e-6):
    v = -grad(theta)
    r = v
    for t in range(max_iter):
        Hv = hess_vector_product(theta, v, grad)
        g = grad(theta)
        numerator = -dot(g, v)
        denominator = dot(v, Hv)
        if abs(denominator) < 1e-12:
            break
        alpha = numerator / denominator
        theta = theta + alpha * v
        r_new = r - alpha * Hv
        if norm(r_new) < tol:
            break
        beta = dot(r_new, r_new) / dot(r, r)
        v = r_new + beta * v
        r = r_new
    return theta
```

**Parameters:**
*   $\gamma$: Step size multiplier (often 1 in pure Newton, but used in damped variants) [S2].
*   $\Delta$: Trust region radius [S2].
*   $\epsilon$: Perturbation magnitude for central-difference Hessian-vector product approximation [S2].

**Edge Cases and Pitfalls:**
*   **Hessian Inversion Cost:** Computing, storing, and inverting the full Hessian $H \in \mathbb{R}^{n \times n}$ is $O(n^3)$ and requires $O(n^2)$ memory, becoming a bottleneck in high-dimensional problems [S2].
*   **Negative Definiteness:** If the Hessian is not positive semi-definite, the Newton direction may point towards a maximum or saddle point, causing divergence [S2]. This is handled by Hessian modification (ensuring PSD) and trust region step truncation [S2].
*   **Root Finding Failure:** In inverse ray tracing, naive Newton-Raphson can fail if the starting point is not close to the root or if the Jacobian is singular [S1].

## Subtopics

### [Quasi-Newton Methods (BFGS/LBFGS)](6456-quasi-newton-methods-bfgs-lbfgs.md)
Quasi-Newton methods utilize second-order information for optimization by approximating the Hessian matrix using zero- or first-order information. Unlike [Newton's Method](6442-newton-s-method.md), which requires explicit computation of the Hessian, these methods build an approximation iteratively. In the context of differentiable rendering, they are distinguished by their sensitivity to noise; while they can converge on smooth tasks, they often fail on noisy derivative estimations common in rendering without specific damping or line-search modifications.

### Positive-Definite Hessian Handling
Naive Newton fails if the Hessian is not positive semi-definite (PSD), as negative eigenvalues can flip the gradient vector and cause divergence [S2]. This is handled by ensuring positive semi-definiteness via Hessian modification and step truncation to a trust region: $\alpha = \min(\alpha, \Delta \|v\|^{-1})$ [S2]. For example, in a negated 2D Gaussian convolution, analytic Hessians can be computed and compared with estimates, with mean error within $1 \times 10^{-4}$ for 100 samples across the interval $(-3,3)^2$ [S2].

### Newton Conjugate Gradient
An upgrade to Newton's method that solves for the Newton direction iteratively using conjugate directions and determines the step length scalar $\alpha$ [S2]. It avoids explicit Hessian inversion by solving $H(\theta)v = -g(\theta)$ iteratively [S2]. The direction $v$ and residual $r$ are initialized to the negative gradient ($v_0 = r_0 = -g(\theta_t)$). The step length is derived from the Taylor expansion derivative set to zero: $\alpha = - \frac{g^T(\theta)v}{v^T H(\theta)v}$ [S2]. The residual is updated as $r_{t+1} = r_t - \alpha H(\theta_t) v_t$, and the new direction is computed using the Fletcher-Reeves formula: $\beta = \frac{r_{t+1}^T r_{t+1}}{r_t^T r_t}$ and $v_{t+1} = r_{t+1} + \beta v_t$ [S2].

## Operating characteristics
*   **Time Complexity:** $O(n^3)$ per iteration for standard Newton's Method due to Hessian inversion [S2]. Newton Conjugate Gradient reduces this to $O(n^2)$ per iteration if Hessian-vector products are $O(n)$, but requires multiple iterations to converge to the Newton direction [S2].
*   **Memory Complexity:** $O(n^2)$ for storing the Hessian matrix [S2].
*   **Convergence:** Quadratic convergence near the optimum for well-behaved functions [S1].
*   **Strengths:** Fast convergence near the solution; accounts for curvature [S2].
*   **Limitations:** High computational cost for large $n$; sensitive to non-PSD Hessians; requires smooth, differentiable objective functions [S2].
*   **When to Use:** Small to medium-scale optimization problems where the Hessian is affordable to compute and invert; when high precision is required near the optimum [S2].
*   **When Not to Use:** Large-scale problems where Hessian storage is prohibitive [S2]. The method can also fail in certain situations, such as poor starting points or singular Jacobians [S1].

## Background
Newton's Method is one of the most-used second-order optimizers, derived from the second-order Taylor expansion of the objective function [S2]. It has been applied to several optimization tasks in vision and graphics [S2]. The Newton-Raphson method is also a standard method for finding approximations of the roots of functions, such as in inverse ray tracing for volume rendering [S1]. Hessian-vector product approximations, such as central differences, have been discussed by Pearlmutter and Werbos to avoid explicit Hessian computation [S2].

## Sources
[S1] Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry, https://doi.org/10.1109/tvcg.2015.2430337, fetched 2026-10-28
[S2] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
