# Gradient Descent Solver

Also called: Adam Optimizer Solver, L1/L2 Norm Solver

## Overview
The Gradient Descent Solver is a matrix-free iterative method for optimizing transfer functions in volume rendering, designed to handle ill-conditioned or rank-deficient systems more stably than direct least-squares approaches [S1, unofficial]. It distinguishes itself from sibling solvers like CGLS by avoiding the explicit formation of normal equations ($A^T A$), thereby reducing memory overhead and numerical instability for large datasets, while supporting both constant step sizes and adaptive optimizers like Adam [S1, unofficial].

## How it works
**Inputs:**
- Residual field or error signal derived from the difference between rendered and target values.
- Optimization hyperparameters: learning rate $\alpha$ (for constant update) or Adam parameters ($\beta_1, \beta_2, \epsilon$).
- Norm type: $L_2$ (least squares) or $L_1$ (least absolute deviations) [S1, unofficial].

**Outputs:**
- Optimized transfer function parameters or voxel values.

**Data Structures:**
- **Gradient Accumulator:** A dense array matching the dimensionality of the transfer function or voxel grid. Implemented in a matrix-free manner, gradients are accumulated directly via atomic additions to avoid storing the full Jacobian or Hessian matrices [S1, unofficial].
- **Optimizer State (Adam only):** First moment estimate ($m$) and second moment estimate ($v$) arrays, same size as the parameters [KB15].

**Steps:**
1. **Compute Residuals:** Calculate the residual $r_i$ for each sample or voxel $i$ based on the current transfer function evaluation.
2. **Compute Gradients:**
   - For $L_2$-norm: The gradient contribution for sample $i$ is proportional to $2 \cdot r_i \cdot \frac{\partial r_i}{\partial \theta}$, where $\theta$ represents the transfer function parameters.
   - For $L_1$-norm: The gradient contribution is proportional to $\text{sign}(r_i) \cdot \frac{\partial r_i}{\partial \theta}$. This norm tolerates larger outliers better than $L_2$ by penalizing deviations linearly rather than quadratically [S1, unofficial].
3. **Accumulate Gradients:** Add the computed gradients to the global gradient accumulator using atomic operations to ensure thread safety in parallel implementations [S1, unofficial].
4. **Update Parameters:**
   - **Constant Rate:** $\theta_{new} = \theta_{old} - \alpha \cdot \nabla L(\theta)$.
   - **Adam Optimizer:** Update $\theta$ using the adaptive learning rates derived from the first and second moments of the gradients, as defined by Kingma and Ba [KB15].
5. **Iterate:** Repeat steps 1–4 until convergence or a maximum number of iterations is reached.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_descent_solver(residuals, gradients_per_sample, params, optimizer_type='adam', alpha=0.01, beta1=0.9, beta2=0.999, epsilon=1e-8):
    """
    Matrix-free gradient descent solver for transfer function optimization.
    
    Args:
        residuals: Array of residuals r_i.
        gradients_per_sample: Array of partial derivatives d(r_i)/d(theta) for each sample.
        params: Current transfer function parameters theta.
        optimizer_type: 'constant' or 'adam'.
        alpha: Learning rate for constant optimizer.
        beta1, beta2, epsilon: Adam hyperparameters.
        
    Returns:
        Updated parameters theta.
    """
    # Initialize optimizer state if using Adam
    if optimizer_type == 'adam':
        m = zeros_like(params)
        v = zeros_like(params)
        t = 1  # Time step
    
    # Accumulate gradients via atomic additions (simulated here with standard addition for clarity)
    # In practice, this is done in parallel with atomicAdd
    total_gradient = zeros_like(params)
    for i in range(len(residuals)):
        r_i = residuals[i]
        grad_i = gradients_per_sample[i]
        
        if optimizer_type == 'constant':
            # L2 norm gradient: 2 * r_i * grad_i
            # L1 norm gradient: sign(r_i) * grad_i
            # Note: The source mentions support for both, but does not specify if the user selects one globally
            # or if it's mixed. We assume a global norm selection for the solver instance.
            # Assuming L2 for this block, L1 would use sign(r_i)
            contribution = 2.0 * r_i * grad_i
        elif optimizer_type == 'adam':
            contribution = 2.0 * r_i * grad_i # Assuming L2 for Adam variant as standard
        
        total_gradient += contribution
        
    # Update parameters
    if optimizer_type == 'constant':
        params = params - alpha * total_gradient
    elif optimizer_type == 'adam':
        # Adam update step
        m = beta1 * m + (1 - beta1) * total_gradient
        v = beta2 * v + (1 - beta2) * (total_gradient ** 2)
        m_hat = m / (1 - beta1 ** t)
        v_hat = v / (1 - beta2 ** t)
        params = params - alpha * m_hat / (sqrt(v_hat) + epsilon)
        t += 1
        
    return params
```

**Edge Cases and Pitfalls:**
- **Ill-conditioning:** While more stable than direct methods, extremely ill-conditioned systems may still require careful tuning of $\alpha$ or the Adam parameters [S1, unofficial].
- **Atomic Operations:** In GPU implementations, atomic additions must be used to prevent race conditions when multiple threads update the same parameter gradient [S1, unofficial].
- **L1 vs L2 Choice:** The choice between $L_1$ and $L_2$ affects robustness to outliers. $L_2$ penalizes large residuals more strongly, while $L_1$ is more robust to outliers but may converge slower in some regimes [S1, unofficial].

## Subtopics

### [CGLS Solver (Conjugate Gradient Least Squares)](3285-cgls-solver-conjugate-gradient-least-squares.md)
The CGLS solver is a sparse linear least squares solver used to avoid the ill-conditioning of normal equations ($A^T A$), providing numerically stable results for large or rank-deficient problems. It builds on [Gradient Descent Solver](3280-gradient-descent-solver.md).

### [Partially Learned Gradient Descent](3771-partially-learned-gradient-descent.md)
Partially Learned Gradient Descent is an iterative reconstruction method that unrolls a fixed number of optimization steps into a neural network, combining a fixed physics-based gradient step with a learnable convolutional network update. It initializes the reconstruction with a filtered backprojection (FBP) estimate and, in its base form, does not use a regularization term in the network input. Unlike standard Gradient Descent which uses a fixed step size, this method learns the update rule via convolutional layers while retaining the physical forward model and gradient structure, making it more interpretable and robust to distribution shifts than fully learned end-to-end CNNs. It builds on [Gradient Descent Solver](3280-gradient-descent-solver.md).

### [Newton's Method](6442-newton-s-method.md)
Newton's method is a second-order optimization algorithm that approximates the objective function $f$ using a second-order Taylor expansion to determine an update step that targets the stationary point where the gradient is zero. It sets itself apart from first-order solvers like Gradient Descent by incorporating local curvature information via the Hessian matrix, allowing for faster convergence near optima, though at the cost of significant computational overhead for high-dimensional problems.

### [Inexact Line Search Optimization](7741-inexact-line-search-optimization.md)
Inexact Line Search Optimization is an iterative method that adapts the step size in gradient descent to achieve a reduction in the objective function while ensuring sufficiently fast progress. It solves the problem of slow convergence in standard gradient descent by dynamically selecting a step size that guarantees descent without the computational cost of finding the exact minimum along the search direction. Unlike exact line search, which may suffer from convergence deterioration, this inexact strategy yields reasonable results and descends much faster. It builds on [Gradient Descent Solver](3280-gradient-descent-solver.md).

### SGD with Positive t-Distribution
An ablation study variant that uses Stochastic Gradient Descent (SGD) with a positive Student's t distribution [S2]. It replaces Gaussian distributions with Student's t distributions, removes negative components and SGHMC (Stochastic Gradient Hamiltonian Monte Carlo), and adds a low-pass filter (0.3 diagonal offset) to ensure fair comparison with similar component counts [S2]. Used in the Ablation Study to provide a fair comparison with similar component numbers against methods using the low-pass filter [S2].

## Operating characteristics
- **Time Complexity:** Per iteration, the complexity is proportional to the number of samples and parameters, $O(N \cdot D)$, where $N$ is the number of samples and $D$ is the dimensionality of the parameter space. It avoids the $O(D^3)$ cost of matrix inversion in direct methods [S1, unofficial].
- **Memory Complexity:** $O(D)$ for storing parameters and gradients, as it is matrix-free. This is significantly lower than methods requiring storage of the Jacobian or Hessian matrices [S1, unofficial].
- **Performance:** Shows stable behavior for ill-conditioned systems, comparable to CGLS, but is generally slower than computing the matrix $A^T A$ and vector $A^T b$ directly for well-conditioned, small-scale problems [S1, unofficial].
- **Strengths:** Robust to ill-conditioning, low memory footprint, supports adaptive learning rates (Adam), and flexible loss functions ($L_1, L_2$) [S1, unofficial].
- **Limitations:** Slower convergence for well-conditioned problems compared to direct solvers; requires careful tuning of learning rates or Adam parameters [S1, unofficial].
- **When to Use:** Large-scale volume rendering optimization, ill-conditioned systems, memory-constrained environments [S1, unofficial].
- **When Not to Use:** Small, well-conditioned problems where direct solvers are faster and more accurate [S1, unofficial].

## Background
The Gradient Descent Solver was implemented as part of the DiffDVR framework for transfer function optimization in comparative volume rendering [S1, unofficial]. It builds on the Adam optimizer introduced by Kingma and Ba [KB15] and leverages the stability of gradient-based methods for ill-conditioned inverse problems in volume rendering. The method was evaluated on various datasets, demonstrating its effectiveness in optimizing transfer functions for volume visualization [S1, unofficial].

## Sources
[S1] Transfer Function Optimization for Comparative Volume Rendering, https://doi.org/10.48550/arxiv.2306.05885, fetched 2026-10-26 (unofficial)
[S2] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
[KB15] Kingma, D. P., & Ba, J. (2015). Adam: A Method for Stochastic Optimization. arXiv preprint arXiv:1412.6980.
