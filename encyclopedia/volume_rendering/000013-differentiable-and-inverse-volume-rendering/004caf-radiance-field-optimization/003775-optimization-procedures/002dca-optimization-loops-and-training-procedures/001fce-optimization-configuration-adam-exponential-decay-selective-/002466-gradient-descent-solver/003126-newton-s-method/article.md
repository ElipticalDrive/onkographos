# Newton's Method

Also called: Newton-Raphson method, Newton-Raphson method for inverse rays, Root Finding Based Algorithm

## Overview
Newton's Method is a second-order optimization algorithm derived from the second-order Taylor expansion of the objective function [S2]. It solves for the root of the gradient (where the derivative is zero) to find local minima or maxima, offering quadratic convergence for well-behaved functions [S1, S2]. Unlike [Gradient Descent Solver](3280-gradient-descent-solver.md), which uses only first-order gradient information and a fixed or adaptive step size, Newton's Method incorporates the Hessian matrix to account for local curvature, allowing it to take larger, more informed steps toward the optimum [S2]. In the context of inverse volume rendering, it is also applied as a root-finding algorithm to solve for ray intersections with implicit geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current parameter vector $\theta_t$, objective function $f(\theta)$, gradient function $g(\theta) = \nabla f(\theta)$, Hessian function $H(\theta) = \nabla^2 f(\theta)$.
*   **Output:** Updated parameter vector $\theta_{t+1}$.

**Data Structures:**
*   $\theta \in \mathbb{R}^n$: Parameter vector.
*   $g \in \mathbb{R}^n$: Gradient vector.
*   $H \in \mathbb{R}^{n \times n}$: Hessian matrix.
*   $v \in \mathbb{R}^n$: Newton direction vector.
*   $\alpha \in \mathbb{R}$: Step length scalar.

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
def newton_method(theta, f, grad, hess, max_iter=100, tol=1e-6, trust_radius=None):
    """
    Standard Newton's Method with optional trust region.
    """
    for t in range(max_iter):
        # 1. Compute gradient and Hessian
        g = grad(theta)
        H = hess(theta)
        
        # 2. Check convergence
        if norm(g) < tol:
            break
            
        # 3. Ensure Hessian is Positive Semi-Definite (PSD)
        # [G] In practice, this involves eigenvalue decomposition or 
        # adding a multiple of the identity matrix (Levenberg-Marquardt style)
        # to the diagonal until eigenvalues are non-negative.
        H = ensure_psd(H)
        
        # 4. Compute Newton direction v = -H^-1 * g
        # [G] Solve linear system H @ v = -g
        v = solve_linear_system(H, -g)
        
        # 5. Determine step length alpha
        # Standard Newton uses alpha = 1 (or gamma)
        alpha = 1.0
        
        # 6. Apply Trust Region if specified
        if trust_radius is not None:
            # alpha = min(alpha, trust_radius / norm(v))
            # [G] Handle case where norm(v) is 0
            if norm(v) > 1e-12:
                alpha = min(alpha, trust_radius / norm(v))
            else:
                alpha = 0.0
                
        # 7. Update theta
        theta = theta + alpha * v
        
    return theta

def newton_conjugate_gradient(theta, f, grad, hess, max_iter=100, tol=1e-6):
    """
    Newton Conjugate Gradient: Solves for Newton direction iteratively.
    """
    # Initialization
    v = -grad(theta)
    r = v # r_0 = v_0 = -g(theta_t)
    
    for t in range(max_iter):
        # Compute Hessian-vector product H*v
        Hv = hess_vector_product(theta, v)
        
        # Compute step length alpha
        # alpha = - (g^T v) / (v^T H v)
        g = grad(theta)
        numerator = -dot(g, v)
        denominator = dot(v, Hv)
        
        if abs(denominator) < 1e-12:
            break
            
        alpha = numerator / denominator
        
        # Update point
        theta = theta + alpha * v
        
        # Update residual
        r_new = r - alpha * Hv
        
        # Check convergence
        if norm(r_new) < tol:
            break
            
        # Compute beta (Fletcher-Reeves)
        beta = dot(r_new, r_new) / dot(r, r)
        
        # Update direction
        v = r_new + beta * v
        r = r_new
        
    return theta
```

**Parameters:**
*   $\gamma$: Step size multiplier (often 1 in pure Newton, but used in damped variants) [S2].
*   $\Delta$: Trust region radius [S2].
*   $\epsilon$: Tolerance for convergence [S2].

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
*   **Convergence:** Quadratic convergence near the optimum for well-behaved functions [S1]. Linear convergence for Newton Conjugate Gradient [S2].
*   **Strengths:** Fast convergence near the solution; accounts for curvature [S2].
*   **Limitations:** High computational cost for large $n$; sensitive to non-PSD Hessians; requires smooth, differentiable objective functions [S2].
*   **When to Use:** Small to medium-scale optimization problems where the Hessian is affordable to compute and invert; when high precision is required near the optimum [S2].
*   **When Not to Use:** Large-scale problems (e.g., deep neural networks with millions of parameters) where Hessian storage is prohibitive; noisy gradient estimates where second-order information is unreliable [S1, S2].

## Background
Newton's Method is one of the most-used second-order optimizers, derived from the second-order Taylor expansion of the objective function [S2]. It has been applied to several optimization tasks in vision and graphics [S2]. The Newton-Raphson method is also a standard method for finding approximations of the roots of functions, such as in inverse ray tracing for volume rendering [S1]. Hessian-vector product approximations, such as central differences, have been discussed by Pearlmutter and Werbos to avoid explicit Hessian computation [S2].

## Sources
[S1] Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry, https://doi.org/10.1109/tvcg.2015.2430337, fetched 2026-10-28
[S2] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
