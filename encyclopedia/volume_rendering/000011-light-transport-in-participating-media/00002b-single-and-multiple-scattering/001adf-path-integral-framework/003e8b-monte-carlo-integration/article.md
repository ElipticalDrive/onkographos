# Monte Carlo Integration
Also called: Monte Carlo estimation, sample averaging estimator

## Overview
Monte Carlo Integration is a numerical technique for estimating definite integrals by averaging function evaluations at random points weighted by their probability density function (pdf) [S1, unofficial]. It solves the problem of computing high-dimensional integrals that lack closed-form analytical solutions, such as those arising in light transport simulation [S1, unofficial]. It sets itself apart from deterministic quadrature methods by maintaining a convergence rate of $O(1/\sqrt{N})$ regardless of the integral's dimension or the smoothness of the integrand, making it the standard approach for rendering equations [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A function $f(x)$ to integrate over a domain, a probability density function $p(x)$ from which samples are drawn, and an integer $N$ representing the number of samples.
- Output: A scalar estimate $F_N$ of the integral $\int f(x) dx$.

**Data Structures:**
- No complex data structures are required. The method relies on a sequence of $N$ independent random samples $X_i$ and the corresponding function evaluations.

**Numbered Steps:**
1. Generate $N$ independent random samples $X_1, X_2, \dots, X_N$ from the domain of integration according to the probability density function $p(x)$.
2. For each sample $X_i$, evaluate the function $f(X_i)$.
3. Compute the importance weight for each sample as $w_i = \frac{1}{p(X_i)}$.
4. Calculate the estimator $F_N$ by averaging the weighted function values:
   $$ F_N = \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)} $$

**Formulas:**
The integral is expressed as the expected value of the ratio of the integrand to the pdf:
$$ \int f(x) dx = E \left[ \frac{f(x)}{p(x)} \right] = \int \frac{f(x)}{p(x)} p(x) dx $$
The Monte Carlo approximation of this expected value is:
$$ F_N = \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)} $$
Where:
- $F_N$ is the random variable representing the integral estimate.
- $N$ is the number of samples.
- $f(X_i)$ is the value of the integrand at sample $i$.
- $p(X_i)$ is the probability density of generating sample $X_i$.

**Parameters:**
- $N$: The number of samples. The estimator $F_N$ is a random variable whose variance decreases as $N$ increases. Typical values depend on the required noise level and computational budget, often ranging from hundreds to millions in rendering contexts.

**Edge Cases and Pitfalls:**
- **Zero Probability Density:** If $p(x) = 0$ for any $x$ where $f(x) \neq 0$, the weight becomes infinite, causing undefined behavior. The support of $p(x)$ must cover the support of $f(x)$ [S1, unofficial].
- **High Variance:** If $p(x)$ is poorly matched to $f(x)$ (e.g., $f(x)$ has sharp peaks not covered by $p(x)$), the variance of the estimator increases, requiring more samples for convergence [S1, unofficial].
- **Precision:** Summing many floating-point numbers can lead to precision loss. Kahan summation or pairwise summation is recommended for large $N$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_integration(f, sample_generator, pdf, N):
    """
    Estimates the integral of f(x) using Monte Carlo integration.
    
    Args:
        f: Function to integrate.
        sample_generator: Function that returns a random sample x from the domain.
        pdf: Function that returns the probability density p(x) for a given x.
        N: Number of samples.
        
    Returns:
        The estimated integral value.
    """
    total = 0.0
    for i in range(N):
        # Generate a random sample
        x = sample_generator()
        
        # Evaluate the function and the pdf
        fx = f(x)
        px = pdf(x)
        
        # Guard against division by zero
        if px == 0:
            raise ValueError("Probability density is zero at sample point.")
            
        # Accumulate the weighted value
        total += fx / px
        
    # Return the average
    return total / N
```

## Subtopics

### [Monte Carlo Estimator of Edge Integrals](10285-monte-carlo-estimator-of-edge-integrals.md)
Estimates edge integrals from radiance discontinuities at geometric boundaries. Chosen when standard path tracing misses contributions from sharp edges or surface boundaries.

### [Residual Color Estimation Scheme](10577-residual-color-estimation-scheme.md)
Reduces bias in Fourier coefficient calculations due to nonuniform sampling. Chosen when spectral integration requires correction for DC component bias.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of samples. Each sample requires one evaluation of $f(x)$ and one evaluation of $p(x)$.
- **Memory Complexity:** $O(1)$ auxiliary space, excluding storage for the samples if processed in a streaming fashion.
- **Convergence Rate:** $O(1/\sqrt{N})$ in any dimension [S1, unofficial]. This rate is independent of the dimensionality of the integral and the smoothness of the function, which is a significant advantage over deterministic methods that suffer from the curse of dimensionality.
- **Strengths:** Dimensionality independence, simplicity of implementation, and unbiased nature (assuming correct sampling).
- **Limitations:** Slow convergence rate compared to deterministic methods in low dimensions for smooth functions; high variance if importance sampling is not used effectively.
- **When to use:** High-dimensional integrals, complex domains, or when the integrand is not smooth or has discontinuities.
- **When not to use:** Low-dimensional integrals with smooth functions where deterministic quadrature offers faster convergence.

## Background
Monte Carlo integration was introduced in the context of rendering by Kajiya (1986) in the derivation of the path integral formulation of the rendering equation [S1, unofficial]. It has since become a foundational technique in computer graphics for solving the light transport equation, particularly in participating media where multiple scattering leads to high-dimensional integrals. The method's robustness in high dimensions makes it suitable for the path integral framework, which combines photon density estimation and Monte Carlo path integration.

## Sources
[S1] PDF A null-scattering path integral formulation of light transport, https://steven.schuerstedt.com/wp-content/uploads/2020/10/schuerstedt.pdf, fetched 2026-10-29 (unofficial)
