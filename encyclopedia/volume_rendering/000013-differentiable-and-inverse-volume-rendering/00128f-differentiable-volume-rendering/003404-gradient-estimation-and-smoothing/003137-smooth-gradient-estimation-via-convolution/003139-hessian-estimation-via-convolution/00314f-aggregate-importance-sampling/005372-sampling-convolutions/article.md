# Sampling Convolutions

Also called: Convolution Correction for Hemispherical Sampling

## Overview
Sampling Convolutions is a technique to compute accurate gradients in integrands with large support using spherical convolutions, specifically addressing the mismatch between standard hemispherical sampling and the full-sphere support of a convolution [S1]. It differs from naive convolution sampling by introducing a precomputed correction factor that compensates for the energy loss incurred when the sampling distribution (typically a BRDF) is restricted to a hemisphere while the convolution kernel extends over the full sphere [S1].

## How it works
**Inputs:**
- The integrand function $f(\mu)$, typically a product of a BSDF and incident illumination.
- The convolution kernel $k(\mu, \omega)$, defined on the unit sphere $S^2$.
- The default sampling strategy (PDF) used by the renderer for $f$, denoted $p_f(\mu)$.

**Outputs:**
- An unbiased estimate of the integral $\int_{S^2} f(\mu) d\mu$.

**Data Structures:**
- **Correction Lookup Table:** A precomputed array or function mapping direction $\mu$ to the scalar correction factor $C(\mu)$. The table covers the hemisphere $\mathcal{H}$ or the full sphere, depending on implementation, but is primarily queried for directions sampled from $\mathcal{H}$.

**Parameters:**
- **Kernel Width:** The method assumes $k$ is a narrowly peaked convolution kernel. If $k$ is wide, the similarity between $f \otimes k$ and $f$ degrades, making the default sampling strategy less effective as an importance distribution [S1].
- **Correction Factor Values:** $C(\mu) = 1/2$ at grazing angles (directions parallel to the surface) and $C(\mu) \approx 1$ for directions inside the hemisphere (near the normal) [S1].

**Steps:**
1. **Precompute Correction Factor:** Calculate $C(\mu)$ for the chosen kernel $k$. This is defined as the integral of the kernel over the hemisphere $\mathcal{H}$ for a given input direction $\mu$:
   $$ C(\mu) = \int_{\mathcal{H}} k(\mu, \omega) d\omega $$
   This value is stored for later retrieval during sampling [S1].

2. **Sample Direction:** Use the renderer's default sampling strategy for $f$ to generate a sample direction $\mu$ from the hemisphere $\mathcal{H}$. The probability density function (PDF) for this sample is $p_f(\mu)$ [S1].

3. **Compute Weighted Integrand:** Evaluate the inner integral term for the sampled direction. To obtain an unbiased estimate, replace $f(\mu)$ with $f(\mu)/C(\mu)$ in the integrand. The contribution for a sample $\mu$ is:
   $$ \text{Contribution} = \frac{f(\mu)}{C(\mu) \cdot p_f(\mu)} \int_{S^2} k(\mu, \omega) d\omega $$
   Note: In practice, the outer integral over $\omega$ is often handled by further sampling or analytical integration depending on the specific rendering equation context, but the core correction is the division by $C(\mu)$ [S1].

4. **Accumulate:** Accumulate the contribution to the final estimate.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_convolutions(f, kernel_k, p_f, precomputed_C):
    """
    Estimates the integral of f using convolution sampling with hemispherical correction.
    
    Args:
        f: Function evaluating the integrand f(mu).
        kernel_k: Convolution kernel function k(mu, omega).
        p_f: PDF function for the default sampling strategy of f.
        precomputed_C: Function or lookup table for C(mu).
        
    Returns:
        unbiased_estimate: Float, the unbiased estimate of the integral.
    """
    # Step 2: Sample direction from hemisphere using default strategy
    # Note: The specific implementation of p_f depends on the renderer's BSDF sampler.
    # We assume a standard importance sampling step.
    mu, pdf_value = p_f.sample()
    
    # Check if mu is valid (on the hemisphere)
    if not is_on_hemisphere(mu):
        return 0.0 # Or handle error appropriately
        
    # Step 1 & 3: Retrieve correction factor and evaluate integrand
    # C(mu) is precomputed based on the kernel k
    C_mu = precomputed_C(mu)
    
    # Avoid division by zero
    if C_mu == 0:
        return 0.0
        
    # Evaluate f(mu)
    f_val = f(mu)
    
    # The unbiased estimate for the inner part involves dividing by C(mu)
    # The full estimator for the convolution integral \int_{S^2} f(mu) dmu
    # is derived from Eq. 35-37.
    # The term \int_{S^2} k(mu, omega) domega is the kernel's total energy.
    # If the kernel is normalized to 1 over S^2, this integral is 1.
    # However, the correction C(mu) accounts for the missing lower hemisphere.
    
    # The estimator for the integral \int_{S^2} f(mu) dmu is:
    # f(mu) / (C(mu) * p_f(mu)) * (integral of k over S^2)
    # Assuming k is normalized over S^2:
    kernel_energy = 1.0 
    
    weight = f_val / (C_mu * pdf_value)
    
    return weight * kernel_energy

def is_on_hemisphere(direction):
    # Check if direction is in the upper hemisphere (dot with normal > 0)
    # Assuming normal is (0,0,1)
    return direction.z > 0
```

**Edge Cases and Pitfalls:**
- **Hemispherical Mismatch:** Standard BRDF sampling only covers the hemisphere $\mathcal{H}$, but the support of $f \otimes k$ is the full sphere $S^2$. Without correction, this causes energy loss [S1].
- **Grazing Angles:** At grazing angles, $C(\mu) = 1/2$. Failure to apply the correction here results in a 50% underestimation of energy [S1].
- **Kernel Width:** The method relies on $f \otimes k$ being similar to $f$. If the kernel is too wide, the default sampling strategy may become a poor importance distribution, leading to high variance [S1].

## Operating characteristics
- **Time Complexity:** The method adds the cost of evaluating or looking up $C(\mu)$ and one division per sample. The sampling cost is dominated by the default BSDF sampler.
- **Memory Complexity:** Requires storage for the precomputed correction factor $C(\mu)$. The size depends on the resolution of the lookup table for the hemisphere.
- **Strengths:** Provides an unbiased estimate of the convolution integral using existing hemispherical sampling strategies. It is simple to implement as a post-processing correction on the integrand value.
- **Limitations:** Relies on the assumption that the default sampling strategy for $f$ is a good approximation for $f \otimes k$. This holds best for narrowly peaked kernels.
- **When to use:** When differentiating integrands involving spherical convolutions with large support, particularly in differentiable rendering contexts where accurate gradients are needed [S1].

## Background
The method originates from the work on reparameterizing discontinuous integrands for differentiable rendering [S1]. It addresses a specific issue in differentiable volume and path rendering where spherical convolutions are used to smooth integrands for gradient estimation. The problem of hemispherical sampling mismatch in the context of spherical convolutions was identified as a source of bias in gradient estimation.

## Sources
[S1] Reparameterizing discontinuous integrands for differentiable rendering, https://doi.org/10.1145/3355089.3356510, fetched 2026-10-26
