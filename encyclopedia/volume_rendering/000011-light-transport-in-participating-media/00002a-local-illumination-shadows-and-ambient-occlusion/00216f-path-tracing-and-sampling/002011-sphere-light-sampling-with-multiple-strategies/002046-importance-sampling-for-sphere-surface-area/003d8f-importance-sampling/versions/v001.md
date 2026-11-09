# Importance Sampling


## Overview
Importance sampling is a variance reduction technique for Monte Carlo integration that accelerates convergence by selecting samples from a probability density function (pdf) $p(x)$ that mimics the shape of the integrand $f(x)$ [S1, unofficial]. Unlike uniform sampling, which treats all regions equally, this method concentrates computational effort on areas contributing significantly to the final result, such as directions pointing toward light sources in rendering [S1, unofficial].

## How it works
**Inputs:**
- Integrand function $f(x)$ to be integrated over domain $\Omega$.
- Number of samples $N$.

**Outputs:**
- Estimated integral value $F_N$.

**Data Structures:**
- Sample set $\{X_1, \dots, X_N\}$ drawn from pdf $p(x)$.

**Steps:**
1. Define a sampling pdf $p(x)$ such that $p(x) \approx c \cdot f(x)$ for some constant $c$.
2. Generate $N$ independent samples $X_i$ from $p(x)$.
3. Evaluate the weighted sum of the integrand at these samples.

**Formula:**
The estimator is given by:
$$ F_N = \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)} $$
where $f(X_i)$ is the value of the integrand at sample $X_i$, and $p(X_i)$ is the probability density of generating that sample [S1, unofficial].

**Parameters:**
- $N$: Number of samples. Convergence rate is $O(1/\sqrt{N})$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $p(x) = 0$ where $f(x) > 0$, the estimator becomes undefined (infinite variance). The support of $p(x)$ must cover the support of $f(x)$.
- If $p(x)$ is not normalized, the estimator is biased unless $p(x)$ is explicitly normalized during evaluation.
- In the ideal case where $p(x) = c f(x)$, the ratio $f(x)/p(x)$ is constant, resulting in zero variance [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$ evaluations of $f(x)$ and $p(x)$.
- **Memory Complexity:** $O(1)$ auxiliary storage if samples are processed on the fly, or $O(N)$ if stored.
- **Performance:** Significantly reduces variance compared to uniform sampling when $p(x)$ closely matches $f(x)$.
- **Strengths:** Accelerates convergence for high-dimensional integrals; simple to implement if a good $p(x)$ is known.
- **Limitations:** Requires deriving or approximating a pdf $p(x)$ similar to $f(x)$; poor choice of $p(x)$ can yield worse results than uniform sampling.

## Background
Originating from statistical theory, importance sampling is widely used in computer graphics to optimize Monte Carlo path tracing. It addresses the inefficiency of uniform sampling in light transport simulations where most sampled directions contribute little to the final image [S1, unofficial].

## Sources
[S1] A null-scattering path integral formulation of light transport, https://steven.schuerstedt.com/wp-content/uploads/2020/10/schuerstedt.pdf, fetched 2026-10-29 (unofficial)
