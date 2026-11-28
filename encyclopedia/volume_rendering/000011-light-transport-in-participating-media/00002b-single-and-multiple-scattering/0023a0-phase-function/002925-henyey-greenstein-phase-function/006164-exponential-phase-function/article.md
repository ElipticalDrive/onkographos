# Exponential Phase Function
Also called: Log-polynomial phase function

## Overview
The Exponential Phase Function is a parametric model for light scattering in participating media that guarantees non-negative values by construction, solving the instability issues of standard polynomial phase functions in inverse rendering [S1, unofficial]. It differs from its parent, the Henyey-Greenstein Phase Function, by using a log-polynomial basis rather than a single asymmetry parameter, allowing it to approximate complex distributions like the Rayleigh or Mie phase functions with higher fidelity [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Scattering angle cosine $\mu = \cos \theta$, where $\theta$ is the angle between the incident and scattered direction.
*   **Input:** Polynomial coefficients $b_0, b_1, \dots, b_M$.
*   **Input:** Basis functions $Q_i(\mu)$.
*   **Output:** Phase function value $p(\mu)$, representing the probability density of scattering into direction $\mu$.

**Data Structures**
The method requires storing $M+1$ coefficients $b_i$. The coefficient $b_0$ is not a free parameter; it is computed to satisfy the normalization constraint. The remaining $M$ coefficients ($b_1 \dots b_M$) are the free parameters.

**Basis Functions**
The paper defines the general form using a polynomial basis $Q_i(\mu)$. While the specific polynomial family (e.g., Legendre, Chebyshev) is not explicitly named in the provided text, the structure relies on $Q_i(\mu)$ being polynomials of degree $i$ in $\mu$ [S1, unofficial].

**Formulas**
The phase function is defined as:
$$
p(\mu) = \exp\left( \sum_{i=0}^{M} b_i Q_i(\mu) \right)
$$

**Normalization Constraint**
The phase function must integrate to $2\pi$ over the sphere (or 1 if normalized as a PDF over solid angle, but the source uses the $2\pi$ convention for the integral over $\mu \in [-1, 1]$ assuming azimuthal symmetry integration factor $2\pi$ is handled or implied in the density definition relative to $\mu$). The source states:
$$
\int_{-1}^{1} p(\mu) d\mu = 2\pi
$$
Note: Standard phase functions $p(\Omega)$ integrate to $4\pi$. The source equation (2) implies $p(\mu)$ here is the azimuthally integrated phase function or the density with respect to $\mu$ such that $\int p(\mu) d\mu = 2\pi$ [S1, unofficial]. To ensure the total probability is 1 over the sphere, the standard relationship is $\int_{-1}^1 p(\mu) \frac{1}{2} d\mu = 1$ if $p(\mu)$ is the azimuthally symmetric PDF. The source explicitly writes $\int_{-1}^1 p(\mu) d\mu = 2\pi$. We follow the source's normalization constraint exactly [S1, unofficial].

The coefficient $b_0$ is fixed by this constraint:
$$
b_0 = \ln\left( \frac{2\pi}{\int_{-1}^{1} \exp\left( \sum_{i=1}^{M} b_i Q_i(\mu) \right) d\mu} \right)
$$
Thus, $b_0$ is computed after $b_1 \dots b_M$ are chosen [S1, unofficial].

**Special Cases**
1.  **Zero-degree ($M=0$):**
    Set $b_i = 0$ for $i > 0$. The function becomes constant, representing an isotropic phase function [S1, unofficial].
2.  **One-degree ($M=1$):**
    Set $b_i = 0$ for $i > 1$ and $b_1 \neq 0$. This corresponds to the von Mises-Fisher (vMF) distribution:
    $$
    p_{vMF}(\mu) = \frac{\kappa \exp(\kappa \mu)}{2\pi \sinh(\kappa)}
    $$
    where $\kappa$ is the shape parameter. This can be reformulated as:
    $$
    p_{vMF} = \exp(\alpha + \kappa \mu)
    $$
    where $\alpha$ is set so that $\exp(\alpha) = \frac{\kappa}{2\pi \sinh(\kappa)}$ [S1, unofficial].
3.  **Two-degree ($M=2$):**
    Set $b_i = 0$ for $i > 2$, with $b_1 \neq 0$ and $b_2 \neq 0$. This resembles the Fisher-Bingham distribution [S1, unofficial].
    *   **Rayleigh Fit:** A two-degree exponential phase function fits the Rayleigh phase function with parameters $(b_1, b_2) = (0, 0.68)$, where $b_0$ is fixed by normalization [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exponential_phase_function(mu, b, Q_func):
    """
    Computes the Exponential Phase Function value.
    
    Args:
        mu (float): Cosine of the scattering angle, in [-1, 1].
        b (list of float): Coefficients [b_0, b_1, ..., b_M].
                           b_0 is the normalization constant.
        Q_func (callable): Function Q(i, mu) returning the i-th basis polynomial value.
        
    Returns:
        float: The phase function value p(mu).
    """
    # Calculate the sum of b_i * Q_i(mu) for i > 0
    # b[0] is the normalization constant, added at the end or included in the sum.
    # The formula is exp(sum(b_i * Q_i(mu))).
    
    exponent = 0.0
    for i in range(len(b)):
        exponent += b[i] * Q_func(i, mu)
        
    return math.exp(exponent)

def compute_b0(b_free, Q_func, num_integration_steps=1000):
    """
    Computes b_0 to satisfy the normalization constraint:
    integral_{-1}^{1} p(mu) dmu = 2*pi
    
    Args:
        b_free (list of float): Coefficients [b_1, ..., b_M].
        Q_func (callable): Basis function.
        num_integration_steps (int): Number of steps for numerical integration.
        
    Returns:
        float: The computed b_0.
    """
    # Numerical integration of exp(sum_{i=1}^M b_i Q_i(mu))
    # Using simple Riemann sum or Trapezoidal rule
    integral = 0.0
    mu_min = -1.0
    mu_max = 1.0
    step = (mu_max - mu_min) / num_integration_steps
    
    for k in range(num_integration_steps):
        mu = mu_min + k * step
        # Calculate exponent for free coefficients
        exponent = 0.0
        for i in range(1, len(b_free) + 1):
            exponent += b_free[i-1] * Q_func(i, mu)
        
        integral += math.exp(exponent) * step
        
    # p(mu) = exp(b_0 + sum_{i=1}^M b_i Q_i(mu))
    # integral p(mu) dmu = exp(b_0) * integral(exp(sum_{i=1}^M b_i Q_i(mu))) dmu = 2*pi
    # exp(b_0) = 2*pi / integral
    # b_0 = ln(2*pi) - ln(integral)
    
    if integral <= 0:
        raise ValueError("Integral must be positive for normalization.")
        
    b0 = math.log(2 * math.pi) - math.log(integral)
    return b0
```

**Parameters**
*   $M$: Degree of the polynomial. The number of free parameters is $M$.
*   $b_1 \dots b_M$: Free coefficients determining the shape.
*   Typical values: For fitting Rayleigh scattering, $(b_1, b_2) = (0, 0.68)$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Negative Values:** Standard polynomial phase functions can exhibit negative values, which is physically invalid. The exponential form avoids this by construction [S1, unofficial].
*   **Optimization:** Enforcing non-negativity on standard polynomial coefficients is computationally expensive and limits representation capability, introducing local minima. The exponential form relaxes this constraint [S1, unofficial].
*   **Normalization:** $b_0$ must be recomputed whenever other coefficients change to maintain the integral constraint [S1, unofficial].

## Subtopics

### von Mises-Fisher Phase Function
A special case of the Exponential Phase Function with $M=1$. It is computationally cheaper than higher-degree models but less flexible. Choose it when the scattering is strongly forward or backward peaked but symmetric. [Link: [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md)]

### Fisher-Bingham Phase Function
A special case of the Exponential Phase Function with $M=2$. It offers more shape flexibility than vMF. Choose it when approximating distributions like Rayleigh or complex Mie scattering. [Link: [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md)]

## Operating characteristics
*   **Time Complexity:** Evaluation is $O(M)$ per sample, where $M$ is the degree. Numerical integration for $b_0$ is $O(M \cdot N)$, where $N$ is the number of integration steps.
*   **Memory Complexity:** $O(M)$ to store coefficients.
*   **Strengths:** Guaranteed non-negativity; flexible representation of various phase functions (isotropic, vMF, Rayleigh-like); suitable for inverse rendering optimization due to smooth, unconstrained parameter space for $b_1 \dots b_M$.
*   **Limitations:** Requires numerical integration for normalization (unless analytical solutions exist for specific bases); higher degrees may overfit noise in inverse rendering.
*   **When to use:** When accurate fitting of measured or theoretical phase functions (like Mie or Rayleigh) is required, or when using inverse rendering techniques where polynomial positivity constraints are problematic.
*   **When not to use:** When computational speed is critical and a simple HG or vMF approximation suffices.

## Background
The Exponential Phase Function was introduced by Thanh-Trung Ngo and Hajime Nagahara in "A General Scattering Phase Function for Inverse Rendering" (2022) [S1, unofficial]. It addresses the limitations of polynomial phase functions in inverse rendering, where enforcing positivity constraints is difficult. The one-degree case relates to the von Mises-Fisher distribution used by Gkioulekas et al. [13], and the two-degree case relates to the Fisher-Bingham distribution [24]. The method was evaluated against Mie scattering theory datasets [S1, unofficial].

## Sources
[S1] A General Scattering Phase Function for Inverse Rendering, https://doi.org/10.48550/arxiv.2209.13875, fetched 2026-10-26 (unofficial)
