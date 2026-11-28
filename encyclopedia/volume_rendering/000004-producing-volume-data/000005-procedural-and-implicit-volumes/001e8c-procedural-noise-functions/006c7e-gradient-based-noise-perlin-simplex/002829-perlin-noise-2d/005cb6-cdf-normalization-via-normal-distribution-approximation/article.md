# CDF Normalization via Normal Distribution Approximation

Also called: Normal Distribution Approximation for Perlin Noise, Perlin Noise CDF Normalization

## Overview
This method compensates for mean shifts introduced by non-linear transformations on Perlin noise by approximating the noise distribution as a normal distribution, mapping it to a uniform distribution via the Cumulative Distribution Function (CDF), applying the desired transformation, and normalizing the result [S1, unofficial].

## How it works
**Inputs:** A scalar Perlin noise value $x$ and a transformation function $f(x)$ [S1, unofficial].
**Outputs:** A normalized transformed noise value $y$ [S1, unofficial].

**Parameters:**
*   $\mu = 0$: The mean of the Perlin noise distribution [S1, unofficial].
*   $\sigma \approx 0.1995$: The standard deviation of the Perlin noise distribution, estimated by random sampling [S1, unofficial].

**Steps:**
1.  **Estimate Parameters:** Use $\mu = 0$ and $\sigma \approx 0.1995$ [S1, unofficial].
2.  **Map to Uniform Distribution:** Apply the CDF of the normal distribution $F(x)$ to the input noise $x$ to obtain a value $u$ in the range $[0, 1]$ [S1, unofficial].
    $$ u = F(x) = 0.5 \left( 1 + \text{erf}\left( \frac{x - \mu}{\sigma \sqrt{2}} \right) \right) $$
    Where $\text{erf}(z)$ is the error function:
    $$ \text{erf}(z) = \frac{2}{\sqrt{\pi}} \int_0^z e^{-t^2} dt $$
3.  **Apply Transformation:** Compute the transformed value $v$ using the desired function $f$ on the uniform value $u$:
    $$ v = f(u) $$
4.  **Normalize:** Calculate the normalization constant $C$, which is the integral of $f(x)$ over the range $[0, 1]$:
    $$ C = \int_0^1 f(x) dx $$
    The final output $y$ is:
    $$ y = \frac{v}{C} = \frac{f(u)}{\int_0^1 f(x) dx} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def erf_func(z):
    return math.erf(z)

def cdf_normal(x, mu, sigma):
    z = (x - mu) / (sigma * math.sqrt(2))
    return 0.5 * (1 + erf_func(z))

def normalize_transformed_noise(x, f, integral_f):
    mu = 0.0
    sigma = 0.1995
    u = cdf_normal(x, mu, sigma)
    v = f(u)
    if integral_f == 0:
        return 0.0  # Guard against division by zero [G]
    y = v / integral_f
    return y
```

**Edge Cases and Pitfalls:**
*   **Negative Values:** The function $f(x)$ must be positive on the range $[0, 1]$ because negative values are invalid for a density field [S1, unofficial].
*   **Integrability:** The integral $\int_0^1 f(x) dx$ must be computable analytically [S1, unofficial].
*   **Approximation Error:** Perlin noise is bounded, whereas the normal distribution is not, making the approximation imperfect [S1, unofficial].

## Subtopics

### [Symmetric Function Transformation](17210-symmetric-function-transformation.md)
A method to modify Perlin noise octaves while preserving the mean value by using functions that satisfy a specific symmetry condition around zero. It sets apart from alternatives by enforcing a mathematical constraint on the transformation function itself, rather than compensating for the mean shift after the fact. This ensures the resulting distribution has a predictable mean without requiring post-hoc normalization or pre-computed sampling.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance and Scaling:** Not given in the sources.
*   **Strengths and Limitations:** The method requires the integral of the transformation function to be known analytically [S1, unofficial]. It is designed for density fields where $f(x)$ must remain positive on $[0, 1]$ [S1, unofficial]. The approximation assumes Perlin noise follows a normal distribution, ignoring its bounded nature [S1, unofficial].
*   **When to use / When not to use:** Not given in the sources.

## Background
This method originates from research into procedural multiplicative noise models for dust cloud generation [S1, unofficial]. It addresses the issue of mean shift when applying non-linear functions to Perlin noise, such as $1 - |x|$ [S1, unofficial]. The approach leverages the property that applying the CDF of a distribution to a random variable from that distribution yields a uniform distribution [S1, unofficial]. By approximating Perlin noise with a normal distribution, the method enables easier analytical compensation of mean shifts [S1, unofficial]. The standard deviation $\sigma \approx 0.1995$ was determined by sampling Perlin noise values [S1, unofficial].

## Sources
[S1] Procedural Multiplicative Perlin Noise Model for Dust Cloud Generation, https://inria.hal.science/hal-05627012v1, fetched 2026-10-26 (unofficial)
