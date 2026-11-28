# Importance Points Sampling

Also called: sampling weight function

## Overview
Importance Points Sampling is a specialized sampling weight function $w_s(t)$ designed to achieve well-balanced Eikonal loss regularization for Unsigned Distance Function (UDF) volume rendering [S1]. Unlike standard rendering weights that concentrate points near the surface, this function distributes more sampling weights before the intersection points to prevent unbalanced gradient magnitudes on the two sides of the surface, which would otherwise hamper UDF quality [S1]. It provides the specific mathematical formulation for the sampling density $\tau_s(t)$ used in the second stage of [NeUDF Hierarchical Importance Sampling](4863-neudf-hierarchical-importance-sampling.md) [S1].

## How it works
**Inputs and Outputs:** The method takes the ray parameter $t$, the UDF value $\Psi(p(t))$ at the ray point $p(t)$, and hyperparameters $s$ and $b$ as inputs [S1]. It outputs the opaque density $\tau_s(t)$ and the sampling weight $w_s(t)$ [S1].

**Data Structures:** The method requires no complex data structures and relies entirely on scalar arithmetic operations on the ray parameter and UDF values [G].

**Formulas:** The sampling weight is defined as $w_s(t) = \tau_s(t) e^{-\int_0^t \tau_s(u) du}$ [S1]. The opaque density is derived from a transformation function $\zeta_s$ as $\tau_s(t) = \zeta_s(\Psi(p(t)))$ [S1]. The transformation function is defined as $\zeta_s(d) = s \frac{e^{-bd}}{(1 + e^{-bd})^2}$ [S1].

**Constraints:** The function $\zeta_s(d)$ must satisfy $\zeta_s(d) > 0$ and $\zeta'_s(d) < 0$ for all $d > 0$ [S1].

**Parameters:** The parameter $s$ starts at $0.05$ and changes every sampling step $z$ with a rate of $2^{z-1}$ [S1]. The parameter $b$ controls the intensity of the weight at $d=0$ [S1].

**Steps:** 
1. Evaluate the UDF value $d = \Psi(p(t))$ at the current ray point [S1].
2. Compute $\zeta_s(d)$ using the current values of $s$ and $b$ [S1].
3. Set the opaque density $\tau_s(t) = \zeta_s(d)$ [S1].
4. Compute the cumulative opacity integral $A(t) = \int_0^t \tau_s(u) du$, which in discrete implementation is the accumulated sum of $\tau_s$ along the ray [G].
5. Compute the sampling weight $w_s(t) = \tau_s(t) e^{-A(t)}$ [S1].
6. Use $w_s(t)$ to guide the selection of sample points along the ray [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_sampling_density(t, udf_value, s, b):
    """
    Computes the opaque density tau_s(t) for importance sampling.
    
    Args:
        t: Current ray parameter (distance).
        udf_value: The Unsigned Distance Function value d at p(t).
        s: Scaling parameter (dynamic).
        b: Intensity control parameter.
        
    Returns:
        tau_s: The opaque density for sampling.
    """
    # Compute zeta_s(d) = s * exp(-b*d) / (1 + exp(-b*d))^2
    exp_bd = math.exp(-b * udf_value)
    denominator = (1 + exp_bd) ** 2
    tau_s = s * exp_bd / denominator
    
    # Note: The integral term exp(-int_0^t tau_s(u) du) is handled by the 
    # renderer's accumulation buffer or ray marching loop. 
    # This function provides the local density tau_s(t) which is then 
    # combined with the accumulated transmittance T(t) = exp(-int_0^t tau_s(u) du).
    # The final weight used for sampling probability is tau_s(t) * T(t).
    # The discrete accumulation of tau_s along the ray is performed externally.
    
    return tau_s
```

**Edge Cases and Pitfalls:** If the sampling function employs the same weights as the rendering function, the Eikonal loss regularization leads to highly unbalanced gradient magnitudes on the two sides of the surface, significantly hampering UDF quality [S1].

## Subtopics

### [Hierarchical Volume Sampling for UDFs](14671-hierarchical-volume-sampling-for-udfs.md)
A technique to upsample points along a ray for UDF volume rendering, addressing sampling bias inherent in standard importance sampling. It introduces an auxiliary point sampling prior to refine sampling densities and a uniform upsampling strategy. Choose this when rendering efficiency and 3D perception quality are critical for UDFs.

## Operating characteristics
The method requires $O(1)$ time per sample point for weight evaluation [G]. It requires $O(1)$ auxiliary memory [G]. Its primary strength is achieving well-balanced regularization across all space, preventing the unbalanced gradient magnitudes that degrade UDF reconstruction quality [S1]. Its main limitation is the requirement for careful tuning of parameters $s$ and $b$ [S1]. It should be used when performing volume rendering with Neural Unsigned Distance Fields where Eikonal loss regularization is critical for surface accuracy [S1]. It should not be used when using Signed Distance Functions, as the specific bias correction for UDFs is not needed and standard rendering weights may suffice [G].

## Background
The method was introduced in the paper "NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering" [S1]. It addresses the specific challenges of rendering UDFs, where the lack of a signed distinction causes standard importance sampling to fail in balancing the Eikonal loss [S1]. The approach is inspired by NeuS [53] but adapts the weight function specifically for the UDF domain to ensure unbiased rendering and stable gradient propagation [S1].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
