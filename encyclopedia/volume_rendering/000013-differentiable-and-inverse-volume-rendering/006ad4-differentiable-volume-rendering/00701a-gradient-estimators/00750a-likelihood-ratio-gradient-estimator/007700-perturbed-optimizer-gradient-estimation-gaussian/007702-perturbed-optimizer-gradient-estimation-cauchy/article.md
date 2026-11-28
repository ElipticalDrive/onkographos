# Perturbed Optimizer Gradient Estimation (Cauchy)

Also called: Jacobian of perturbed renderers with Cauchy noise

## Overview
This method estimates the gradient of a differentiable renderer's output with respect to its parameters $\theta$ by using noise drawn from a Cauchy distribution to perturb the optimizer's solution. It serves as a heavy-tailed alternative to the Gaussian-based Perturbed Optimizer Gradient Estimation, offering different variance properties for the likelihood-ratio gradient estimator. It builds on [Perturbed Optimizer Gradient Estimation (Gaussian)](23617-perturbed-optimizer-gradient-estimation-gaussian.md).

## How it works
**Inputs:**
*   $\theta$: The optimization parameters (e.g., pose, shape, lighting).
*   $\epsilon$: The smoothing parameter (scalar).
*   $y^*(\cdot)$: The differentiable renderer function mapping parameters to an output (e.g., image).
*   $Z$: A random noise vector drawn from a standard Cauchy distribution.

**Outputs:**
*   An unbiased estimate of the Jacobian $J_\theta y_\epsilon^*(\theta)$.

**Data Structures:**
*   $Z$: A vector of independent samples from a 1D Cauchy distribution. In implementation, these are typically generated via $Z_i = \tan(\pi(U_i - 0.5))$ where $U_i \sim \text{Uniform}(0, 1)$ [S1, unofficial].

**Steps:**
1.  Sample $N$ independent noise vectors $Z^{(1)}, \dots, Z^{(N)}$ from the Cauchy distribution.
2.  For each sample $k$:
    a. Compute the perturbed parameter vector: $\theta_k = \theta + \epsilon Z^{(k)}$.
    b. Render the perturbed scene: $y_k = y^*(\theta_k)$.
    c. Render the unperturbed scene: $y_0 = y^*(\theta)$. (Note: $y_0$ can be cached and reused for all $k$).
    d. Compute the potential function gradient $\nabla \nu(Z^{(k)})$. For a Cauchy distribution, the potential function is $\nu(z) = \ln(1 + z^2)$. The gradient with respect to $z$ is:
       $$ \nabla \nu(Z) = \frac{2Z}{1 + \|Z\|^2} $$
       where $\|Z\|^2$ is the squared Euclidean norm of $Z$ [S1, unofficial].
    e. Compute the gradient contribution for this sample:
       $$ g_k = (y_k - y_0) \left( \nabla \nu(Z^{(k)})^\top \frac{1}{\epsilon} \right) $$
3.  Average the contributions:
    $$ \hat{J}_\theta y_\epsilon^*(\theta) = \frac{1}{N} \sum_{k=1}^N g_k $$

**Formulas:**
The gradient of the expected output with respect to parameters $\theta$ is estimated as [S1, unofficial]:
$$ J_\theta y_\epsilon^*(\theta) = \mathbb{E}_Z \left[ (y^*(\theta + \epsilon Z) - y^*(\theta)) \nabla \nu(Z)^\top \frac{1}{\epsilon} \right] $$
where $Z$ follows a Cauchy distribution and $\nu(z) = \ln(1 + z^2)$ [S1, unofficial].

**Parameters:**
*   $\epsilon$: Smoothing parameter. Typical values depend on the scale of $\theta$ and the desired smoothness of the loss landscape. Not given in the sources.
*   $N$: Number of Monte Carlo samples. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Heavy Tails:** The Cauchy distribution has undefined variance. This can lead to high-variance gradient estimates if $\epsilon$ is large or if the renderer $y^*$ is highly sensitive to perturbations. [G] Implementation should monitor the magnitude of $Z$ and potentially clip extreme values or use robust averaging if numerical instability occurs.
*   **Singularity:** The term $\frac{2Z}{1 + \|Z\|^2}$ is well-defined for all $Z \in \mathbb{R}^d$. No singularity at $Z=0$ or infinity.
*   **Baseline Zero:** The term $\mathbb{E}_Z [y^*(\theta) \nabla \nu(Z)^\top]$ is exactly zero due to the symmetry of the Cauchy distribution and the odd nature of the gradient term $\frac{2Z}{1+\|Z\|^2}$ [S1, unofficial]. This allows the simplified form $(y^*(\theta + \epsilon Z) - y^*(\theta))$ instead of $y^*(\theta + \epsilon Z)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def cauchy_sample(n_samples, dim):
    """
    Generate n_samples of dim-dimensional Cauchy noise.
    Uses the property that if U ~ Uniform(0,1), then tan(pi*(U-0.5)) ~ Cauchy(0,1).
    """
    u = np.random.uniform(0, 1, size=(n_samples, dim))
    z = np.tan(np.pi * (u - 0.5))
    return z

def compute_cauchy_gradient_estimate(y_star, theta, epsilon, n_samples):
    """
    Estimate the Jacobian of the perturbed renderer w.r.t theta using Cauchy noise.
    
    Args:
        y_star: Function that renders the scene given parameters.
        theta: Current parameter vector.
        epsilon: Smoothing parameter.
        n_samples: Number of Monte Carlo samples.
        
    Returns:
        grad_estimate: Estimated gradient vector.
    """
    # 1. Sample noise
    Z = cauchy_sample(n_samples, len(theta))
    
    # 2. Render unperturbed scene (cached)
    y_0 = y_star(theta)
    
    # 3. Initialize gradient accumulator
    grad_accum = np.zeros_like(theta)
    
    # 4. Loop over samples
    for k in range(n_samples):
        z_k = Z[k]
        
        # Perturb parameters
        theta_perturbed = theta + epsilon * z_k
        
        # Render perturbed scene
        y_perturbed = y_star(theta_perturbed)
        
        # Compute difference
        diff = y_perturbed - y_0
        
        # Compute gradient of potential nu(z) = ln(1 + ||z||^2)
        # grad_nu = 2*z / (1 + ||z||^2)
        z_norm_sq = np.dot(z_k, z_k)
        grad_nu = 2.0 * z_k / (1.0 + z_norm_sq)
        
        # Compute contribution: diff * (grad_nu^T / epsilon)
        # Note: diff is scalar (or vector if rendering multiple outputs), 
        # grad_nu is vector. The formula implies a scalar output for simplicity 
        # in the source derivation. If y is vector, this needs outer product handling.
        # Assuming scalar output y for this implementation based on source notation.
        contribution = diff * (np.dot(grad_nu, 1.0 / epsilon))
        
        grad_accum += contribution
        
    # 5. Average
    grad_estimate = grad_accum / n_samples
    
    return grad_estimate
```

## Subtopics

### Perturbed Optimizer Sensitivity Estimation w.r.t Smoothing Parameter (Cauchy)
This variant estimates the sensitivity of the perturbed optimizer's output with respect to the smoothing parameter $\epsilon$ itself, rather than the model parameters $\theta$. It uses the same Cauchy noise distribution. The formula is [S1, unofficial]:
$$ J_\epsilon y_\epsilon^*(\theta) = \mathbb{E}_Z \left[ (y^*(\theta + \epsilon Z) - y^*(\theta)) \frac{\nabla \nu(Z)^\top Z - 1}{\epsilon} \right] $$
where $\nabla \nu(Z)^\top Z = \frac{2\|Z\|^2}{1+\|Z\|^2}$. The baseline term $\mathbb{E}_Z [y^*(\theta) \frac{\nabla \nu(Z)^\top Z - 1}{\epsilon}]$ is zero, proven via integration by parts [S1, unofficial]. This is useful for adaptive smoothing strategies where $\epsilon$ is optimized [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot T_{render})$, where $N$ is the number of samples and $T_{render}$ is the cost of one rendering pass. The noise generation and potential gradient computation are negligible compared to rendering.
*   **Memory Complexity:** $O(N \cdot D)$ for storing noise samples, where $D$ is the dimension of $\theta$. $O(1)$ additional memory if samples are processed sequentially.
*   **Variance:** The Cauchy distribution has heavy tails, which can lead to higher variance in gradient estimates compared to Gaussian noise, especially for large $\epsilon$. However, it may provide more robust gradients in the presence of outliers or non-smooth loss landscapes. [G] Variance reduction techniques (e.g., control variates) are recommended [S1, unofficial].
*   **Strengths:** Unbiased gradient estimate. Simple implementation. Heavy tails can help escape local minima in optimization.
*   **Limitations:** High variance due to heavy tails. Requires careful tuning of $\epsilon$.
*   **When to use:** When the loss landscape is non-smooth or when Gaussian noise leads to poor convergence. When robustness to outliers is desired.
*   **When not to use:** When low-variance gradients are critical and computational budget for many samples is limited. When the renderer is very sensitive to small perturbations (small $\epsilon$ required, but Cauchy noise can still produce large jumps).

## Background
This method is derived from the theory of perturbed optimizers and differentiable rendering. It is presented in the context of adaptive smoothing and variance reduction for differentiable renderers. The specific formulation for Cauchy noise is detailed in Proposition 6 and Proposition 7 of the source [S1, unofficial]. The work is associated with the development of differentiable perturbed renderers for pose optimization and other inverse graphics tasks.

## Sources
[S1] Differentiable Rendering with Perturbed Optimizers, https://core.ac.uk/works/403056225, fetched 2026-10-29 (unofficial)
