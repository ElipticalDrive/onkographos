# Non-Exponential Free Flight Model
Also called: Davis and Xu model, Generalized linear transport model for spatially correlated stochastic media

## Overview
The Non-Exponential Free Flight Model describes light transport in participating media where scattering particles are positively correlated, resulting in free path distributions with longer tails than the standard exponential distribution [S1, unofficial]. It allows for artistic control over the depth of light bleeding into subsurface media by introducing a shape parameter that modifies the probability of interaction based on distance traveled [S1, unofficial]. Unlike the Exponential Fog model, which assumes uncorrelated particle distribution, this model gives photons "memory," where past non-interactions influence future interaction probabilities [S1, unofficial].

## How it works
**Inputs:**
- $\sigma_e$: Extinction coefficient (constant for the homogeneous medium).
- $\sigma_s$: Scattering coefficient (constant for the homogeneous medium).
- $s$: Distance traveled from the current position.
- $b$: User-defined "bleed" parameter controlling the shape of the distribution.

**Outputs:**
- $T_a(s)$: Transmittance at distance $s$.
- $p(s)$: Probability density function (PDF) for sampling distance $s$.
- $c(s)$: Cumulative distribution function (CDF) for sampling distance $s$.
- $s(\xi)$: Sampled distance from a uniform random variable $\xi$.
- $S(s)$: Modified scattering coefficient for Monte Carlo random walk steps.

**Data Structures:**
No complex data structures are required. The model relies on scalar values and standard floating-point arithmetic. The parameter $a$ is derived from the user-friendly parameter $b$ as $a = 1/b$.

**Parameters:**
- $b$ (bleed): Controls the tail length of the distribution. Typical values are not explicitly given in the sources, but the mapping $a=1/b$ is designed to provide a meaningful unit range for artists [S1, unofficial].
- $a$: Shape parameter. $a = 1/b$.

**Formulas:**
The transmittance $T_a$ is defined as [S1, unofficial]:
$$ T_a(s) = \frac{1}{(1 + \frac{\sigma_e s}{a})^a} $$

The probability density function (PDF) for sampling distance $s$ is [S1, unofficial]:
$$ p(s) = \frac{\sigma_e}{(1 + \frac{\sigma_e s}{a})^{a+1}} $$

The cumulative distribution function (CDF) is [S1, unofficial]:
$$ c(s) = \frac{1}{(1 + \frac{\sigma_e s}{a})^a} $$

To sample a distance $s$ from a uniform random variable $\xi \in (0, 1]$, use the inverse CDF [S1, unofficial]:
$$ s(\xi) = \frac{a \cdot (\xi^{-1/a} - 1)}{\sigma_e} $$

In the context of a Monte Carlo random walk, the scattering coefficient $\sigma_s$ is modified to depend on the distance traveled $s$ [S1, unofficial]:
$$ S(s) = \frac{\sigma_s}{1 + \frac{\sigma_e s}{a}} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_transmittance(sigma_e, s, a):
    """
    Computes the non-exponential transmittance Ta(s).
    
    Args:
        sigma_e (float): Extinction coefficient.
        s (float): Distance traveled.
        a (float): Shape parameter (a = 1/bleed).
        
    Returns:
        float: Transmittance value.
    """
    if sigma_e == 0:
        return 1.0
    term = 1.0 + (sigma_e * s) / a
    return 1.0 / (term ** a)

def sample_free_flight(sigma_e, sigma_s, s, a, xi):
    """
    Samples a new free flight distance and computes the modified scattering coefficient.
    
    Args:
        sigma_e (float): Extinction coefficient.
        sigma_s (float): Scattering coefficient.
        s (float): Current distance traveled from the last interaction.
        a (float): Shape parameter (a = 1/bleed).
        xi (float): Uniform random number in (0, 1].
        
    Returns:
        tuple: (new_s, modified_sigma_s)
            new_s: The sampled distance to the next interaction.
            modified_sigma_s: The scattering coefficient at the new position.
    """
    # Sample distance using inverse CDF
    # Note: xi should be in (0, 1]. If xi is 0, handle as edge case (e.g., return 0 or max float)
    if xi <= 0.0:
        xi = 1e-9  # Guard against log(0) or division by zero, though inverse CDF uses power
        
    # s_new = a * (xi^(-1/a) - 1) / sigma_e
    # This is the distance to the NEXT interaction relative to the start of the step.
    # In a random walk, we usually sample the step length directly.
    # The formula s(xi) gives the distance s such that c(s) = xi.
    # Since c(s) is the probability of NO interaction up to s, 
    # sampling xi from uniform and solving for s gives the distance to the next interaction.
    
    if sigma_e == 0:
        return float('inf'), sigma_s
        
    exponent = -1.0 / a
    term = xi ** exponent
    new_s = (a * (term - 1.0)) / sigma_e
    
    # Compute modified scattering coefficient at the new position
    # The model modifies the scattering probability based on total distance traveled.
    # However, in a step-by-step random walk, 's' in the formula S(s) usually refers 
    # to the distance from the last scattering event.
    # The source states: "scatter S(s) = sigma_s / (1 + sigma_e s / a)"
    # This implies the scattering coefficient is evaluated at the current position s.
    
    if sigma_e == 0:
        modified_sigma_s = sigma_s
    else:
        modified_sigma_s = sigma_s / (1.0 + (sigma_e * new_s) / a)
        
    return new_s, modified_sigma_s

def compute_scattering_probability(sigma_e, sigma_s, s, a):
    """
    Computes the modified scattering coefficient S(s) for use in Russian Roulette or albedo calculations.
    
    Args:
        sigma_e (float): Extinction coefficient.
        sigma_s (float): Scattering coefficient.
        s (float): Distance traveled.
        a (float): Shape parameter.
        
    Returns:
        float: Modified scattering coefficient.
    """
    if sigma_e == 0:
        return sigma_s
    return sigma_s / (1.0 + (sigma_e * s) / a)
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $\sigma_e = 0$, the medium is transparent. The formulas involve division by $\sigma_e$ or terms like $1 + \sigma_e s/a$. If $\sigma_e = 0$, transmittance is 1, and no scattering occurs. The pseudocode handles this with guards.
- **Parameter Range:** The parameter $a$ can result in unintuitive ranges for users. The model maps it to 'bleed' ($b$) where $a = 1/b$ to provide a meaningful unit range [S1, unofficial]. Users should input $b$ and compute $a$.
- **Random Variable Range:** The inverse CDF formula $s(\xi) = \frac{a \cdot (\xi^{-1/a} - 1)}{\sigma_e}$ requires $\xi \in (0, 1]$. If $\xi = 0$, the term $\xi^{-1/a}$ becomes infinite. In practice, $\xi$ should be sampled from $(0, 1]$ or clamped to a small positive value [G].
- **Memory Effect:** The probability of scattering depends on both the scattering cross section and the distance traveled. This "memory" effect means that photons that have traveled further without interacting have a lower probability of scattering in the immediate future compared to the exponential case [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The computational cost per step is similar to the exponential model, involving a few floating-point operations (power, division, multiplication). The inverse CDF sampling requires a power operation, which is slightly more expensive than the logarithm used in exponential sampling, but still constant time $O(1)$ per step.
- **Memory Complexity:** $O(1)$ per particle/step. No additional data structures are required.
- **Strengths:** Provides artistic control over light bleeding depth. The long-tail distribution allows a small fraction of photons to travel much deeper than an exponential distribution would allow, which can be useful for simulating subsurface scattering in materials like skin [S1, unofficial].
- **Limitations:** The model assumes a homogeneous medium with positively correlated particle distribution. It may not accurately represent all types of participating media. The parameter $a$ (or $b$) requires tuning to achieve desired visual results.
- **When to use:** When simulating subsurface scattering in materials where light bleeding depth is a key artistic control, and the medium can be approximated as having positively correlated scattering particles.
- **When not to use:** For media with uncorrelated particle distributions (use Exponential Fog) or heterogeneous media (use Heterogeneous Medium Transmittance).

## Background
The Non-Exponential Free Flight Model was developed by Anthony B. Davis and Feng Xu in 2014, published in "A generalized linear transport model for spatially correlated stochastic media" in the Journal of Computational and Theoretical Transport [S1, unofficial]. It was later adopted and popularized in the graphics community by Pixar for path-traced subsurface scattering, as described in "Path Traced Subsurface Scattering Model" [S1, unofficial]. The model addresses the limitation of Beer's Law, which assumes uncorrelated particle distribution, by introducing a shape parameter that allows for non-exponential free path distributions. This is particularly useful for simulating the visual appearance of subsurface scattering in materials like skin, where light bleeding depth is a critical factor.

## Sources
[S1] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
