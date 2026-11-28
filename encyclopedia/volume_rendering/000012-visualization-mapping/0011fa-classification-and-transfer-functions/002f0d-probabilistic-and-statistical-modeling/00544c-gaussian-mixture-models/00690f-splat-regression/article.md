# Splat Regression

Also called: Splat Function Training

## Overview
Splat Regression is a probabilistic modeling technique that represents a target function as a linear combination of anisotropic bump functions, termed "splats," each weighted by an output vector [S1, unofficial]. It solves the problem of fitting complex, potentially discontinuous functions by optimizing over the space of mixing measures using Wasserstein-Fisher-Rao gradient flows [S1, unofficial]. Unlike standard Gaussian Mixture Models which often assume isotropic components or fixed shapes, splat regression allows each component to locally adjust its scale and direction (covariance) to achieve high accuracy and interpretability [S1, unofficial]. It is distinguished from siblings like the Gaussian Sum Transfer Function by its explicit formulation of the splat as a transport of a mother distribution, enabling the use of optimal transport theory for optimization [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A dataset of $n$ samples $(x_i, y_i)$ where $x_i \in \mathbb{R}^d$ and $y_i \in \mathbb{R}^p$, and a target number of splats $k$.
- **Output**: A set of parameters $\{(v_j, A_j, b_j)\}_{j=1}^k$ defining the splat model $f_\mu(x)$.

**Data Structures**
The model is defined by a mixing measure $\mu$ on the space $\mathbb{R}^p \times BW_\rho(\mathbb{R}^d)$, where $BW_\rho(\mathbb{R}^d)$ is the set of splats. Each splat component $j$ is parameterized by:
- $v_j \in \mathbb{R}^p$: The value vector (amplitude/output weight).
- $A_j \in \mathbb{R}^{d \times d}$: The covariance matrix factor (shape/scale/direction). The covariance is $\Sigma_j = A_j A_j^T$.
- $b_j \in \mathbb{R}^d$: The mean vector (center).

The splat density $\rho_{A,b}(x)$ is defined as the pushforward of a centered isotropic mother splat $\rho$ (typically standard normal $\mathcal{N}(0, I_d)$) by the affine map $x \mapsto Ax + b$.

**Algorithm Steps**
1. **Initialization**: Initialize the means $b_j$ and covariance factors $A_j$ for $j=1 \dots k$.
   - Set $b_j = \cos\left(\frac{\pi(2j + 1)}{2k}\right)$ for 1D problems (Chebyshev grid) [S1, unofficial].
   - Set $A_j = \frac{|b_{j-1} - b_{j+1}|}{2}$ with boundary conditions $b_{-1} = 0$ and $b_{k+1} = 1$ [S1, unofficial].
   - Initialize $v_j$ (e.g., to zero or based on local data averages) [G].
2. **Loss Definition**: Define the loss function $F(f_\mu)$, typically Mean Squared Error (MSE) over the training data:
   $$ F(f_\mu) = \frac{1}{n} \sum_{i=1}^n \| f_\mu(x_i) - y_i \|_2^2 $$
   where $f_\mu(x) = \sum_{j=1}^k v_j \rho_{A_j, b_j}(x)$ [S1, unofficial].
3. **Gradient Computation**: Compute the Wasserstein-Fisher-Rao gradients for the parameters $(v, A, b)$ with respect to the measure $\mu$. The gradients are derived from the first variation $\delta F(x)$ of the loss [S1, unofficial]:
   - Gradient w.r.t. $v_j$:
     $$ \nabla^{W}_{v_j} F = \int \delta F(x) \rho_{A_j, b_j}(x) \pi(dx) $$
     In practice, this is approximated by summing over data points weighted by the splat density:
     $$ \nabla^{W}_{v_j} F \approx \frac{1}{n} \sum_{i=1}^n \frac{\partial F}{\partial f(x_i)} \rho_{A_j, b_j}(x_i) $$
   - Gradient w.r.t. $b_j$:
     $$ \nabla^{W}_{b_j} F = - \int \langle \delta F(x), v_j \rangle \nabla_x \log \rho_{A_j, b_j}(x) \rho_{A_j, b_j}(x) \pi(dx) $$
   - Gradient w.r.t. $A_j$:
     $$ \nabla^{W}_{A_j} F = \int \langle \delta F(x), v_j \rangle \left( I + \nabla_x \log \rho_{A_j, b_j}(x)(x - b_j)^T A_j^{-T} \right) \rho_{A_j, b_j}(x) \pi(dx) $$
     where $\nabla_x \log \rho_{A,b}(x) = -A^{-T}(x-b)$ for Gaussian splats [S1, unofficial].
4. **Update**: Update the parameters $(v_j, A_j, b_j)$ along the computed gradient flow. This can be done using standard optimizers (e.g., Adam) or specific Wasserstein-Fisher-Rao gradient descent steps [S1, unofficial].
5. **Iteration**: Repeat steps 2-4 until convergence.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def splat_regression(x_data, y_data, k, num_iterations, learning_rate):
    """
    Fits a Splat Regression model to data.
    x_data: (n, d) array of inputs
    y_data: (n, p) array of targets
    k: number of splats
    """
    n, d = x_data.shape
    p = y_data.shape[1]
    
    # Initialization
    # 1D Chebyshev initialization example for b, extended to d dimensions if needed
    # For d > 1, a grid or random initialization might be used, but the source specifies 1D Chebyshev
    # We assume d=1 for the specific Chebyshev init, or generalize by dimension-wise application
    if d == 1:
        b = np.cos(np.pi * (2 * np.arange(k) + 1) / (2 * k))
        b_ext = np.concatenate(([0], b, [1]))
        A = np.abs(b_ext[:-2] - b_ext[2:]) / 2
        A = A.reshape(-1, 1, 1) # (k, 1, 1) for d=1
    else:
        # [G] Generic initialization for d > 1 not specified in source. 
        # Using random isotropic initialization as a placeholder.
        b = np.random.randn(k, d)
        A = np.eye(d).reshape(1, d, d).repeat(k, axis=0)
        
    v = np.zeros((k, p)) # (k, p)
    
    # Precompute splat densities for all data points and splats
    # rho[j, i] = density of splat j at data point i
    # For Gaussian: rho(x) = (2*pi)^(-d/2) |A|^-1 exp(-0.5 * (x-b)^T A^-T A^-1 (x-b))
    # Simplified for isotropic mother: A^-T A^-1 = (A A^T)^-1
    
    for step in range(num_iterations):
        # Forward pass: compute f_mu(x) for all data points
        # f_mu(x_i) = sum_j v_j * rho_j(x_i)
        # Shape: (n, p)
        
        # Compute Mahalanobis distance squared for each splat and data point
        # dist_sq[j, i] = (x_i - b_j)^T (A_j A_j^T)^-1 (x_i - b_j)
        # This requires inverting A_j A_j^T for each splat
        inv_covs = np.linalg.inv(A @ A.transpose(0, 2, 1)) # (k, d, d)
        
        # Compute (x_i - b_j)
        diff = x_data[np.newaxis, :, :] - b[:, np.newaxis, :] # (k, n, d)
        
        # Compute quadratic form
        # (k, n, d) @ (k, d, d) -> (k, n, d)
        temp = diff @ inv_covs
        # (k, n, d) @ (k, n, d)^T -> (k, n)
        dist_sq = np.sum(temp * diff, axis=2)
        
        # Compute log determinant of covariance
        # log |A A^T| = 2 log |A|
        log_det = 2 * np.sum(np.log(np.abs(np.linalg.det(A))), axis=(1, 2)) # (k,)
        
        # Compute densities
        # rho = exp(-0.5 * dist_sq - 0.5 * d * log(2*pi) - 0.5 * log_det)
        log_rho = -0.5 * dist_sq - 0.5 * d * np.log(2 * np.pi) - 0.5 * log_det[:, np.newaxis]
        rho = np.exp(log_rho) # (k, n)
        
        # Compute model output f_mu(x)
        # v shape (k, p), rho shape (k, n) -> (n, p)
        f_mu = rho.T @ v # (n, p)
        
        # Compute loss gradient (first variation) delta F(x)
        # F = 1/n sum ||f_mu(x_i) - y_i||^2
        # dF/df(x_i) = 2/n (f_mu(x_i) - y_i)
        error = f_mu - y_data # (n, p)
        delta_F = 2.0 / n * error # (n, p)
        
        # Compute gradients for v, b, A
        
        # Gradient for v: sum_i delta_F(x_i) * rho_j(x_i)
        # (n, p) * (k, n)^T -> (k, p)
        grad_v = delta_F.T @ rho # (k, p)
        
        # Gradient for b: - sum_i <delta_F(x_i), v_j> * grad_x log rho_j(x_i) * rho_j(x_i)
        # grad_x log rho = - (A A^T)^-1 (x - b) = - inv_cov @ diff
        # inner product <delta_F, v_j> is dot product over p dimensions
        # (n,) = sum_p (delta_F * v_j)
        inner_prod = np.sum(delta_F * v[:, np.newaxis, :], axis=2) # (k, n)
        
        # grad_x log rho shape (k, n, d)
        grad_log_rho = - (inv_covs @ diff[:, :, :, np.newaxis]).squeeze(axis=3) # (k, n, d)
        
        # grad_b = - sum_i inner_prod_j_i * grad_log_rho_j_i
        # (k, n) * (k, n, d) -> (k, d)
        grad_b = - np.sum(inner_prod[:, :, np.newaxis] * grad_log_rho, axis=1)
        
        # Gradient for A: sum_i <delta_F(x_i), v_j> * (I + grad_log_rho_j_i @ (x_i - b_j)^T) * A_j^-T * rho_j(x_i)
        # Term 1: I
        # Term 2: grad_log_rho @ diff^T
        # grad_log_rho shape (k, n, d), diff shape (k, n, d)
        # We need (I + grad_log_rho @ diff^T) @ A^-T
        # Let M = I + grad_log_rho @ diff^T. M shape (k, d, d)
        # M = I + einsum('kn d, kn e -> k d e', grad_log_rho, diff)
        M = np.eye(d)[np.newaxis, :, :] + np.einsum('knd,kne->kde', grad_log_rho, diff)
        
        # A^-T shape (k, d, d)
        A_inv_T = np.linalg.inv(A).transpose(0, 2, 1)
        
        # Term inside integral: M @ A^-T
        term = M @ A_inv_T # (k, d, d)
        
        # grad_A = sum_i inner_prod_j_i * term_j * rho_j(x_i)
        # (k, n) * (k, d, d) * (k, n) -> (k, d, d)
        grad_A = np.sum(inner_prod[:, :, np.newaxis, np.newaxis] * rho[:, :, np.newaxis, np.newaxis] * term[np.newaxis, :, :, :], axis=1)
        
        # Update parameters
        v -= learning_rate * grad_v
        b -= learning_rate * grad_b
        A -= learning_rate * grad_A
        
    return v, A, b
```

**Edge Cases and Pitfalls**
- **Singular Covariance**: If $A_j A_j^T$ becomes singular, the density is undefined. Regularization or constraints on $A$ may be needed [G].
- **High Dimensions**: The Chebyshev initialization is specified for 1D. For $d > 1$, the source does not specify an initialization strategy [S1, unofficial].
- **Numerical Stability**: Computing the inverse of $A$ and the determinant can be numerically unstable if $A$ is ill-conditioned. Using the Cholesky decomposition or log-determinants is recommended [G].

## Subtopics

### Wasserstein-Fisher-Rao Gradient Flow Optimization
An optimization algorithm for fitting splat models that treats the model parameters as a hierarchical 'distribution over distributions' (a mixing measure $\mu$) [S1, unofficial]. It computes gradient updates in parameter space using Wasserstein-Fisher-Rao gradient flows [S1, unofficial].
- **Steps**:
  1. Interpret splat parameters $(v, A, b)$ as elements of a mixing measure $\mu$ on $\mathbb{R}^p \times BW_\rho(\mathbb{R}^d)$ [S1, unofficial].
  2. Define the loss function on the training dataset [S1, unofficial].
  3. Compute the gradient of the loss with respect to $\mu$ using Wasserstein-Fisher-Rao geometry [S1, unofficial].
  4. Update $\mu$ along the gradient flow [S1, unofficial].
- **Comparison**: Differs from optimizing ReLU networks because the first-layer output neurons in Splat models are represented as elements of a Bures-Wasserstein manifold, creating a fundamentally different parameter space geometry [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Each iteration involves computing densities for all $k$ splats and $n$ data points, which is $O(k \cdot n \cdot d^3)$ due to matrix inversions and multiplications [G]. Gradient updates are $O(k \cdot n \cdot d^2)$ or $O(k \cdot d^3)$ depending on implementation [G].
- **Memory Complexity**: $O(k \cdot n)$ to store densities and gradients, plus $O(k \cdot d^2)$ for parameters [G].
- **Performance**: In 1D regression tasks (e.g., fitting a sawtooth function with 1000 samples), splat regression outperforms Haar wavelet decomposition and Chebyshev polynomial interpolation [S1, unofficial].
- **Strengths**: High interpretability, ability to model anisotropic features, unified framework for Gaussian Splatting [S1, unofficial].
- **Limitations**: Optimization geometry is complex (Bures-Wasserstein manifold), initialization sensitivity (Chebyshev init is slightly less performant than others in some cases) [S1, unofficial].
- **When to Use**: Low-dimensional regression, inverse problems, and scenarios requiring interpretable, locally adaptive basis functions [S1, unofficial].
- **When Not to Use**: High-dimensional data where the curse of dimensionality makes density estimation difficult, or when real-time inference is critical and pre-trained models are not available [G].

## Background
Splat Regression was introduced in the paper "Splat Regression Models" (2025) [S1, unofficial]. It lies at the intersection of computer graphics, machine learning, and optimal transport theory. The work recovers 3D Gaussian Splatting as a special case, providing a unified theoretical framework that disambiguates the inverse problem, the model, and the optimization algorithm [S1, unofficial]. It builds on the theory of Wasserstein-Fisher-Rao gradient flows and the Bures-Wasserstein metric [S1, unofficial].

## Sources
[S1] Splat Regression Models, https://doi.org/10.48550/arxiv.2511.14042, fetched 2026-10-26 (unofficial)
