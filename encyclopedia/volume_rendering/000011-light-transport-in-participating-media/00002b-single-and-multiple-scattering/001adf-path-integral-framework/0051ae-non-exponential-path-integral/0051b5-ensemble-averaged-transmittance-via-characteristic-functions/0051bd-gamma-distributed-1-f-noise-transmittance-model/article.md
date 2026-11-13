# Gamma-distributed 1/f^β noise transmittance model

Also called: Gamma distribution transmittance

## Overview
This method computes the ensemble-averaged transmittance of participating media by modeling the line-averaged micro-scale density as a Gamma-distributed random variable. It solves the problem of non-physical negative extinction coefficients inherent in Gaussian approximations [S1]. Unlike the [Gaussian 1/f^β noise transmittance model](14555-gaussian-1-f-noise-transmittance-model.md), which allows negative values, the Gamma model ensures strictly non-negative extinction, making it suitable for heterogeneous media where the mean extinction is low relative to the variance [S1].

## How it works
**Inputs:**
*   $\mu_\sigma$: Mean extinction coefficient.
*   $C$: Amplitude of variation (noise strength).
*   $\beta$: Spectral exponent (color of noise).
*   $t$: Path length.

**Outputs:**
*   $\text{ff}_\Gamma(t \mid \psi)$: Ensemble-averaged transmittance.

**Data Structures:**
No persistent data structures are required. The method computes parameters on the fly.

**Steps:**
1.  Compute the variance of the line-averaged density, $\nu_{\overline{\sigma}\mu}(t)$, based on the Gaussian model's variance scaling (derived from the parent method's characteristic function framework) [S1].
2.  Calculate the shape parameter $\alpha(t)$ using the mean and variance:
    $$ \alpha(t) = \frac{\mu_\sigma^2}{\nu_{\overline{\sigma}\mu}(t)} $$
3.  Compute the transmittance using the characteristic function of the Gamma distribution:
    $$ \text{ff}_\Gamma(t \mid \psi) = \left(1 + \frac{\mu_\sigma t}{\alpha(t)}\right)^{-\alpha(t)} $$

**Parameters:**
*   $\mu_\sigma > 0$: Mean extinction.
*   $C \ge 0$: Amplitude.
*   $\beta \ge 0$: Spectral exponent.

**Edge Cases:**
*   **Homogeneous/White Noise:** If $C=0$ or $\beta=0$, the variance vanishes, $\alpha \to \infty$, and the model reduces to the standard exponential transmittance $e^{-\mu_\sigma t}$ [S1].
*   **Precision:** For very large $\alpha$, numerical stability of $(1 + x/\alpha)^{-\alpha}$ should be handled using `exp(-x)` or logarithmic evaluation to avoid precision loss [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gamma_transmittance(mu_sigma, C, beta, t):
    """
    Compute ensemble-averaged transmittance using the Gamma-distributed 1/f^beta noise model.
    
    Args:
        mu_sigma: Mean extinction coefficient.
        C: Amplitude of variation.
        beta: Spectral exponent.
        t: Path length.
        
    Returns:
        Transmittance value.
    """
    # Step 1: Compute variance nu_sigma_mu(t)
    # The variance scaling is derived from the Gaussian model's characteristic function.
    # From Eq 34 in [S1], the variance scales as:
    # nu_sigma_mu(t) = (C * mu_sigma * t)^(beta + 1) / (2 * t^2) ? 
    # Note: The source [S1] Eq 35 shows the Gaussian exponent term as (C*mu_sigma*t)^(beta+1)/2.
    # The variance nu is the second moment term in the exponent expansion.
    # Specifically, for the Gaussian model, the exponent is -mu*t + nu*t^2/2.
    # Eq 35: exp(-mu*t + (C*mu*t)^(beta+1)/2).
    # Thus, nu * t^2 / 2 = (C * mu * t)^(beta+1) / 2
    # nu = (C * mu * t)^(beta+1) / t^2 = C^(beta+1) * mu^(beta+1) * t^(beta-1)
    
    if t == 0:
        return 1.0
        
    # Calculate variance nu_sigma_mu(t)
    # nu = C^(beta+1) * mu^(beta+1) * t^(beta-1)
    # Using log-exp for stability if needed, but direct computation for typical values:
    nu = (C ** (beta + 1)) * (mu_sigma ** (beta + 1)) * (t ** (beta - 1))
    
    # Step 2: Calculate shape parameter alpha(t)
    # alpha = mu^2 / nu
    if nu == 0:
        # Variance is zero, alpha is infinite, returns exponential
        return math.exp(-mu_sigma * t)
        
    alpha = (mu_sigma ** 2) / nu
    
    # Step 3: Compute transmittance
    # ff = (1 + (mu * t) / alpha)^(-alpha)
    # Substitute alpha: mu*t/alpha = mu*t * nu / mu^2 = t * nu / mu
    # ff = (1 + t * nu / mu)^(-alpha)
    
    term = 1.0 + (t * nu) / mu_sigma
    
    # Use log to compute power for numerical stability
    # log_ff = -alpha * log(term)
    # ff = exp(log_ff)
    
    if term <= 0:
        # Should not happen for positive inputs, but guard against precision issues
        return 0.0
        
    log_ff = -alpha * math.log(term)
    return math.exp(log_ff)
```

## Subtopics

### Pink noise special case
A special case where the spectral exponent $\beta = 1$ (pink noise). In this scenario, the distribution of extinction values becomes independent of the path length $t$ [S1]. The shape parameter $\alpha$ becomes constant:
$$ \alpha = C^{-\beta-1} = C^{-2} $$
The transmittance reduces to the characteristic function of the static probability density function:
$$ \text{ff}_{\text{pink}}(t) = \varphi_{\bar{\sigma}_\mu}(t) = \int_0^\infty e^{-t\bar{\sigma}_\mu} \text{pdf}(\bar{\sigma}_\mu) \, d\bar{\sigma}_\mu $$
This scale-invariant property simplifies the model to a single-parameter family determined by $C$ and $\mu_\sigma$ [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per path length evaluation. Requires basic arithmetic operations and a logarithm/exponential.
*   **Memory Complexity:** $O(1)$. No storage required.
*   **Strengths:** Guarantees non-negative extinction coefficients, avoiding the non-physical artifacts of the Gaussian model [S1]. Provides a closed-form solution for non-exponential transmittance.
*   **Limitations:** More complex than the exponential model. Requires tuning of three parameters ($\mu_\sigma, C, \beta$).
*   **When to use:** When simulating heterogeneous media with significant density fluctuations where the mean extinction is not sufficiently high to make negative Gaussian samples negligible [S1].

## Background
The Gamma-distributed model was proposed by Davis and Mineev-Weinstein [2011] to address the limitations of the Gaussian approximation in the path integral framework for non-exponential media [S1]. It is part of the broader effort to model light transport in participating media with fractal $1/f^\beta$ noise density distributions.

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
