# Aggregated Microfacet Distribution via Beta Distribution

Also called: Beta Distribution Microfacet Roughness

## Overview
This method models the aggregated microfacet roughness distribution $p_A(\alpha)$ using a Beta distribution $B(\alpha; a, b)$ to estimate the statistical spread of roughness values $\alpha$ from asset data [S1, unofficial]. It is chosen over Gaussian approximations because the Beta distribution is naturally bounded within the valid $[0, 1]$ range of $\alpha$, preventing the invalid range errors that occur when Gaussian tails exceed these bounds [S1, unofficial].

## How it works
**Inputs:** A set of roughness maps or a statistical summary (mean $\mu$ and variance $\sigma^2$) of the roughness values $\alpha$ from the input assets.
**Outputs:** Shape parameters $a$ and $b$ for the Beta distribution, and the resulting probability density function $p_A(\alpha)$.

**Data Structures:**
The method operates on scalar values. The roughness $\alpha$ is a float in $[0, 1]$. The parameters $a$ and $b$ are positive floats.

**Steps:**
1.  Compute the mean $\mu$ and variance $\sigma^2$ of the roughness values $\alpha$ from the input data.
2.  Calculate the shape parameter $a$ using the formula:
    $$ a = -\frac{\mu (\sigma^2 + \mu^2 - \mu)}{\sigma^2} $$
3.  Calculate the shape parameter $b$ using the formula:
    $$ b = \frac{(\sigma^2 + \mu^2 - \mu)(\mu - 1)}{\sigma^2} $$
4.  Define the aggregated distribution as the Beta distribution with these parameters:
    $$ p_A(\alpha) = B(\alpha; a, b) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fit_beta_distribution(mean_mu, variance_sigma2):
    """
    Estimates Beta distribution shape parameters a and b from mean and variance.
    
    Args:
        mean_mu (float): The mean roughness value.
        variance_sigma2 (float): The variance of the roughness values.
        
    Returns:
        tuple: (a, b) shape parameters for the Beta distribution.
    """
    # Calculate the common term (sigma^2 + mu^2 - mu)
    term = variance_sigma2 + mean_mu**2 - mean_mu
    
    # Calculate shape parameter a
    # Note: The source formula is a = -mu * (sigma^2 + mu^2 - mu) / sigma^2
    a = -mean_mu * term / variance_sigma2
    
    # Calculate shape parameter b
    # Note: The source formula is b = (sigma^2 + mu^2 - mu) * (mu - 1) / sigma^2
    b = term * (mean_mu - 1) / variance_sigma2
    
    return a, b
```

**Edge Cases and Pitfalls:**
*   **Range Validity:** Unlike Gaussian distributions, the Beta distribution naturally respects the $[0, 1]$ range of $\alpha$. This avoids the pitfall where a Gaussian fit for roughness maps might assign non-zero probability to invalid roughness values outside $[0, 1]$ [S1, unofficial].
*   **Parameter Constraints:** The shape parameters $a$ and $b$ must be positive for a valid Beta distribution. The source formulas are derived from the moments of the Beta distribution. If the input $\mu$ and $\sigma^2$ do not correspond to a valid Beta distribution (e.g., variance is too large for the given mean), the resulting $a$ or $b$ may be negative or zero, leading to undefined behavior. The source notes that this fitting is performed on actual assets, implying the data is well-behaved [S1, unofficial].
*   **Truncated Gaussian Alternative:** A truncated Gaussian could be restricted to the valid range and may produce similar quality, but the fitting process is more tedious [S1, unofficial].

## Subtopics

### Beta Distribution Fitting for Roughness Maps
This variant involves fitting roughness maps from assets to beta distributions to estimate the distribution of alpha (roughness) [S1, unofficial]. It is preferred over Gaussian distributions because Gaussians can exceed the valid $[0, 1]$ range of alpha, whereas beta distributions are bounded [S1, unofficial]. Truncated Gaussians could be restricted to the valid range and may produce similar quality, but the fitting process is more tedious [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The fitting process involves constant-time arithmetic operations given the mean and variance. The complexity is $O(1)$ per asset or map summary. Storing the parameters $a$ and $b$ requires minimal memory (two floats).

**Measured Performance:**
The source validates the approach by estimating $p_A(\alpha)$ from roughness maps used for actual assets. The fits are described as "reasonably accurate" and adapt to different modes of real data [S1, unofficial]. The method is compared to ground-truth aggregated microfacet distributions and simple averaged roughness, showing that the fits are "overall more accurate and guaranteed to be never worse than the simple fits" [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Naturally bounded to $[0, 1]$, avoiding invalid roughness values. More accurate than simple averaged roughness.
*   **Limitations:** Requires accurate estimation of mean and variance. The source does not provide specific hardware performance metrics or resolution scaling data.

**When to Use:**
Use this method when aggregating microfacet distributions for level-of-detail rendering or when approximating the roughness distribution from asset data, especially when the roughness values are bounded in $[0, 1]$ and a Gaussian approximation would risk exceeding these bounds.

## Background
This method is part of the "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" work, which uses a Beta distribution to represent the roughness distribution $p_A(\alpha)$ [S1, unofficial]. It builds on the Aggregated Microfacet Distribution Approximation [10880-aggregated-microfacet-distribution-approximation.md].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
