# Gradient Descent Solver
Also called: Adam Optimizer, Adam Optimizer Solver, L1/L2 Norm Solver

## Overview
The Gradient Descent Solver is a matrix-free optimization method used to update voxel grid parameters or transfer function coefficients in differentiable volume rendering [S2, unofficial]. It solves the problem of finding optimal scalar field values by minimizing a loss function via iterative gradient updates, offering stable behavior for ill-conditioned systems [S1, unofficial]. It sets itself apart from direct linear solvers (like CGLS) by avoiding the explicit formation of normal equations ($A^T A$), making it suitable for large-scale or rank-deficient problems where memory or numerical stability is a concern [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Current parameter vector $\mathbf{x}$ (e.g., voxel densities or transfer function weights), gradient vector $\mathbf{g}$ computed via backpropagation through the differentiable renderer, and solver hyperparameters.
- **Output:** Updated parameter vector $\mathbf{x}'$.

**Data Structures:**
- **Parameters:** A dense array of floating-point values representing the voxel grid or TF coefficients.
- **Moments (Adam only):** Two auxiliary arrays of the same shape as $\mathbf{x}$:
  - $\mathbf{m}$: First moment estimate (mean of gradients).
  - $\mathbf{v}$: Second moment estimate (uncentered variance of gradients).
- **State:** Iteration counter $t$ (initially 1).

**Loss Functions:**
The solver supports two norm-based objectives [S1, unofficial]:
1.  **$L_2$-norm (Least Squares):** Penalizes larger residuals more strongly due to the quadratic term.
2.  **$L_1$-norm (Least Absolute Deviations):** Tolerates larger outliers in favor of minimizing many lower deviations.

**Algorithm Steps:**
The implementation is matrix-free, accumulating gradients via atomic additions to handle parallel ray contributions [S1, unofficial].

1.  **Compute Gradient:** Calculate the gradient $\mathbf{g}$ of the loss function with respect to the parameters $\mathbf{x}$ using automatic differentiation through the volume rendering pipeline.
2.  **Update Moments (if using Adam):**
    -   Update biased first moment estimate:
        $$ \mathbf{m}_t = \beta_1 \mathbf{m}_{t-1} + (1 - \beta_1) \mathbf{g}_t $$
    -   Update biased second raw moment estimate:
        $$ \mathbf{v}_t = \beta_2 \mathbf{v}_{t-1} + (1 - \beta_2) \mathbf{g}_t^2 $$
    -   Apply bias correction:
        $$ \hat{\mathbf{m}}_t = \frac{\mathbf{m}_t}{1 - \beta_1^t} $$
        $$ \hat{\mathbf{v}}_t = \frac{\mathbf{v}_t}{1 - \beta_2^t} $$
    -   *Note:* If using constant update rate $\alpha$, skip this step and set effective step size to $\alpha$.
3.  **Compute Update Direction:**
    -   For Adam: $\mathbf{d}_t = \frac{\hat{\mathbf{m}}_t}{\sqrt{\hat{\mathbf{v}}_t} + \epsilon}$
    -   For Constant Rate: $\mathbf{d}_t = \mathbf{g}_t$
4.  **Update Parameters:**
    $$ \mathbf{x}_{t} = \mathbf{x}_{t-1} - \alpha \cdot \mathbf{d}_t $$
    Where $\alpha$ is the learning rate.

**Parameters:**
-   **$\alpha$ (Learning Rate):** Step size. Typical values range from $10^{-4}$ to $10^{-2}$ depending on the problem scale [S1, unofficial].
-   **$\beta_1$ (Adam):** Exponential decay rate for the first moment estimates. Default: 0.9.
-   **$\beta_2$ (Adam):** Exponential decay rate for the second moment estimates. Default: 0.999.
-   **$\epsilon$ (Adam):** Small constant for numerical stability. Default: $10^{-8}$.

**Edge Cases and Pitfalls:**
-   **Ill-conditioning:** While more stable than direct solvers for ill-conditioned systems, poor choice of $\alpha$ can still lead to divergence [S1, unofficial].
-   **Atomic Additions:** In parallel implementations, gradient accumulation must use atomic operations to prevent race conditions when multiple rays contribute to the same voxel [S1, unofficial].
-   **$L_1$ vs $L_2$:** The choice of norm affects robustness to outliers. $L_2$ is sensitive to large errors, while $L_1$ is more robust but may converge slower near the optimum [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_descent_step(x, g, alpha, beta1=0.9, beta2=0.999, eps=1e-8, use_adam=True):
    """
    Performs one step of gradient descent or Adam optimization.
    
    Args:
        x: Current parameter array.
        g: Gradient array (computed via backprop).
        alpha: Learning rate.
        beta1: Adam first moment decay.
        beta2: Adam second moment decay.
        eps: Adam numerical stability constant.
        use_adam: Boolean flag to switch between constant rate and Adam.
        
    Returns:
        Updated parameter array x_new.
    """
    if use_adam:
        # Initialize moments if not present (handled by solver state)
        # m and v are assumed to be maintained in solver state
        m = solver_state.m
        v = solver_state.v
        t = solver_state.t
        
        # Update biased first moment
        m = beta1 * m + (1 - beta1) * g
        # Update biased second raw moment
        v = beta2 * v + (1 - beta2) * (g * g)
        
        # Bias correction
        m_hat = m / (1 - beta1 ** t)
        v_hat = v / (1 - beta2 ** t)
        
        # Compute update direction
        update = m_hat / (sqrt(v_hat) + eps)
        
        # Update state
        solver_state.m = m
        solver_state.v = v
        solver_state.t = t + 1
    else:
        # Constant update rate
        update = g
        
    # Apply update
    x_new = x - alpha * update
    return x_new
```

## Subtopics

### [CGLS Solver (Conjugate Gradient Least Squares)](3285-cgls-solver-conjugate-gradient-least-squares.md)
A sparse linear least squares solver that avoids ill-conditioning of normal equations, providing numerically stable results for large or rank-deficient problems. Choose when direct linear system solving is preferred over iterative gradient steps.

### [Partially Learned Gradient Descent](3771-partially-learned-gradient-descent.md)
Unrolls optimization steps into a neural network, combining fixed physics-based gradients with learnable updates. Choose when data-driven adaptation of step size and momentum is needed while retaining physical interpretability.

### [SGD with Positive t-Distribution](5809-sgd-with-positive-t-distribution.md)
Replaces Gaussian noise assumptions with a positive Student's t-distribution to model gradient uncertainty. Choose as a baseline to isolate the impact of heavy-tailed noise modeling.

### [Newton's Method](6442-newton-s-method.md)
A second-order algorithm using Hessian curvature information for faster convergence near optima. Choose when computational overhead is acceptable and faster local convergence is critical.

### [Inexact Line Search Optimization](7741-inexact-line-search-optimization.md)
Adapts step size dynamically to ensure sufficient reduction in the objective function. Choose when standard gradient descent converges too slowly due to fixed step sizes.

## Operating characteristics
-   **Time Complexity:** $O(N)$ per iteration, where $N$ is the number of parameters. Each iteration requires a forward and backward pass through the renderer.
-   **Memory Complexity:** $O(N)$ for parameters. If using Adam, an additional $2N$ for moment estimates. Matrix-free implementation avoids $O(N^2)$ storage for Hessian or normal equations.
-   **Measured Performance:** Stable for ill-conditioned systems, comparable to CGLS in stability but potentially slower in convergence speed for well-conditioned problems [S1, unofficial].
-   **Strengths:** Robust to ill-conditioning, low memory footprint (matrix-free), supports both $L_1$ and $L_2$ norms.
-   **Limitations:** Slower convergence than second-order methods for well-conditioned problems; sensitive to learning rate selection.
-   **When to Use:** Large-scale volume rendering, ill-conditioned inverse problems, or when memory constraints prevent forming normal equations.
-   **When Not to Use:** When fast convergence is critical and the problem is well-conditioned (consider Newton's method or CGLS).

## Background
The Adam optimizer, a variant of gradient descent, was introduced by Kingma and Ba in 2015 [S2, unofficial]. It has been widely adopted in differentiable volume rendering for optimizing voxel grid parameters and transfer functions due to its adaptive learning rate capabilities [S2, unofficial]. The specific implementation described here supports both constant update rates and Adam, and is noted for its stability in ill-conditioned systems [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
[S2] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
