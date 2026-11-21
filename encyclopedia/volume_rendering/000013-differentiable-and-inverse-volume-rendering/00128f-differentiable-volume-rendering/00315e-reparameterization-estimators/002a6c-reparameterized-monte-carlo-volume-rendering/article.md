# Reparameterized Monte Carlo Volume Rendering

Also called: R/MC, R/MC estimate, R/SMC estimate, Reparameterized Stratified Monte Carlo (R/SMC) Volume Rendering

## Overview
Reparameterized Monte Carlo Volume Rendering (R/MC) is a differentiable volume rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick, allowing end-to-end gradient-based optimization without auxiliary models [S1, unofficial]. It sets itself apart from standard volumetric integration by transforming the integration domain from ray distance $t$ to cumulative opacity $y$, enabling unbiased gradient estimation through the inverse opacity function [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $o$, direction $d$, near bound $t_n$, far bound $t_f$, density field $\sigma(x)$, radiance field $c(x)$ [S1, unofficial].
*   **Outputs:** Estimated radiance $\hat{C}$ for the ray [S1, unofficial].

**Data Structures:**
*   **Ray:** Defined as $r(t) = o + td$ [S1, unofficial].
*   **Density Field:** $\sigma_r(t) = \sigma(o + td)$ [S1, unofficial].
*   **Radiance Field:** $c_r(t) = c(o + td)$ [S1, unofficial].
*   **Opacity Function:** $F_r(t) = 1 - \exp\left(-\int_{t_n}^{t} \sigma_r(s) ds\right)$ [S1, unofficial].
*   **Total Opacity:** $y_f = F_r(t_f)$ [S1, unofficial].

**Steps:**
1.  Define the ray $r = o + td$ with near/far bounds $t_n, t_f$ [S1, unofficial].
2.  Define the density field $\sigma$ and radiance field $c$ [S1, unofficial].
3.  Compute the total opacity $y_f = F_r(t_f) = 1 - \exp\left(-\int_{t_n}^{t_f} \sigma_r(s) ds\right)$ [S1, unofficial].
4.  Generate $k$ independent and identically distributed (i.i.d.) samples $u_i \sim U[0,1]$ for R/MC, or stratified samples $v_i$ for R/SMC [S1, unofficial].
5.  For each sample, compute the target cumulative opacity $y_i = y_f u_i$ (or $y_f v_i$) [S1, unofficial].
6.  Solve for the ray distance $t_i$ such that $F_r(t_i) = y_i$ using the inverse opacity function $F_r^{-1}$ [S1, unofficial].
7.  Query the radiance $c_r(t_i)$ at the computed distances [S1, unofficial].
8.  Compute the estimate $\hat{C} = \frac{y_f}{k} \sum_{i=1}^k c_r(t_i)$ [S1, unofficial].

**Formulas:**
The integral boundaries are normalized to $[0, 1]$ by substituting $y = y_f u$ where $u \sim U[0,1]$ [S1, unofficial]:
$$
\int_{0}^{y_f} c_r(F_r^{-1}(y)) dy = \int_0^1 y_f c_r(F_r^{-1}(y_f u)) du
$$
The R/MC estimate for a ray $r$ with $k$ i.i.d. uniform samples $u_1, ..., u_k \sim U[0,1]$ is [S1, unofficial]:
$$
\hat{C}_{MC}^R(r) := \frac{y_f}{k} \sum_{i=1}^k c_r(F_r^{-1}(y_f u_i))
$$
The R/SMC estimate replaces uniform samples $u_i$ with stratified samples $v_i$ drawn from regular grid bins [S1, unofficial]:
$$
v_i \sim U\left[\frac{i-1}{k+1}, \frac{i}{k+1}\right] \quad \text{for } i=1,...,k
$$
The estimate is [S1, unofficial]:
$$
\hat{C}_{SMC}^R(r) := \frac{y_f}{k} \sum_{i=1}^k c_r(F_r^{-1}(y_f v_i))
$$

**Parameters:**
*   $k$: Number of samples per ray. Typical values are not specified in the sources [S1, unofficial].

**Pitfalls:**
*   The method cannot compute $F_r$ analytically in practice because $\sigma_r$ is only queried at discrete points, requiring approximations of $F_r$ and its inverse $F_r^{-1}$ [S1, unofficial].
*   The sources do not specify the numerical method for approximating $F_r$ and $F_r^{-1}$ (e.g., piecewise constant vs. linear interpolation, root-finding algorithms) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def reparameterized_mc_volume_rendering(ray_origin, ray_direction, t_near, t_far, density_field, radiance_field, num_samples):
    """
    Estimates radiance for a ray using Reparameterized Monte Carlo Volume Rendering.
    
    Args:
        ray_origin: 3D vector (x, y, z)
        ray_direction: 3D vector (x, y, z), assumed normalized
        t_near: float, near bound of the ray
        t_far: float, far bound of the ray
        density_field: function p -> float, returns density at point p
        radiance_field: function p -> float, returns radiance at point p
        num_samples: int, number of samples k
        
    Returns:
        float, estimated radiance
    """
    # [G] Simple deterministic uniform random number generator
    def uniform_random():
        nonlocal rng_state
        rng_state = (rng_state * 1103515245 + 12345) & 0x7fffffff
        return rng_state / 0x7fffffff
    rng_state = 42  # Initial seed

    # Step 3: Compute total opacity y_f
    total_density_integral = numerical_integrate_density(density_field, ray_origin, ray_direction, t_near, t_far)
    y_f = 1.0 - math.exp(-total_density_integral)
    
    # Handle degenerate case where the ray is completely transparent
    if y_f == 0.0:
        return 0.0
    
    radiance_sum = 0.0
    
    for _ in range(num_samples):
        # Step 4 & 5: Generate sample and compute target cumulative opacity
        u = uniform_random()
        y_target = y_f * u
        
        # Step 6: Solve for ray distance t_i such that F_r(t_i) = y_target
        # F_r(t) = 1 - exp(-int_{t_near}^t sigma_r(s) ds)
        # y_target = 1 - exp(-int_{t_near}^t sigma_r(s) ds)
        # int_{t_near}^t sigma_r(s) ds = -ln(1 - y_target)
        target_density_integral = -math.log(1.0 - y_target)
        
        # [G] Binary search to invert the cumulative density integral
        t_i = invert_opacity_function(density_field, ray_origin, ray_direction, t_near, t_far, target_density_integral)
        
        # Step 7: Query radiance at the sampled point
        point = ray_origin + t_i * ray_direction
        radiance = radiance_field(point)
        
        radiance_sum += radiance
        
    # Step 8: Compute estimate
    estimate = (y_f / num_samples) * radiance_sum
    
    return estimate

def numerical_integrate_density(density_field, ray_origin, ray_direction, t_start, t_end):
    """
    Approximates the integral of density along the ray from t_start to t_end.
    [G] Trapezoidal rule approximation.
    """
    num_steps = 100
    dt = (t_end - t_start) / num_steps
    integral = 0.0
    for j in range(num_steps):
        t1 = t_start + j * dt
        t2 = t_start + (j + 1) * dt
        p1 = ray_origin + t1 * ray_direction
        p2 = ray_origin + t2 * ray_direction
        sigma1 = density_field(p1)
        sigma2 = density_field(p2)
        integral += (sigma1 + sigma2) / 2.0 * dt
    return integral

def invert_opacity_function(density_field, ray_origin, ray_direction, t_near, t_far, target_integral):
    """
    Finds t such that the integral of density from t_near to t equals target_integral.
    [G] Uses binary search as a robust root-finding method for monotonic functions.
    """
    t_low = t_near
    t_high = t_far
    for _ in range(50):
        t_mid = (t_low + t_high) / 2.0
        current_integral = numerical_integrate_density(density_field, ray_origin, ray_direction, t_near, t_mid)
        if current_integral < target_integral:
            t_low = t_mid
        else:
            t_high = t_mid
    return (t_low + t_high) / 2.0
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(k \cdot N_{steps})$ per ray, where $k$ is the number of samples and $N_{steps}$ is the number of steps in the numerical integration and root-finding procedures [S1, unofficial]. The sources do not specify $N_{steps}$ [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per ray, assuming the density and radiance fields are queried on the fly [S1, unofficial].
*   **Measured Performance:** The sources do not provide measured performance metrics [S1, unofficial].
*   **Scaling:** The method scales linearly with the number of samples $k$ [S1, unofficial].
*   **Strengths:** Allows end-to-end learning with gradient descent; alleviates costs associated with the color component of the neural radiance field at the additional cost of the density sampling algorithm [S1, unofficial].
*   **Limitations:** Requires approximations of $F_r$ and its inverse $F_r^{-1}$, which can be computationally expensive and introduce approximation errors [S1, unofficial].
*   **When to use:** When differentiable volume rendering is required and the density field is available [S1, unofficial].
*   **When not to use:** When analytical computation of $F_r$ and $F_r^{-1}$ is not possible and numerical approximations are too costly [S1, unofficial].

## Background
*   **Origin:** The method was proposed in the paper "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial].
*   **History and Development:** The paper introduces the reparameterization of the sampling algorithm to allow end-to-end learning with gradient descent [S1, unofficial].
*   **Where it is used:** Learning-based approaches for representing 3D scenes and generating novel views [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
