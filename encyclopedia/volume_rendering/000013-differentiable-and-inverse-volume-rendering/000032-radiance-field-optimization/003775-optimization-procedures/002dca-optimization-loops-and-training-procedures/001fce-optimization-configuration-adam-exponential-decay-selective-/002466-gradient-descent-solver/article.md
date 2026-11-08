# Gradient Descent Solver

Also called: Adam Optimizer, Adam Optimizer Solver, L1/L2 Norm Solver

## Overview
The Gradient Descent Solver is a matrix-free optimization method used to update voxel grid parameters in differentiable volume rendering, specifically for computing optimal transfer functions [S1, unofficial][S2, unofficial]. It solves the problem of ill-conditioned linear systems by providing stable behavior for large or nearly rank-deficient problems, serving as a robust alternative to direct solvers like CGLS [S1, unofficial]. It distinguishes itself from its siblings by supporting both constant learning rates and the Adam optimizer, while offering a choice between $l_2$-norm (least squares) and $l_1$-norm (least absolute deviations) loss functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of voxel parameters (e.g., transfer function values) to be optimized, gradients computed via backpropagation through the differentiable renderer, and optimization hyperparameters (learning rate $\alpha$, Adam moments $m, v$, loss norm type).
*   **Output:** Updated voxel parameters.

**Data Structures:**
The solver operates in a matrix-free manner. Gradients are accumulated directly into the parameter storage using atomic additions to handle concurrent updates, avoiding the explicit construction of the Hessian or normal equations [S1, unofficial].

**Steps:**
1.  **Gradient Accumulation:** For each training sample or batch, compute the gradient of the loss function with respect to the voxel parameters. Accumulate these gradients into a global gradient buffer using atomic additions to ensure thread safety in parallel implementations [S1, unofficial].
2.  **Loss Norm Selection:** Determine the loss function type:
    *   **$l_2$-norm:** Minimizes the sum of squared residuals. Penalizes larger residuals more strongly due to the quadratic term [S1, unofficial].
    *   **$l_1$-norm:** Minimizes the sum of absolute residuals. Tolerates larger outliers in favor of minimizing many lower deviations [S1, unofficial].
3.  **Update Rule Application:** Apply the chosen optimizer (Constant $\alpha$ or Adam) to update the parameters based on the accumulated gradients.

**Formulas:**
For the $l_2$-norm loss $L_2 = \frac{1}{2} \sum (y_i - \hat{y}_i)^2$, the gradient is proportional to the residual.
For the $l_1$-norm loss $L_1 = \sum |y_i - \hat{y}_i|$, the gradient is proportional to the sign of the residual.

**Adam Optimizer Parameters:**
If Adam is selected, maintain first and second moment estimates $m_t$ and $v_t$ for each parameter:
$$ m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t $$
$$ v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2 $$
$$ \hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} $$
$$ \theta_{t+1} = \theta_t - \alpha \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} $$
Where $g_t$ is the gradient, $\alpha$ is the learning rate, $\beta_1, \beta_2$ are decay rates (typically 0.9, 0.999), and $\epsilon$ is a small constant for numerical stability (typically $10^{-8}$) [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_descent_solver(parameters, gradients, optimizer_type, alpha, beta1=0.9, beta2=0.999, epsilon=1e-8):
    """
    Updates parameters using accumulated gradients.
    parameters: Tensor of voxel values.
    gradients: Tensor of accumulated gradients (via atomic additions).
    optimizer_type: 'constant' or 'adam'.
    alpha: Learning rate.
    """
    if optimizer_type == 'constant':
        # Simple gradient descent with constant step size
        # Note: The specific norm (l1 vs l2) affects the gradient values 
        # computed before accumulation, not the update step here.
        parameters = parameters - alpha * gradients
    elif optimizer_type == 'adam':
        # Adam update step
        # Assumes m and v are maintained state variables for each parameter
        # This is a single step update; m and v must be initialized and updated 
        # in the training loop prior to this call or within it.
        m = parameters.m # First moment estimate
        v = parameters.v # Second moment estimate
        
        # Update moments
        m = beta1 * m + (1 - beta1) * gradients
        v = beta2 * v + (1 - beta2) * gradients**2
        
        # Bias correction
        t = parameters.t # Current timestep
        m_hat = m / (1 - beta1**t)
        v_hat = v / (1 - beta2**t)
        
        # Update parameters
        parameters = parameters - alpha * m_hat / (sqrt(v_hat) + epsilon)
        
        # Update state
        parameters.m = m
        parameters.v = v
        parameters.t = t + 1
        
    return parameters
```

**Edge Cases and Pitfalls:**
*   **Ill-conditioning:** While stable, gradient descent may converge slowly for ill-conditioned systems compared to second-order methods, though it avoids the numerical instability of computing $A^T A$ [S1, unofficial].
*   **Atomic Operations:** Accumulating gradients via atomic additions requires careful synchronization in parallel implementations to avoid race conditions or performance bottlenecks [S1, unofficial].
*   **Loss Choice:** The choice between $l_1$ and $l_2$ norms can impact final results; $l_2$ is sensitive to outliers, while $l_1$ is more robust but may yield less crisp transfer functions in some contexts [S1, unofficial].

## Subtopics

### [CGLS Solver (Conjugate Gradient Least Squares)](3285-cgls-solver-conjugate-gradient-least-squares.md)
A sparse linear least squares solver that avoids ill-conditioning of normal equations, providing numerically stable results for large or rank-deficient problems.

### [Partially Learned Gradient Descent](3771-partially-learned-gradient-descent.md)
An iterative reconstruction method that unrolls optimization steps into a neural network, combining fixed physics-based gradient steps with learnable convolutional updates.

### [SGD with Positive t-Distribution](5809-sgd-with-positive-t-distribution.md)
Replaces Gaussian noise assumptions with a positive Student's t-distribution to model gradient uncertainty, serving as a baseline for heavy-tailed noise modeling.

### [Newton's Method](6442-newton-s-method.md)
A second-order optimization algorithm using the Hessian matrix for faster convergence near optima, at the cost of higher computational overhead.

### [Inexact Line Search Optimization](7741-inexact-line-search-optimization.md)
Adapts the step size in gradient descent to ensure sufficient reduction in the objective function without the cost of exact line search.

## Operating characteristics
*   **Time Complexity:** Linear in the number of parameters per iteration, as it avoids matrix inversion or factorization. Convergence speed depends on the condition number of the problem; it is generally slower than CGLS for well-conditioned systems but more stable for ill-conditioned ones [S1, unofficial].
*   **Memory Complexity:** Low, as it is matrix-free. It only stores parameters, gradients, and optimizer state (for Adam), avoiding the storage of large matrices like $A^T A$ [S1, unofficial].
*   **Strengths:** Stable for ill-conditioned systems; supports both constant and adaptive (Adam) learning rates; flexible loss functions ($l_1, l_2$); matrix-free implementation reduces memory overhead [S1, unofficial].
*   **Limitations:** Slower convergence for well-conditioned problems compared to direct solvers; sensitive to learning rate selection (unless using Adam); atomic operations may introduce synchronization overhead [S1, unofficial].
*   **When to Use:** When dealing with large, ill-conditioned, or rank-deficient systems where direct solvers are unstable or too memory-intensive; when adaptive learning rates (Adam) are beneficial [S1, unofficial].
*   **When Not to Use:** When fast convergence is critical for well-conditioned problems; when second-order information (Hessian) is available and computationally feasible [S1, unofficial].

## Background
The Gradient Descent Solver, particularly in its Adam variant, was introduced by Kingma and Ba in 2015 [S2, unofficial]. It has been applied in differentiable volume rendering contexts, such as DiffVox, to optimize voxel grid parameters for transfer function computation [S2, unofficial]. It is used in comparative volume rendering studies to evaluate the stability and quality of optimized transfer functions against reference methods [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
[S2] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
