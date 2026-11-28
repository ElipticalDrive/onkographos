# Perturbed Optimizer Gradient Estimation (Gaussian)

Also called: Jacobian of perturbed optimizer w.r.t epsilon (Gaussian), Jacobian of renderers with Gaussian noise, Perturbed Optimizer Sensitivity Estimation w.r.t Smoothing Parameter (Gaussian), Score Function Estimator with Gaussian Perturbation

## Overview
This method estimates the gradient of a differentiable renderer's output with respect to its parameters $\theta$ by using noise drawn from a standard Gaussian distribution to perturb the optimizer's solution. It serves as the primary variant of the Likelihood-Ratio Gradient Estimator for continuous noise distributions, offering a specific variance profile compared to heavy-tailed alternatives like the Cauchy variant. It builds on [Likelihood-Ratio Gradient Estimator](23160-likelihood-ratio-gradient-estimator.md).

## How it works
**Inputs and Outputs:**
- Input: Renderer parameters $\theta \in \mathbb{R}^d$, smoothing parameter $\epsilon > 0$, and a sample $Z \sim \mathcal{N}(0, I_d)$ drawn from a standard Gaussian distribution.
- Output: Gradient estimate $J_\theta y_\epsilon^*(\theta)$ with respect to $\theta$, and optionally the sensitivity $J_\epsilon y_\epsilon^*(\theta)$ with respect to $\epsilon$.
- $y^*(\cdot)$ denotes the output of the perturbed optimizer (the renderer's result).

**Data Structures:**
- $Z$: A vector of size $d$ containing independent standard normal samples.
- $\nu(Z)$: The potential function associated with the noise distribution. For Gaussian noise, $\nu(Z) = \frac{\|Z\|^2}{2}$.

**Formulas:**
The gradient of the expected output with respect to parameters $\theta$ is estimated as [S1, unofficial]:
$$
J_\theta y_\epsilon^*(\theta) = \mathbb{E}_Z \left[ (y^*(\theta + \epsilon Z) - y^*(\theta)) \nabla \nu(Z)^\top \frac{1}{\epsilon} \right]
$$
where $\nabla \nu(Z) = Z$ for the standard Gaussian distribution.

The sensitivity of the perturbed optimizer with respect to the smoothing parameter $\epsilon$ is estimated as [S1, unofficial]:
$$
J_\epsilon y_\epsilon^*(\theta) = \mathbb{E}_Z \left[ (y^*(\theta + \epsilon Z) - y^*(\theta)) \frac{\nabla \nu(Z)^\top Z - 1}{\epsilon} \right]
$$
For Gaussian noise, $\nabla \nu(Z)^\top Z = \|Z\|^2$, so the term becomes $\frac{\|Z\|^2 - 1}{\epsilon}$.

**Derivation Steps:**
1.  **Change of Variable:** Apply the change of variable $z' = \theta + \epsilon z$ to the integral representation of the expected output $\int_{-\infty}^{\infty} y^*(\theta + \epsilon z) \mu(z) dz$ [S1, unofficial].
2.  **Differentiate w.r.t $\theta$:** Differentiate under the integral sign with respect to $\theta$ [S1, unofficial].
3.  **Chain Rule on Density:** Apply the chain rule to the derivative of the density $\mu$, introducing the gradient of the potential function $\nabla \nu$ [S1, unofficial]. This yields an integral involving $\frac{1}{\epsilon} \nabla \nu \left( \frac{z' - \theta}{\epsilon} \right)^\top$ [S1, unofficial].
4.  **Inverse Change of Variable:** Perform the inverse change of variable to return to an expectation over $Z$ [S1, unofficial].
5.  **Baseline Vanishing:** Show that the baseline term $\mathbb{E}_Z [y^*(\theta) \nabla \nu(Z)^\top]$ vanishes because $Z$ is centered (mean zero) [S1, unofficial]. Specifically, $\mathbb{E}_Z [Z^\top] = 0$ [S1, unofficial].
6.  **Final Form:** Combine terms to obtain the final estimator for $J_\theta$ [S1, unofficial].

For the sensitivity w.r.t $\epsilon$:
1.  **Differentiate w.r.t $\epsilon$:** Differentiate the integral with respect to $\epsilon$ [S1, unofficial].
2.  **Chain Rule on Density:** Apply the chain rule to the density derivative w.r.t $\epsilon$, yielding a term involving $\frac{1}{\epsilon^2} \left( \nabla \nu \left( \frac{z' - \theta}{\epsilon} \right)^\top \frac{z' - \theta}{\epsilon} - 1 \right)$ [S1, unofficial].
3.  **Convert to Expectation:** Convert the integral back to an expectation form over $Z$ [S1, unofficial].
4.  **Baseline Vanishing:** Show the baseline term $\mathbb{E}_Z \left[ y^*(\theta) \frac{\nabla \nu(Z)^\top Z - 1}{\epsilon} \right]$ vanishes for standard Gaussian noise because $\mathbb{E}_Z [\|Z\|^2 - 1] = 0$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def estimate_gradient(theta, epsilon, y_star, num_samples=1):
    """
    Estimates the gradient of the renderer output y* w.r.t theta
    using Perturbed Optimizer Gradient Estimation with Gaussian noise.
    
    Args:
        theta: Array of renderer parameters.
        epsilon: Smoothing parameter.
        y_star: Function that returns the renderer output given parameters.
        num_samples: Number of Monte Carlo samples.
        
    Returns:
        grad_theta: Estimated gradient w.r.t theta.
    """
    dim = len(theta)
    grad_theta = np.zeros_like(theta)
    
    for _ in range(num_samples):
        # Sample Z from standard Gaussian distribution
        Z = np.random.normal(size=dim)
        
        # Perturbed parameters
        theta_perturbed = theta + epsilon * Z
        
        # Evaluate renderer at perturbed and unperturbed parameters
        y_perturbed = y_star(theta_perturbed)
        y_base = y_star(theta)
        
        # Difference in output
        delta_y = y_perturbed - y_base
        
        # Gradient of potential function nu(Z) = ||Z||^2 / 2 is Z
        # Term: delta_y * (Z^T / epsilon)
        # Note: If y is scalar, this is a vector. If y is vector, 
        # this requires outer product or specific handling depending on 
        # the definition of J_theta. Based on formula (53), it assumes 
        # scalar output or element-wise application.
        # For scalar y:
        grad_theta += delta_y * (Z / epsilon)
        
    grad_theta /= num_samples
    return grad_theta

def estimate_epsilon_sensitivity(theta, epsilon, y_star, num_samples=1):
    """
    Estimates the sensitivity of the renderer output y* w.r.t epsilon.
    """
    dim = len(theta)
    grad_epsilon = 0.0
    
    for _ in range(num_samples):
        Z = np.random.normal(size=dim)
        theta_perturbed = theta + epsilon * Z
        
        y_perturbed = y_star(theta_perturbed)
        y_base = y_star(theta)
        
        delta_y = y_perturbed - y_base
        
        # Term: delta_y * (||Z||^2 - 1) / epsilon
        norm_sq_Z = np.dot(Z, Z)
        sensitivity_term = (norm_sq_Z - 1) / epsilon
        
        grad_epsilon += delta_y * sensitivity_term
        
    grad_epsilon /= num_samples
    return grad_epsilon
```

**Parameters:**
- $\epsilon$: Smoothing parameter. Typical values are small positive numbers (e.g., $10^{-3}$ to $10^{-1}$), balancing bias and variance. Not given in the sources.
- $Z$: Standard Gaussian noise.

**Edge Cases and Pitfalls:**
- **Precision:** When $\epsilon$ is very small, the term $1/\epsilon$ can amplify numerical noise in $y^*(\theta + \epsilon Z) - y^*(\theta)$.
- **Degenerate Input:** If $y^*$ is not differentiable or the optimizer does not converge, the gradient estimate may be invalid.
- **Baseline Term:** The derivation relies on the baseline term vanishing. This holds for Gaussian noise because $\mathbb{E}[Z]=0$ and $\mathbb{E}[\|Z\|^2-1]=0$. This property does not hold for all distributions (e.g., it requires specific handling for Cauchy, though the baseline also vanishes there due to symmetry and specific moment properties [S1, unofficial]).

## Subtopics

### [Perturbed Optimizer Gradient Estimation (Cauchy)](23621-perturbed-optimizer-gradient-estimation-cauchy.md)
This method estimates the gradient of a differentiable renderer's output with respect to its parameters $\theta$ by using noise drawn from a Cauchy distribution to perturb the optimizer's solution. It serves as a heavy-tailed alternative to the Gaussian-based Perturbed Optimizer Gradient Estimation, offering different variance properties for the likelihood-ratio gradient estimator. It builds on [Perturbed Optimizer Gradient Estimation (Gaussian)](23617-perturbed-optimizer-gradient-estimation-gaussian.md).

### [Perturbed Differentiable Renderer](23624-perturbed-differentiable-renderer.md)
The Perturbed Differentiable Renderer formulates non-smooth rasterization and aggregation steps as optimization problems (argmax) and makes them differentiable by applying random perturbations to the input arguments, effectively smoothing the Heaviside and Z-buffer selection functions. Unlike methods that use gradient approximations in the backward pass (e.g., OpenDR, NMR) or smooth approximations of the forward pass (e.g., SoftRas, DIB-R), this method uses randomized smoothing of the optimization formulation, providing a theoretical link to perturbed optimizers and guaranteeing non-null gradients everywhere.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of Monte Carlo samples and $C$ is the cost of evaluating the renderer $y^*$.
- **Memory Complexity:** $O(d)$, where $d$ is the dimensionality of $\theta$, to store the noise samples and gradients.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear in the number of samples. Variance decreases as $1/N$.
- **Strengths:** Unbiased gradient estimate (under the likelihood-ratio framework). Simple to implement.
- **Limitations:** High variance for small $\epsilon$ due to the $1/\epsilon$ factor. Requires multiple samples for stable estimates.
- **When to use:** When an unbiased gradient estimate is required and the renderer is expensive to evaluate but differentiable via the perturbed optimizer framework.
- **When not to use:** When computational budget is extremely limited and biased but lower-variance estimators (like straight-through or soft approximations) are acceptable.

## Background
This method is derived from the likelihood-ratio identity applied to perturbed optimizers. It is presented in the context of differentiable rendering with perturbed optimizers [S1, unofficial]. The specific formulation for Gaussian noise leverages the properties of the Gaussian distribution (mean zero, specific second moments) to simplify the baseline terms.

## Sources
[S1] Differentiable Rendering with Perturbed Optimizers, https://core.ac.uk/works/403056225, fetched 2026-10-29 (unofficial)
