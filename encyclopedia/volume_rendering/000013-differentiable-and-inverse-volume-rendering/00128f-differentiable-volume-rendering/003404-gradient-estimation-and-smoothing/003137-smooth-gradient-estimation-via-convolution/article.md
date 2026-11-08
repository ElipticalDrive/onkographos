# Smooth Gradient Estimation via Convolution

Also called: Combined gradient-smoothing operator, Convolutional Smoothing for Gradients, Gradient Smoothing via Convolution, Leibniz Rule Gradient Estimation via Convolution, Smooth Differentiation via Convolution, Smooth Gradient

## Overview
This method enables the computation of non-zero gradients for differentiable volume rendering by smoothing the rendering integrand via convolution with a kernel, thereby allowing the differential operator to move inside the integral via Leibniz's rule [S1]. It solves the problem where standard automatic differentiation yields zero gradients (plateaus) because the integrand is discontinuous with respect to scene parameters [S1]. It sets itself apart from [Non-local Approximated Gradients](6784-non-local-approximated-gradients.md) by explicitly enforcing smoothness through convolution to permit analytical differentiation exchange, rather than relying on non-local image statistics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene parameters $\theta \in \Theta$, incoming radiance $L_i$, and the rendering integrand $R(\omega_i ; \theta)$.
*   **Output:** The gradient of the rendered image $\nabla L(x, \omega_o ; \theta)$.

**Data Structures:**
*   **Smoothing Kernel $\kappa$:** A Gaussian distribution $\mathcal{N}(\tau, \sigma)$ defined over the parameter space $\Theta$. The standard deviation $\sigma$ controls the smoothing extent.
*   **Positivized CDF $P_G$:** A cumulative distribution function derived from the signed gradient-smoothing kernel, normalized to integrate to 1 and restricted to positive values for sampling [S1].

**Numbered Steps:**
1.  **Define the Smoothed Rendering Equation:** Apply a linear smoothing operator $S$ to the integrand $R$ to create a smooth function $\bar{L}$ [S1]:
    $$ \bar{L}(x, \omega_o ; \theta) = \int_{\Omega} S R(\omega_i ; \theta) d\omega_i $$
2.  **Apply Convolutional Smoothing:** Define the smoothing operator $S$ as a convolution with kernel $\kappa$ [S1]:
    $$ S f (\theta) = \int \kappa(\tau )f (\theta - \tau )d\tau $$
    where $\kappa$ is a Gaussian kernel [S1].
3.  **Exchange Differentiation and Integration:** Use Leibniz's rule to move the differential operator $D$ (gradient $\nabla$) inside the integral, which is valid because the integrand is now smooth [S1]:
    $$ D \bar{L}(x, \omega_o ; \theta) = \int_{\Omega} D S R(\omega_i ; \theta)d\omega_i $$
4.  **Expand the Integral:** Rearrange the terms to separate the differential operator applied to the kernel from the radiance term [S1]:
    $$ D \bar{L}(x, \omega_o ; \theta) = \int_{\Omega} \int_{\Theta} D \kappa(\tau )R(\omega_i ; \theta - \tau )d\tau d\omega_i $$
5.  **Monte Carlo Sampling:** Approximate the integral by sampling from the product space of light paths $\omega_i$ and scene parameters $\tau$ [S1].
    *   Sample light paths $\omega_i$ using standard rendering strategies [S1].
    *   Sample parameters $\tau$ using the gradient-specific sampling strategy described below.

**Gradient Estimation (Our Grad.) Details:**
*   **Operator:** The gradient operator is $D_G = \nabla = \partial/\partial\tau_i$ [S1].
*   **Kernel Derivative:** The combination of smoothing and differentiation for a Gaussian kernel is [S1]:
    $$ D_G \kappa(\tau ) = \nabla_i \mathcal{N}(\tau, \sigma) = -\frac{\tau_i}{\sigma^2} \cdot \mathcal{N}(\tau, \sigma) $$
*   **Sampling Strategy:**
    1.  **Positivization:** Convert the signed kernel derivative into a positive function and normalize it to form a PDF [S1].
    2.  **CDF Construction:** Compute the cumulative distribution function (CDF) $P_G$ of the positivized kernel [S1].
    3.  **Inverse Transform Sampling:** For a random variable $\xi \in [0, 1]$, sample $\tau$ using the inverse CDF $P_G^{-1}(\xi)$ [S1]:
        $$ P_G^{-1}(\xi) = \begin{cases} -\sqrt{2\sigma^2 \log(2\xi)} & \text{if } \xi \le 0.5 \\ \sqrt{2\sigma^2 \log(1 - \xi)} & \text{if } \xi > 0.5 \end{cases} $$
    4.  **Multi-dimensional Sampling:** Due to the separability of Gaussians, sample the dimension being differentiated using the inverse CDF above, and sample all other dimensions independently from a standard Gaussian distribution [S1].

**Parameters:**
*   **$\sigma$ (Standard Deviation):** Controls the width of the Gaussian kernel. Typical values are not specified in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Naive AD Failure:** Naive automatic differentiation computes $\int_{\Omega} D R(\omega_i ; \theta) d\omega_i$, which is incorrect because the integrand is discontinuous in $\theta$, preventing the exchange of integral and derivative operations [S1].
*   **Step Size:** Smooth gradients point in the correct direction but may cause large steps far from the optimum compared to classic gradients [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def sample_gradient_kernel(xi, sigma):
    """
    Samples a parameter tau from the positivized, normalized CDF 
    of the gradient-smoothing kernel using inverse transform sampling.
    
    Args:
        xi: Random float in [0, 1].
        sigma: Standard deviation of the Gaussian kernel.
        
    Returns:
        tau: Sampled parameter value.
    """
    if xi <= 0.5:
        return -math.sqrt(2 * sigma**2 * math.log(2 * xi))
    else:
        return math.sqrt(2 * sigma**2 * math.log(1 - xi))

def compute_smooth_gradient(R, theta, sigma, num_samples):
    """
    Computes the smooth gradient of the rendering equation via convolution.
    
    Args:
        R: Function representing the rendering integrand R(omega_i; theta).
        theta: Current scene parameters.
        sigma: Standard deviation of the Gaussian smoothing kernel.
        num_samples: Number of Monte Carlo samples.
        
    Returns:
        gradient_estimate: Estimated gradient vector.
    """
    gradient_sum = np.zeros_like(theta)
    
    for _ in range(num_samples):
        # 1. Sample light path omega_i (placeholder for standard rendering sampling)
        omega_i = sample_light_path() # [G] Placeholder for standard light path sampling
        
        # 2. Sample parameter tau from the gradient kernel
        # Sample the dimension being differentiated
        xi = np.random.uniform(0, 1)
        tau_diff = sample_gradient_kernel(xi, sigma)
        
        # Sample other dimensions independently from Gaussian
        tau_other = np.random.normal(0, sigma, size=len(theta) - 1) # [G] Assuming 1D differentiation for simplicity in structure
        
        # Construct full tau vector (assuming theta[0] is the differentiated dimension)
        tau = np.zeros_like(theta)
        tau[0] = tau_diff
        tau[1:] = tau_other
        
        # 3. Evaluate integrand at shifted parameter
        # R(omega_i; theta - tau)
        shifted_theta = theta - tau
        radiance = R(omega_i, shifted_theta)
        
        # 4. Compute kernel derivative weight: -tau_i / sigma^2 * N(tau, sigma)
        # For the differentiated dimension (index 0)
        kernel_val = np.exp(-0.5 * (tau[0]**2) / sigma**2) / (math.sqrt(2 * math.pi) * sigma)
        weight = -tau[0] / sigma**2 * kernel_val
        
        # 5. Accumulate gradient contribution
        # The gradient estimate is the average of weight * radiance
        gradient_sum += weight * radiance
        
    return gradient_sum / num_samples
```

## Subtopics

### [Hessian Estimation via Convolution](6469-hessian-estimation-via-convolution.md)
Hessian Estimation via Convolution computes the second-order derivatives (the Hessian matrix) of a scalar field by convolving the field with a kernel that combines Gaussian smoothing and the Hessian differential operator. It solves the problem of estimating $\nabla^2 f$ in differentiable volume rendering where direct differentiation is unstable or impossible due to stochastic sampling. It sets itself apart from [Smooth Gradient Marginalization](6472-smooth-gradient-marginalization.md) by explicitly constructing the $n \times n$ Hessian field rather than marginalizing gradient uncertainties, and it differs from [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md) by operating on second-order derivatives instead of first-order gradients.

### [Smooth Gradient Marginalization](6472-smooth-gradient-marginalization.md)
Smooth Gradient Marginalization is a technique to sample an $n$-dimensional Gaussian gradient by marginalizing the probability density function (PDF) over all dimensions except the one being differentiated, allowing individual dimension sampling. It differs from methods that only blur in the direction of differentiation by blurring all dimensions, which is more consistent with higher-order differentials.

## Operating characteristics
*   **Time Complexity:** $O(n)$ function evaluations without importance sampling (IS), $O(n)$ with IS, and $O(1)$ with Aggregate IS [S1].
*   **Measured Performance:** Classic gradients are zero almost everywhere (plateaus) except where overlaps occur, causing non-convergence. Smooth gradients point in the right direction but make large steps far from the optimum [S1].
*   **Strengths:** Enables differentiation of discontinuous integrands; provides non-zero gradients for optimization.
*   **Limitations:** Large step sizes far from the optimum; requires careful tuning of $\sigma$ [S1].
*   **When to use:** When standard automatic differentiation fails due to discontinuities in the rendering integrand.
*   **When not to use:** When high-order curvature information is required (use Hessian Estimation) or when non-local gradient approximations are preferred.

## Background
The method originates from the work on Stochastic Gradient Estimation for Higher-Order Differentiable Rendering [S1]. It addresses the fundamental issue that the differential operator cannot be moved inside the integral for discontinuous integrands, leading to incorrect gradients in differentiable rendering [S1]. The approach leverages Leibniz's rule by enforcing smoothness through convolution [S1].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
