# Parametric Transmittance Sampling

Also called: Parametric transmittance curves, Statistical distributions for transmittance

## Overview
Parametric Transmittance Sampling is a Monte Carlo technique for estimating the Volume Rendering Equation (VRE) by importance sampling the medium extinction transmittance using non-exponential statistical distributions [S1]. It extends standard distance sampling by allowing the free-flight distance to be drawn from parametric families (e.g., Gamma, Uniform) rather than strictly the exponential distribution, enabling more efficient sampling in media with non-constant or complex extinction profiles [S1]. It differs from [Whole Volume Distance Sampling](9002-whole-volume-distance-sampling.md) by using local parametric approximations rather than global volume statistics, and from [Closed-form Tracking](9223-closed-form-tracking.md) by relying on statistical sampling of parametric curves rather than analytical integration of specific density functions [S1].

## How it works
**Inputs:**
- Current position $x_0$ in the medium.
- Direction of travel $\omega$.
- A parametric distribution family $\mathcal{D}$ defined by parameters $\theta$ (e.g., mean $\mu$, shape $\alpha$).
- A random number generator producing uniform variates $\xi \in [0, 1)$.

**Outputs:**
- A sampled free-flight distance $d$ drawn from the parametric distribution.
- The transmittance $T(x_0, x_0+d)$ evaluated along the sampled distance.

**Data Structures:**
The method requires storing the parameters of the chosen distribution. For a Gamma distribution $\Gamma(\mu, \alpha)$, these are the mean $\mu \ge 0$ and shape $\alpha > 0$ [S1]. For a DeltaSum distribution, it stores a list of impulses $\{\mu_1, \dots, \mu_n\}$ [S1].

**Steps:**
1. **Select Distribution:** Choose a parametric distribution family $\mathcal{D}$ appropriate for the local extinction behavior. Common choices include the Delta distribution $\delta_\mu$, DeltaSum III(n), Uniform $\mathcal{U}(a, b)$, Linear $\mathcal{L}(b)$, Normal $\mathcal{N}(\mu, \nu)$, or Gamma $\Gamma(\mu, \alpha)$ [S1].
2. **Parameterize:** Determine the parameters $\theta$ for $\mathcal{D}$ based on the local extinction coefficient $\sigma_t$. For example, if approximating the exponential distribution, the mean $\mu$ might be set to $1/\sigma_t$ [S1].
3. **Sample Distance:** Generate a random variate $\xi$ and compute the sampled distance $d$ using the inverse CDF or specific sampling algorithm for $\mathcal{D}$ [S1].
4. **Evaluate Transmittance:** Calculate the transmittance $T$ from $x_0$ to $x_0+d$. If the distribution is an approximation of the exponential, the transmittance is often evaluated using the standard exponential formula $e^{-\sigma_t d}$ or the specific parametric form if defined [S1].
5. **Estimate Contribution:** Use the sampled distance and transmittance to estimate the radiance contribution, weighting by the ratio of the true PDF to the sampling PDF if the distribution is not the exact exponential [S1].

**Formulas:**
- **Delta Distribution $\delta_\mu$:**
  - PDF: $p(x) = \delta(x - \mu)$
  - CDF: $H(x - \mu)$
  - Sampling: $x_\delta = \mu$ [S1]
  - Transmittance weight: $-\frac{\ln \xi}{x_\delta}$ [S1]

- **DeltaSum III(n):**
  - PDF: $p(x) = \frac{1}{n} \sum_{i=1}^n \delta(x - \mu_i)$
  - CDF: $F(x) = \frac{1}{n} \sum_{i=1}^n H(x - \mu_i)$
  - Sampling: $x_{\text{III}} = \text{uniform from } \{\mu_1, \dots, \mu_n\}$ [S1]
  - Transmittance weight: $-\frac{\ln \xi}{x_{\text{III}}}$ [S1]

- **Uniform Distribution $\mathcal{U}(a, b)$:**
  - PDF: $p(x) = \frac{1}{b-a}$ for $a < x < b$
  - CDF: $F(x) = \frac{x-a}{b-a}$
  - Sampling: $x_u = (b-a)\xi + a$ [S1]

- **Linear Distribution $\mathcal{L}(b)$:**
  - PDF: $p(x) = \frac{2}{b} (1 - \frac{x}{b})$ for $0 < x < b$
  - CDF: $F(x) = 1 - 2\frac{x}{b} + \frac{x^2}{b^2}$
  - Sampling: $x_l = b(1 - \sqrt{\xi})$ [S1]

- **Normal Distribution $\mathcal{N}(\mu, \nu)$:**
  - PDF: $p(x) = (2\pi)^{-\frac{1}{2}} e^{-\frac{\nu^2 - \mu^2}{2\nu}}$ (Note: Source formula appears to have typos; standard normal PDF is $(2\pi\sigma^2)^{-1/2}e^{-(x-\mu)^2/2\sigma^2}$) [S1]
  - Sampling: $x_n = \sqrt{-2\ln \xi_1} \cos(2\pi \xi_2)$ (Box-Muller transform) [S1]

- **Gamma Distribution $\Gamma(\mu, \alpha)$:**
  - PDF: $p(x) = \frac{1}{\Gamma(\alpha)} (\frac{\alpha}{\mu})^{\alpha} x^{\alpha-1} e^{-x\alpha/\mu}$
  - Parameters: Mean $\mu \ge 0$, Shape $\alpha > 0$ [S1]
  - Sampling: Marsaglia's method [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_parametric_transmittance(dist_type, params, xi):
    """
    Sample a distance from a parametric distribution.
    
    Args:
        dist_type: String identifying the distribution ('delta', 'delta_sum', 'uniform', 'linear', 'gamma').
        params: Dictionary of parameters.
        xi: Uniform random variate in [0, 1).
        
    Returns:
        Sampled distance d.
    """
    if dist_type == 'delta':
        mu = params['mu']
        return mu
    
    elif dist_type == 'delta_sum':
        impulses = params['impulses']
        n = len(impulses)
        # Uniformly select one of the impulses
        idx = random.randint(0, n - 1)
        return impulses[idx]
    
    elif dist_type == 'uniform':
        a = params['a']
        b = params['b']
        return (b - a) * xi + a
    
    elif dist_type == 'linear':
        b = params['b']
        return b * (1 - math.sqrt(xi))
    
    elif dist_type == 'gamma':
        mu = params['mu']
        alpha = params['alpha']
        # Marsaglia's method for Gamma distribution
        # Note: This is a simplified implementation. Full Marsaglia method requires
        # handling cases where alpha < 1 and alpha >= 1 differently.
        if alpha < 1:
            # Use Cheng's method or similar for alpha < 1
            # Placeholder for full Marsaglia implementation
            pass
        else:
            # Standard Marsaglia for alpha >= 1
            d = alpha - 1.0/3.0
            c = 1.0 / math.sqrt(9.0 * d)
            while True:
                x = random.gauss(0, 1)
                v = 1.0 + c * x
                while v <= 0.0:
                    v = 1.0 + c * random.gauss(0, 1)
                v = v * v * v
                u = random.random()
                if u < 1.0 - 0.0331 * (x * x) * (x * x):
                    return d * v * mu / alpha
                if math.log(u) < 0.5 * x * x + d * (1.0 - v + math.log(v)):
                    return d * v * mu / alpha
    else:
        raise ValueError("Unknown distribution type")

def evaluate_transmittance(distance, sigma_t):
    """
    Evaluate transmittance for exponential medium.
    """
    return math.exp(-sigma_t * distance)
```

**Edge Cases and Pitfalls:**
- **Precision:** For Gamma distribution sampling, numerical stability is crucial when $\alpha$ is small or large [S1].
- **Boundaries:** Uniform distribution requires $a < b$ [S1].
- **Degenerate Input:** Delta distribution has zero variance, which may not be suitable for all Monte Carlo estimators [S1].

## Subtopics

### [Whole Volume Distance Sampling](9002-whole-volume-distance-sampling.md)
Uses global volume statistics for sampling. Choose when the entire volume has uniform or predictable density.

### [Closed-form Tracking](9223-closed-form-tracking.md)
Uses analytical solutions for specific density functions. Choose when the extinction function allows closed-form integration.

### [Rejection Sampling for Arbitrary Media](9576-rejection-sampling-for-arbitrary-media.md)
Uses rejection sampling for complex, non-parametric media. Choose when parametric approximations are insufficient.

## Operating characteristics
- **Time Complexity:** Sampling from parametric distributions is generally $O(1)$ for simple distributions like Delta and Uniform, but may be $O(1)$ on average for Gamma using Marsaglia's method [S1].
- **Memory Complexity:** $O(n)$ for DeltaSum where $n$ is the number of impulses [S1].
- **Strengths:** Allows flexible modeling of non-exponential media; can reduce variance in specific scenarios where the parametric distribution matches the local extinction behavior [S1].
- **Limitations:** Requires careful selection of the distribution family and parameters; may introduce bias if the parametric approximation is poor [S1].
- **When to Use:** When the medium's extinction profile can be well-approximated by a parametric distribution.
- **When Not to Use:** When the medium is highly heterogeneous and no simple parametric form fits well.

## Background
Parametric Transmittance Sampling was introduced in the paper "A radiative transfer framework for non-exponential media" [S1]. It extends the standard radiative transfer equation to handle media where the exponential distribution assumption does not hold, providing a framework for importance sampling using various statistical distributions [S1].

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
