# Newton's Method

Also called: Second-order Taylor expansion optimizer, Newton-Raphson optimization (in scalar contexts), Newton direction solver

## Overview
Newton's method is a second-order optimization algorithm that approximates the objective function $f$ using a second-order Taylor expansion to determine an update step that targets the stationary point where the gradient is zero [S1]. It sets itself apart from first-order solvers like Gradient Descent by incorporating local curvature information via the Hessian matrix, allowing for faster convergence near optima, though at the cost of significant computational overhead for high-dimensional problems [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current parameter vector $\theta_t \in \mathbb{R}^n$, objective function $f: \mathbb{R}^n \to \mathbb{R}$, step size scalar $\gamma$ (or trust region radius $\Delta$).
*   **Output:** Updated parameter vector $\theta_{t+1}$.

**Data Structures:**
*   $\theta_t$: Vector of size $n$.
*   $g(\theta_t)$: Gradient vector $\nabla f(\theta_t)$ of size $n$.
*   $H(\theta_t)$: Hessian matrix $\nabla^2 f(\theta_t)$ of size $n \times n$.

**Formulas:**
The second-order Taylor expansion of $f$ around $\theta_t$ is:
$$ f(\theta) \approx f(\theta_t) + g(\theta_t)^T (\theta - \theta_t) + \frac{1}{2} (\theta - \theta_t)^T H(\theta_t) (\theta - \theta_t) $$
where $g(\theta_t) = \nabla f(\theta_t)$ and $H(\theta_t) = \nabla^2 f(\theta_t)$ [S1].

To find the optimum, we set the derivative of this approximation with respect to $\theta$ to zero:
$$ \frac{d}{d\theta} f(\theta) \approx g(\theta_t) + H(\theta_t)(\theta - \theta_t) = 0 $$

Solving for $\theta$ yields the Newton direction $v$:
$$ v = -H^{-1}(\theta_t) g(\theta_t) $$

The update rule is:
$$ \theta_{t+1} = \theta_t + \gamma v = \theta_t - \gamma H^{-1}(\theta_t) g(\theta_t) $$
where $\gamma$ is a step size scalar [S1].

**Parameters:**
*   $\gamma$: Step size. In standard Newton's method, $\gamma=1$ is implied by the derivation, but in practice, it may be adjusted or bounded by a trust region [S1].
*   $\Delta$: Trust region radius, used to truncate steps if the Hessian is not positive semi-definite [S1].

**Pitfalls and Edge Cases:**
1.  **Computational Cost:** Computing, storing, and inverting the full Hessian $H \in \mathbb{R}^{n \times n}$ is $O(n^3)$ for inversion and $O(n^2)$ for storage, which becomes a bottleneck in high-dimensional optimization problems typical in volume rendering [S1].
2.  **Negative Definiteness:** If $H(\theta_t)$ is not positive semi-definite (PSD), it may have negative eigenvalues. Applying $H^{-1}$ to the gradient can flip the direction, causing divergence away from the minimum [S1].
3.  **Mitigation:** To handle non-PSD Hessians, Hessian modification techniques are used to ensure positive semi-definiteness, combined with step truncation to a trust region: $\alpha = \min(\alpha, \Delta \|v\|^{-1})$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def newton_step(theta, f, grad_fn, hess_fn, gamma=1.0, delta=None):
    """
    Performs one step of Newton's Method.
    
    Args:
        theta: Current parameter vector (numpy array).
        f: Objective function.
        grad_fn: Function returning gradient g(theta).
        hess_fn: Function returning Hessian matrix H(theta).
        gamma: Step size scalar.
        delta: Trust region radius. If None, no truncation is applied.
        
    Returns:
        theta_next: Updated parameter vector.
    """
    # 1. Compute gradient and Hessian
    g = grad_fn(theta)
    H = hess_fn(theta)
    
    # 2. Ensure Hessian is positive semi-definite (PSD)
    # Note: The specific modification algorithm (e.g., eigenvalue shifting) 
    # is not detailed in the source, but is required to prevent divergence.
    H = ensure_psd(H)
    
    # 3. Solve for Newton direction v = -H^-1 * g
    # In practice, this is solved via linear system solver H * v = -g
    # rather than explicit inversion for numerical stability.
    try:
        v = solve_linear_system(H, -g)
    except SingularMatrixError:
        # Fallback to gradient descent if Hessian is singular
        v = -g
        
    # 4. Apply step
    theta_next = theta + gamma * v
    
    # 5. Trust region truncation if delta is provided
    if delta is not None:
        norm_v = norm(v)
        if norm_v > 0:
            # Truncate step to stay within trust region
            # alpha = min(alpha, delta / ||v||)
            # The source defines alpha as the step length scalar.
            # Here, we scale the step v by gamma.
            # If the source implies gamma is the step length along v:
            max_gamma = delta / norm_v
            effective_gamma = min(gamma, max_gamma)
            theta_next = theta + effective_gamma * v
            
    return theta_next

def ensure_psd(H):
    """
    Modifies H to be positive semi-definite.
    Implementation detail [G]: Common approach involves eigenvalue decomposition 
    and clamping negative eigenvalues to zero or a small positive epsilon.
    """
    eigenvalues, eigenvectors = eig(H)
    eigenvalues = max(eigenvalues, 1e-6) # Clamp to ensure PSD
    return eigenvectors @ diag(eigenvalues) @ eigenvectors.T
```

## Subtopics

### Quasi-Newton Methods (BFGS/LBFGS)
Quasi-Newton methods like BFGS and LBFGS approximate second-order information using zero or first-order information, avoiding the explicit computation of the full Hessian [S1]. Vanilla BFGS, BFGS with Armijo-Wolfe line search, and LBFGS were tested but only converged for the QUAD task, likely due to the noisy nature of derivative estimation in rendering [S1]. Damped BFGS and adaptive finite difference BFGS were also tested but did not improve convergence for other tasks [S1].

### Positive-Definite Hessian Handling
Naive Newton fails if the Hessian is not positive semi-definite (PSD), as negative eigenvalues can flip the gradient vector and cause divergence [S1]. This is handled by ensuring the Hessian is always positive semi-definite using Hessian modification techniques and step truncation to a trust region [S1]. In a 2D Gaussian convolution example with a non-PSD Hessian, analytic Hessians were compared with estimates, showing a mean error within $1 \times 10^{-4}$ for 100 samples across the interval $(-3, 3)^2$ [S1]. With modification and trust regions, optimization finds the correct minimum for any starting point in that interval [S1].

### Newton Conjugate Gradient
Newton Conjugate Gradient upgrades Newton's method by solving for the Newton direction iteratively using conjugate directions and determining the step length scalar $\alpha$ [S1]. It solves the linear equation $H(\theta)v = -g(\theta)$ iteratively [S1].
**Steps:**
1.  **Initialization:** Set direction $v_0$ and residual $r_0$ to the negative gradient: $v_0 = r_0 = -g(\theta_t)$ [S1].
2.  **Step Length:** Compute $\alpha$ by setting the derivative of the Taylor expansion along $v$ to zero:
    $$ \frac{d}{d\alpha} f(\theta + \alpha v) \approx g(\theta)^T v + \alpha v^T H(\theta) v = 0 $$
    $$ \alpha = - \frac{g(\theta)^T v}{v^T H(\theta) v} $$
    [S1].
3.  **Update Point:** $\theta_{t+1} = \theta_t + \alpha v_t$ [S1].
4.  **Update Residual:** $r_{t+1} = r_t - \alpha H(\theta_t) v_t$ [S1].
5.  **New Direction:** Compute $\beta$ using the Fletcher-Reeves formula:
    $$ \beta = \frac{r_{t+1}^T r_{t+1}}{r_t^T r_t} $$
    and update the direction:
    $$ v_{t+1} = r_{t+1} + \beta v_t $$
    [S1].

## Operating characteristics
*   **Time Complexity:** Computing the Hessian is $O(n^2)$ or higher depending on the method (analytic vs. finite difference). Inverting the Hessian is $O(n^3)$ [S1].
*   **Memory Complexity:** Storing the full Hessian requires $O(n^2)$ memory [S1].
*   **Strengths:** Second-order methods outperform first-order ones in terms of convergence speed and robustness to sampling noise when the Hessian is well-behaved [S1].
*   **Limitations:** The requirement to compute, store, and invert the full Hessian makes it a bottleneck in higher dimensions, such as those found in neural network optimization [S1]. It is sensitive to non-PSD Hessians, requiring modification techniques [S1].
*   **When to use:** Suitable for problems with moderate dimensionality where curvature information is critical and available. Not suitable for very high-dimensional problems without quasi-Newton approximations [S1].

## Background
Newton's method is one of the most-used second-order optimizers, derived from the second-order Taylor expansion of the objective function [S1]. It has been applied to several optimization tasks in vision and graphics [S1]. The specific application to stochastic gradient estimation for higher-order differentiable rendering is detailed in the source [S1].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
