# Monte Carlo Integration
Also called: Monte Carlo estimation, sample averaging estimator

## Overview
Monte Carlo integration is a numerical technique for estimating definite integrals by averaging function evaluations at random points weighted by their probability density function (pdf) [S1, unofficial]. It solves the problem of computing high-dimensional or analytically intractable integrals, such as those arising in the path integral formulation of light transport, where closed-form solutions for incident radiance are unavailable [S1, unofficial]. It sets itself apart from deterministic quadrature methods by its independence from the integral's dimensionality, making it the standard approach for rendering equations in complex scenes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A function $f(x)$ to integrate over a domain, a probability density function $p(x)$ for sampling, and an integer $N$ representing the number of samples.
- Output: A scalar estimate $F_N$ of the integral $\int f(x) dx$.

**Data Structures:**
- No complex data structures are required for the basic estimator. The implementation requires storage for $N$ random samples $X_i$ and their corresponding function evaluations.

**Numbered Steps:**
1. Generate $N$ independent random samples $X_1, X_2, \dots, X_N$ from the domain according to the probability density function $p(x)$.
2. For each sample $X_i$, evaluate the function $f(X_i)$.
3. Evaluate the probability density $p(X_i)$ at each sample point.
4. Compute the weighted contribution for each sample as $\frac{f(X_i)}{p(X_i)}$.
5. Sum these contributions and divide by $N$ to obtain the final estimate.

**Formulas:**
The integral of a function $f(x)$ is estimated as the expected value of the ratio $f(x)/p(x)$:
$$ \int f(x) dx = E \left[ \frac{f(x)}{p(x)} \right] = \int \frac{f(x)}{p(x)} p(x) dx \approx \frac{1}{N} \sum_{i=1}^N \frac{f(X_i)}{p(X_i)} \quad [S1, unofficial] $$

The estimator $F_N$ is defined as:
$$ F_N = \frac{1}{N} \sum_{i=1}^N \frac{f(X_i)}{p(X_i)} \quad [S1, unofficial] $$

Where:
- $F_N$ is the random variable representing the estimate, whose values depend on the number of samples $N$ [S1, unofficial].
- $N$ is the number of samples taken [S1, unofficial].
- $X_i$ are the random samples drawn from $p(x)$ [S1, unofficial].
- $p(x)$ is the probability density function used for sampling [S1, unofficial].

**Parameters:**
- $N$: The number of samples. Typical values depend on the desired variance reduction and computational budget. The convergence rate is $O(1/\sqrt{N})$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $p(x) = 0$ where $f(x) \neq 0$, the estimator is undefined (division by zero). The sampling distribution must have support over the entire domain where $f(x)$ is non-zero.
- High variance in $f(x)/p(x)$ leads to noisy estimates, requiring larger $N$ for convergence.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_integration(f, p, sample_generator, N):
    """
    Estimates the integral of f(x) using Monte Carlo integration.
    
    Args:
        f: Function to integrate.
        p: Probability density function (pdf) for sampling.
        sample_generator: Function that returns a random sample X drawn from p.
        N: Number of samples.
        
    Returns:
        The estimated integral value.
    """
    total = 0.0
    for i in range(N):
        # Generate a random sample X_i from the distribution p
        X_i = sample_generator()
        
        # Evaluate the function at the sample point
        f_val = f(X_i)
        
        # Evaluate the pdf at the sample point
        p_val = p(X_i)
        
        # Guard against division by zero
        if p_val == 0:
            raise ValueError("PDF is zero at sample point, estimator undefined.")
            
        # Accumulate the weighted contribution
        total += f_val / p_val
        
    # Average the contributions
    estimate = total / N
    return estimate
```

## Subtopics

### [Monte Carlo Estimator of Edge Integrals](10285-monte-carlo-estimator-of-edge-integrals.md)
Estimates edge integrals from radiance discontinuities across boundary curves in participating media. It distinguishes itself by explicitly handling geometric discontinuities (surface boundaries/sharp edges) that standard volumetric path tracing misses, ensuring unbiased estimation of radiance jumps[10299-monte-carlo-estimator-of-radiance-change.md].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of samples. Each sample requires one evaluation of $f(x)$ and one evaluation of $p(x)$.
- **Memory Complexity:** $O(1)$ auxiliary space if samples are processed sequentially, or $O(N)$ if storing all samples.
- **Convergence:** The rate of convergence is $O(1/\sqrt{N})$ in any dimension, regardless of the smoothness of the function $f(x)$ [S1, unofficial]. This slow convergence is a primary limitation, often requiring many samples to reduce variance.
- **Strengths:** Dimensionality independence; simplicity of implementation; unbiased estimation (assuming correct sampling).
- **Limitations:** Slow convergence rate; sensitive to the choice of pdf $p(x)$ (importance sampling is often needed for efficiency); high variance if $p(x)$ does not match $f(x)$.
- **When to use:** High-dimensional integrals, complex geometries, or when analytical solutions are impossible.
- **When not to use:** Low-dimensional integrals where deterministic quadrature is faster and more accurate; real-time applications with strict latency constraints unless $N$ is very small.

## Background
Monte Carlo integration is a fundamental technique in computer graphics, particularly for solving the rendering equation. Its application to light transport was formalized in the path integral framework, which derives the rendering equation (Kajiya 1986) and evaluates it using Monte Carlo methods [S1, unofficial]. The method's advantage of operating independently of the integral's dimension makes it essential for volume rendering processes, including absorption, scattering, and emission [S1, unofficial].

## Sources
[S1] A null-scattering path integral formulation of light transport, https://steven.schuerstedt.com/wp-content/uploads/2020/10/schuerstedt.pdf, fetched 2026-10-29 (unofficial)
