# Plateau-Reduced Differentiable Path Tracing

Also called: PR-DPT, Differentiable Path Tracing with Derivative-Kernel (Our∂κP), Convolutional Gradient Estimation, Our∂κP

## Overview
Plateau-Reduced Differentiable Path Tracing (PR-DPT) is an inverse rendering method that alleviates optimization plateaus (regions of zero gradient) by convolving the rendering equation with a smoothing kernel over the parameter space [S1]. Unlike standard differentiable renderers that suffer from discontinuous gradients due to visibility changes, PR-DPT ensures that small parameter changes continuously influence the objective function in expectation, allowing gradient-based optimizers to converge on problems with intricate light transport like caustics [S1]. It sets itself apart from its siblings by requiring only a forward path tracer and skipping the computation of the renderer's internal gradient tape or adjoint scattering, making it significantly faster than adjoint-based methods while remaining compatible with non-differentiable renderers [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Scene function $f(x, \theta)$ (the renderer), current parameters $\theta \in \Theta$, kernel standard deviation $\sigma$, number of samples $N$.
- **Outputs:** Gradient estimate $\nabla_\theta Q(\theta)$.

**Data Structures:**
- $\theta$: Vector of scene parameters (e.g., position, rotation, texture values).
- $\tau$: Parameter offset vector, sampled from the kernel distribution.
- $x$: Light path sample.
- $P(\theta)$: The rendered image (or scalar loss) for parameters $\theta$.

**Formulas:**
The plateau-reduced rendering equation $Q(\theta)$ is defined as the convolution of the original rendering equation $P(\theta)$ with a kernel $\kappa$ over the parameter space $\Theta$ [S1]:
$$
Q(\theta) = \kappa * P(\theta) = \int_{\Theta} \int_{\Omega} \kappa(\tau) f(x, \theta - \tau) dx d\tau
$$
where $f(x, \theta)$ is the scene function, $x \in \Omega$ are light paths, $\theta \in \Theta$ are scene parameters, and $\tau$ is a parameter offset [S1].

The Monte Carlo estimator for $Q(\theta)$ is [S1]:
$$
\hat{Q} \approx \frac{1}{N} \sum_{i=1}^{N} \kappa(\tau_i) f(x_i, \theta - \tau_i)
$$

The gradient of the estimator can be computed in two ways depending on renderer differentiability [S1]:
1.  **Differentiable Renderer:**
    $$
    \frac{\partial Q}{\partial \theta} = \frac{1}{N} \sum_{i=1}^{N} \kappa(\tau_i) \frac{\partial P}{\partial \theta}(\theta - \tau_i)
    $$
2.  **Non-Differentiable Renderer (Primary Method):**
    $$
    \frac{\partial Q}{\partial \theta} \approx \frac{1}{N} \sum_{i=1}^{N} \frac{\partial \kappa}{\partial \theta}(\tau_i) P(\theta - \tau_i)
    $$
    This method skips the renderer's gradient computation $\frac{\partial P}{\partial \theta}$ and only requires a forward model [S1].

The Gaussian kernel gradient used in the non-differentiable case is [S1]:
$$
\frac{\partial \kappa}{\partial \theta}(\tau) = \sqrt{\frac{2}{\sigma^2 \pi}} \frac{-\tau}{\sigma^2} \exp\left(\frac{-\tau^2}{2\sigma^2}\right)
$$

**Parameters:**
- $\kappa(\tau)$: Symmetric monotonous decreasing function. Gaussian is used by default [S1].
- $\sigma$: Standard deviation of the blur kernel. Initial value $\sigma_0 \in [0, 1]$ (typical default 0.5, recommended ~50% of domain). Minimum $\sigma_m = 0.01$ to avoid numerical instability [S1].
- $N$: Number of perturbations (samples) per iteration. Typical values: 2, 4, 8, 16, 32. $N=2$ is often sufficient [S1].
- $\sigma_0$: Initial kernel spread.
- $\sigma_m$: Minimum $\sigma$ during annealing.

**Steps:**
1.  **Normalize Parameters:** Normalize parameter spaces to $[0, 1]$ where possible (e.g., rotation) [S1].
2.  **Initialize $\sigma$:** Set $\sigma = \sigma_0$.
3.  **Loop for Gradient Estimation:** For each optimization step:
    a. Initialize gradient accumulator $G = 0$.
    b. Loop $i$ from $1$ to $N/2$:
        i. Sample $\xi \sim U(0, 1)$.
        ii. Compute offset magnitude $\tau_{mag} = \sqrt{-2\sigma^2 \log(\xi)}$.
        iii. Generate antithetic offsets $\tau = \tau_{mag}$ and $-\tau$ [S1].
        iv. Evaluate renderer at shifted parameters: $P_+ = P(\theta + \tau)$ and $P_- = P(\theta - \tau)$ [S1].
        v. Accumulate gradient contribution: $G \leftarrow G + (P_+ - P_-)$ [S1].
    c. Return $G / N$ [S1].
4.  **Adaptive Bandwidth Decay:** After approximately 50% of total iterations, decay $\sigma$ linearly from $\sigma_0$ to $\sigma_m$ using $\sigma_{t+1} = \sigma_0 - t(\sigma_0 - \sigma_m)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_gradient(P, theta, sigma, N):
    """
    Estimates the gradient of the plateau-reduced objective using the 
    derivative-kernel method (Eq. 10) with antithetic sampling.
    
    P: Function that renders an image/loss given parameters theta.
    theta: Current scene parameters.
    sigma: Standard deviation of the smoothing kernel.
    N: Total number of samples (must be even).
    """
    G = 0.0
    # Loop N/2 times to generate N samples via antithetic pairs
    for i in range(1, N // 2 + 1):
        # Sample from the distribution proportional to |d_kappa/d_theta|
        # Inverse CDF method for positivized gradient of Gaussian
        xi = uniform(0, 1)
        tau_mag = sqrt(-2 * sigma**2 * log(xi))
        
        # Antithetic sampling: generate tau and -tau
        # This provides a zero-variance estimator for the point-symmetric function
        tau = tau_mag
        
        # Evaluate renderer at shifted parameters
        # Note: P is the forward renderer (non-differentiable or differentiable)
        P_plus = P(theta + tau)
        P_minus = P(theta - tau)
        
        # Accumulate gradient contribution
        # The kernel gradient term is absorbed into the sampling distribution
        G = G + (P_plus - P_minus)
        
    return G / N

def optimize(P, theta_init, sigma_0, sigma_m, N, total_iterations):
    """
    Optimization loop with adaptive bandwidth decay.
    """
    theta = theta_init
    sigma = sigma_0
    
    for t in range(total_iterations):
        # Compute gradient using PR-DPT
        grad = estimate_gradient(P, theta, sigma, N)
        
        # Update parameters (e.g., using Adam)
        theta = update_parameters(theta, grad)
        
        # Adaptive bandwidth decay
        # Warm-start decay after 50% of iterations
        if t > total_iterations / 2:
            # Linear schedule from sigma_0 to sigma_m over the second half
            progress = (t - total_iterations / 2) / (total_iterations / 2)
            sigma = sigma_0 - progress * (sigma_0 - sigma_m)
            
    return theta
```

**Edge Cases and Pitfalls:**
- **$\sigma_0$ too small:** If $\sigma_0 < 0.2$, optimization fails to converge. As $\sigma_0 \to 0$, the method approaches rigid optimization, and the loss landscape is not smoothed, causing stagnation [S1].
- **$\sigma_0$ too large:** If $\sigma_0 \to 1$, perturbation samples may fall outside the view frustum. With $N=2$, it is unlikely to sample near the true parameter position, producing noisy gradients that cause divergence. This can be alleviated by enlarging the camera FOV (e.g., from $40^\circ$ to $60^\circ$) [S1].
- **High Dimensions:** Variance increases with higher dimensions, mitigated by increasing $N$ [S1].

## Subtopics

### Reparameterized Volume Sampling
Uses reparameterization trick for volume rendering. [Link](3309-reparameterized-volume-sampling.md)

### Boundary Path Integral Formulation
Handles discontinuities via boundary terms. [Link](3389-boundary-path-integral-formulation.md)

### Reparameterized Monte Carlo Volume Rendering
Applies reparameterization to MC volume rendering. [Link](4798-reparameterized-monte-carlo-volume-rendering.md)

### Differential path integral estimator with MIS and NEE
Uses MIS and NEE for differentiable path tracing. [Link](6332-differential-path-integral-estimator-with-mis-and-nee.md)

### Reparametrized Integral Gradients
Computes gradients via integral reparameterization. [Link](6785-reparametrized-integral-gradients.md)

### Monte Carlo Estimation of Differentiable Path Integrals
General MC approach for differentiable path integrals. [Link](10814-monte-carlo-estimation-of-differentiable-path-integrals.md)

### Reparameterization
General reparameterization trick. [Link](11557-reparameterization.md)

## Operating characteristics
- **Time Complexity:** Runtime scales linearly with $N$. The bulk of time is spent evaluating $f$ (rendering). Sampling and gradient computation overhead is negligible [S1]. At $N \approx 16$, runtime equals Mitsuba's constant baseline (which uses re-parametrization/adjoint methods for 1 sample) [S1].
- **Measured Performance:** On average 8x faster than differentiable rendering with Mitsuba (Our∂κP) [S1].
- **Scaling:** Convergence improves with $N$ up to $\sim 10$, then stagnates. Final parameter MSE decreases slightly ($9.5 \times 10^{-5}$ for $N=2$ vs $4.7 \times 10^{-6}$ for $N=16$), but no visible rendering improvement occurs due to the small error scale. $N>2$ yields no net benefit for tested tasks due to offsetting runtime [S1].
- **Strengths:** Does not require a gradient tape or compute graph. Compatible with alternative differentiable path tracers (e.g., Redner) and non-differentiable renderers. Resolves plateaus in optimization landscapes.
- **Limitations:** Variance increases with higher dimensions. Requires careful tuning of $\sigma_0$.
- **When to use:** Inverse rendering tasks with intricate light transport (caustics, global illumination) where standard differentiable renderers fail to converge due to plateaus.
- **When not to use:** When the renderer is already differentiable and converges well, or when extremely high-dimensional parameter spaces require very large $N$ for acceptable variance.

## Background
Originated in the paper "Plateau-Reduced Differentiable Path Tracing" by M. Fischer et al. (CVPR 2023) [S1]. The method addresses the issue of optimization plateaus in inverse rendering, where regions of zero gradient prevent convergence. It combines strengths of differentiable rasterization (smoothing) and differentiable path tracing (accuracy) by convolving the rendering equation with a smoothing kernel [S1]. It has been tested on tasks including light source optimization (caustics), shadow optimization, and geometry optimization [S1].

## Sources
[S1] Plateau-Reduced Differentiable Path Tracing, https://doi.org/10.1109/cvpr52729.2023.00417, fetched 2026-10-26
