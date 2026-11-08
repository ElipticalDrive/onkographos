# Smooth Gradient Marginalization

Also called: Marginalization of Gaussian Gradient

## Overview
Smooth Gradient Marginalization is a technique to sample an $n$-dimensional Gaussian gradient by marginalizing the probability density function (PDF) over all dimensions except the one being differentiated, allowing individual dimension sampling [S1]. It differs from methods that only blur in the direction of differentiation by blurring all dimensions, which is more consistent with higher-order differentials [S1].

## How it works
**Inputs:** The $n$-dimensional Gaussian function $N(\tau)$, the dimension index $i$ to differentiate, and the standard deviation $\sigma$.
**Outputs:** A marginal PDF $p_G(\tau)$ for dimension $i$ and the corresponding CDF for sampling.

**Data Structures:**
*   $\tau$: Vector of length $n$.
*   $N(\tau)$: Scalar value of the Gaussian at $\tau$.
*   $p_G(\tau)$: Scalar marginal PDF for dimension $i$.

**Formulas:**
The marginal PDF is obtained by integrating the $n$-dimensional gradient over all dimensions $j \neq i$ [S1]:
$$ p_G(\tau) = \int p_G(\tau) d\tau_{j \neq i} $$
Note: The source notation $p_G(\tau)$ on the RHS refers to the joint gradient PDF before marginalization [S1].

The $n$-dimensional Gaussian gradient with respect to $\tau_i$ is:
$$ \nabla_i N(\tau) = -\frac{\tau_i}{\sigma^2} N(\tau) $$
Since the Gaussian is separable, $N(\tau) = \prod_{j=1}^n N(\tau_j)$, the marginalization integrates out all $N(\tau_j)$ for $j \neq i$. Since $\int N(\tau_j) d\tau_j = 1$ and the gradient term depends only on $\tau_i$, the marginal PDF becomes [S1]:
$$ p_G(\tau) = \left| -\frac{\tau_i}{\sigma^2} N(\tau_i) \right| \prod_{j \neq i} \int N(\tau_j) d\tau_j = \left| \frac{\tau_i}{\sigma^2} N(\tau_i) \right| $$
The PDF is positivized and scaled to form a valid distribution [S1].

**Hessian Elements:**
The diagonal element of the Hessian is:
$$ \nabla^2 N(\tau)_{ii} = \frac{\partial^2 N(\tau)}{\partial \tau_i^2} = \left(\frac{\tau_i^2}{\sigma^4} - \frac{1}{\sigma^2}\right) N(\tau) $$
The off-diagonal element ($i \neq j$) is:
$$ \nabla^2 N(\tau)_{ij} = \frac{\tau_i \tau_j}{\sigma^4} N(\tau) $$
A critical distinction is that the diagonal element is not a special case of the off-diagonal element because the variable appears twice in the differentiation ($d(u^2)/du = 2u$ vs $d(uv)/du = v$) [S1].

**Sampling Steps:**
1.  Identify the roots of the second-order derivative at $\tau_i = \pm \sigma$ [S1].
2.  Calculate the scaling constant $\beta$ using the gradient at $-\sigma$:
    $$ \beta = \frac{1}{4 \nabla N(-\sigma)} $$
3.  Construct the CDF $P^H_{ii}(\tau_i)$ by positivizing and scaling the second-order derivative such that the integral over $[-\sigma, \sigma]$ is twice the integral outside, ensuring the CDF is $1/2$ at $-\sigma$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smooth_gradient_marginalization(N_tau, tau_i, sigma):
    """
    Computes the marginal PDF for the gradient of an n-dimensional Gaussian
    along dimension i.
    
    Args:
        N_tau: The value of the n-dimensional Gaussian N(tau).
        tau_i: The scalar value of the i-th dimension.
        sigma: The standard deviation of the Gaussian.
        
    Returns:
        p_g: The marginal PDF value for dimension i.
    """
    # Gradient of 1D Gaussian component
    grad_1d = -(tau_i / (sigma ** 2)) * N_tau # Note: N_tau here is effectively N(tau_i) if separable
    # However, the marginalization result depends only on tau_i and N(tau_i)
    # The full N(tau) factors out as 1 for the marginal PDF of the gradient direction
    # p_G(tau) = |tau_i / sigma^2 * N(tau_i)|
    
    # Re-evaluating based on source derivation:
    # The marginal PDF is derived from |grad_i N(tau)| integrated over other dims.
    # Since N(tau) = N(tau_i) * Product(N(tau_j)), and integral of N(tau_j) is 1.
    # The term |grad_i N(tau)| = | -tau_i/sigma^2 * N(tau) |
    # Marginal PDF = | -tau_i/sigma^2 * N(tau_i) | * Product(1)
    
    # To be precise, N_tau passed should be N(tau_i) for the marginal calculation
    # or the function handles the separation. Assuming N_tau is N(tau_i):
    
    p_g = abs((tau_i / (sigma ** 2)) * N_tau)
    return p_g

def sample_hessian_diagonal(tau_i, sigma):
    """
    Computes the CDF value for sampling the diagonal of the Hessian.
    """
    # Calculate beta
    # Gradient of 1D Gaussian: d/dt N(t) = -t/sigma^2 * exp(-t^2/2sigma^2)
    # N(t) = 1/sqrt(2pi sigma^2) * exp(-t^2/2sigma^2)
    # grad_N(-sigma) = -(-sigma)/sigma^2 * N(-sigma) = (1/sigma) * N(-sigma)
    
    # N(-sigma) = 1/sqrt(2pi sigma^2) * exp(-1/2)
    # grad_N(-sigma) = (1/sigma) * 1/sqrt(2pi sigma^2) * exp(-0.5)
    
    # beta = 1 / (4 * grad_N(-sigma))
    
    # CDF definition
    if tau_i < -sigma:
        # beta * grad_N(tau_i)
        # Note: grad_N is negative for tau_i > 0, positive for tau_i < 0?
        # grad_N(t) = -t/sigma^2 * N(t). For t < 0, grad_N > 0.
        # The source says: beta * grad_N(tau_i) if tau_i < -sigma
        # We need to ensure positivity. The source implies the formula handles it or beta absorbs sign.
        # Let's use the explicit piecewise from source.
        cdf = beta * gradient_1d(tau_i, sigma)
    elif -sigma <= tau_i <= sigma:
        cdf = 0.5 + beta * gradient_1d(tau_i, sigma)
    else:
        cdf = 1.0 - beta * gradient_1d(tau_i, sigma)
        
    return cdf

def gradient_1d(t, sigma):
    return -(t / (sigma ** 2)) * gaussian_1d(t, sigma)

def gaussian_1d(t, sigma):
    import math
    return (1.0 / math.sqrt(2 * math.pi * sigma ** 2)) * math.exp(-0.5 * (t / sigma) ** 2)
```
*[G] The pseudocode assumes standard 1D Gaussian definitions. The constant $\beta$ must be pre-calculated as $1/(4 \nabla N(-\sigma))$. The source notes that the CDF is $1/2$ at $-\sigma$, which implies the scaling ensures the total probability mass is normalized correctly for the positivized distribution.*

## Subtopics

### Sampling Diagonal of Hessian
This variant samples the diagonal elements of the Hessian matrix. It derives the valid distribution of the second-order derivative of the one-dimensional Gaussian by positivization and scaling, which then applies to higher dimensions [S1]. The 1D Gaussian second-order derivative is $\beta \left( \frac{1}{\sigma^2} - \frac{\tau_i^2}{\sigma^4} \right) N(\tau_i)$ [S1]. The roots are at $\tau_i = \pm \sigma$ [S1]. The PDF is $p^H_{ii} = |\beta \nabla^2 N(\tau_i)|$ [S1]. The CDF is defined piecewise: $\beta \nabla N(\tau_i)$ if $\tau_i < -\sigma$; $1/2 + \beta \nabla N(\tau_i)$ if $\tau_i \in [-\sigma, \sigma]$; $1 - \beta \nabla N(\tau_i)$ if $\tau_i > \sigma$ [S1].

## Operating characteristics
Not given in the sources.

## Background
This method is described in "Stochastic Gradient Estimation for Higher-Order Differentiable Rendering" [S1]. It builds on the work of Fischer and Ritschel [7] who derived 1D Gaussian gradient sampling, but extends it to $n$-dimensions via marginalization [S1].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
