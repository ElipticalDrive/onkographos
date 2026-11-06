# Weighted Importance Sampling


## Overview
Weighted Importance Sampling (WIS) is a biased but consistent technique used to simulate the use of a known target probability density function (PDF) $p$ with samples drawn from an arbitrary source PDF $q$ [S1]. It solves the problem where the ideal PDF for an integrand (the target) is unknown or difficult to sample from directly, but a suboptimal source PDF is available. It sets itself apart from standard Importance Sampling by using self-normalizing weights that prevent poor samples (low $q$ but high $p$) from inflating variance, thereby escaping Veach's theoretical bounds for optimal weights when $p$ is well-chosen [S1].

## How it works
**Inputs and Outputs:**
- Input: A set of $N$ samples $\{x_i\}_{i=1}^N$ drawn from a source PDF $q(x)$.
- Input: A target PDF $p(x)$ which is known and desirable for the integrand.
- Output: An estimator for the integral $\int f(x) dx$.

**Data Structures:**
- Samples $x_i$: Points in the domain of integration.
- Weights $w_i$: Scalar values associated with each sample.

**Method:**
The core idea is to re-weight samples drawn from $q$ to behave as if they were drawn from $p$. The estimator for a function $f(x)$ is given by:

$$
\hat{I} = \frac{\sum_{i=1}^N w_i f(x_i)}{\sum_{i=1}^N w_i}
$$

where the weight $w_i$ for sample $x_i$ is defined as the ratio of the target PDF to the source PDF:

$$
w_i = \frac{p(x_i)}{q(x_i)}
$$

**Parameters:**
- $N$: Number of samples. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def weighted_importance_sampling(samples, target_pdf, source_pdf, integrand):
    """
    Estimates the integral using Weighted Importance Sampling.
    
    Args:
        samples: List of N samples drawn from source_pdf.
        target_pdf: Function p(x) representing the ideal PDF.
        source_pdf: Function q(x) from which samples were drawn.
        integrand: Function f(x) to integrate.
        
    Returns:
        Estimated integral value.
    """
    numerator = 0.0
    denominator = 0.0
    
    for x in samples:
        # Calculate importance weight
        # Note: If source_pdf(x) is 0, this is undefined. 
        # In practice, samples are drawn from q, so q(x) > 0. [G]
        weight = target_pdf(x) / source_pdf(x)
        
        numerator += weight * integrand(x)
        denominator += weight
        
    # Avoid division by zero if all weights are zero
    if denominator == 0:
        return 0.0
        
    return numerator / denominator
```

**Edge Cases and Pitfalls:**
- **Zero Source PDF:** If $q(x_i) = 0$ for any sample $x_i$, the weight is undefined. However, since samples are drawn from $q$, this should not occur in valid implementations [S1].
- **Bias:** The estimator is biased for finite $N$ due to the normalization by the sum of weights, but it is consistent (converges to the true value as $N \to \infty$) [S1].
- **Variance:** By using self-normalizing weights, WIS prevents samples with low $q$ but high $p$ from dominating the variance, which is a key advantage over standard importance sampling in this context [S1].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of samples. Each sample requires evaluation of $p(x)$, $q(x)$, and $f(x)$.
- **Memory Complexity:** $O(N)$ to store samples and weights.
- **Strengths:** Reduces variance compared to standard importance sampling when the target PDF is a better match for the integrand than the source PDF. It is consistent.
- **Limitations:** It is biased for finite samples. It requires evaluating both $p(x)$ and $q(x)$ for every sample.
- **When to use:** When you have samples from a source distribution $q$ but know a target distribution $p$ that is more suitable for the integrand, and you cannot resample from $p$ directly.
- **When not to use:** When unbiasedness is strictly required for finite samples, or when resampling is feasible (in which case Resampled Importance Sampling (RIS) may be preferred) [S1].

## Background
Weighted Importance Sampling was proposed as a technique to simulate the use of a known target PDF $p$ with samples drawn from an arbitrary source PDF $q$ [S1]. It has been used in rendering applications [S1]. Bitterli et al. showed that Resampled Importance Sampling (RIS) is a bias-corrected version of WIS [S1]. However, RIS requires resampling prefix candidates, which is intractable for unbounded paths in volumetric rendering, making WIS a practical alternative in such contexts [S1].

## Sources
[S1] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
