# Reparameterized Monte Carlo Volume Rendering

Also called: R/MC, R/MC estimate, R/SMC estimate, Reparameterized Stratified Monte Carlo (R/SMC) Volume Rendering

## Overview
Reparameterized Monte Carlo Volume Rendering (R/MC) is a differentiable volume rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick, allowing end-to-end gradient-based optimization without auxiliary models [S1, unofficial]. It sets itself apart from standard volumetric integration by transforming the integration domain from ray distance $t$ to cumulative opacity $y$, enabling unbiased gradient estimation through the inverse opacity function [S1, unofficial]. It differs from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) by explicitly defining the sampling distribution via the ray's opacity function rather than general volume elements.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $o$, direction $d$, near bound $t_n$, far bound $t_f$, density field $\sigma(x)$, radiance field $c(x)$.
*   **Outputs:** Estimated radiance $\hat{C}$ for the ray.

**Data Structures:**
*   **Ray:** Defined as $r(t) = o + td$.
*   **Density Field:** $\sigma_r(t) = \sigma(o + td)$.
*   **Radiance Field:** $c_r(t) = c(o + td)$.
*   **Opacity Function:** $F_r(t) = 1 - \exp\left(-\int_{t_n}^{t} \sigma_r(s) ds\right)$.
*   **Total Opacity:** $y_f = F_r(t_f)$.

**Steps:**
1.  Define the ray $r = o + td$ with near/far bounds $t_n, t_f$.
2.  Define the density field $\sigma$ and radiance field $c$.
3.  Compute the total opacity $y_f = F_r(t_f) = 1 - \exp\left(-\int_{t_n}^{t_f} \sigma_r(s) ds\right)$ [S1, unofficial].
4.  Generate $k$ independent and identically distributed (i.i.d.) samples $u_i \sim U[0,1]$ for R/MC, or stratified samples $v_i$ for R/SMC [S1, unofficial].
5.  For each sample, compute the target cumulative opacity $y_i = y_f u_i$ (or $y_f v_i$) [S1, unofficial].
6.  Solve for the ray distance $t_i$ such that $F_r(t_i) = y_i$. This requires the inverse opacity function $F_r^{-1}$ [S1, unofficial].
7.  Query the radiance $c_r(t_i)$ at the computed distances [S1, unofficial].
8.  Compute the estimate $\hat{C} = \frac{y_f}{k} \sum_{i=1}^k c_r(t_i)$ [S1, unofficial].

**Formulas:**
The integral boundaries are normalized to $[0, 1]$ by substituting $y = y_f u$ where $u \sim U[0,1]$ [S1, unofficial]:
$$
\int_{y_n}^{y_f} c_r(F_r^{-1}(y)) dy = \int_0^1 y_f c_r(F_r^{-1}(y_f u)) du
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
*   $k$: Number of samples per ray. Typical values are not specified in the sources.

**Pitfalls:**
*   The method cannot compute $F_r$ analytically in practice because $\sigma_r$ is only queried at discrete points. It requires approximations of $F_r$ and its inverse $F_r^{-1}$ [S1, unofficial].
*   The sources do not specify the numerical method for approximating $F_r$ and $F_r^{-1}$ (e.g., piecewise constant vs. linear interpolation, root-finding algorithms) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_mc_volume_rendering(ray_origin, ray_direction, t_near, t_far, density_field, radiance_field, num_samples):
    """
    Estimates radiance for a ray using Reparameterized Monte Carlo Volume Rendering.
    
    Args:
        ray_origin: 3D vector (x, y, z)
        ray_direction: 3D vector (x, y, z), assumed normalized
        t_near: float, near bound of the ray
        t_far: float, far bound of the ray
        density_field: function sigma(t) -> float, returns density at ray distance t
        radiance_field: function c(t) -> float, returns radiance at ray distance t
        num_samples: int, number of samples k
        
    Returns:
        float, estimated radiance
    """
    # Step 3: Compute total opacity y_f
    # Note: The sources state F_r(t) = 1 - exp(-int_{t_n}^t sigma_r(s) ds)
    # In practice, this integral must be approximated. The sources mention
    # piecewise constant or linear approximations but do not provide the exact algorithm.
    # [G] We assume a numerical integration method (e.g., trapezoidal rule) is available
    # to compute the integral of density along the ray.
    total_density_integral = numerical_integrate_density(density_field, ray_origin, ray_direction, t_near, t_far)
    y_f = 1.0 - math.exp(-total_density_integral)
    
    # Step 4: Generate samples
    # For R/MC: uniform samples
    # For R/SMC: stratified samples
    # The pseudocode below implements R/MC. For R/SMC, replace uniform sampling with stratified sampling.
    samples = [random.uniform(0.0, 1.0) for _ in range(num_samples)]
    
    radiance_sum = 0.0
    
    for u in samples:
        # Step 5: Compute target cumulative opacity
        y_target = y_f * u
        
        # Step 6: Solve for ray distance t_i such that F_r(t_i) = y_target
        # This requires inverting the opacity function.
        # F_r(t) = 1 - exp(-int_{t_n}^t sigma_r(s) ds)
        # y_target = 1 - exp(-int_{t_n}^t sigma_r(s) ds)
        # exp(-int_{t_n}^t sigma_r(s) ds) = 1 - y_target
        # -int_{t_n}^t sigma_r(s) ds = ln(1 - y_target)
        # int_{t_n}^t sigma_r(s) ds = -ln(1 - y_target)
        # Let target_density_integral = -ln(1 - y_target)
        # We need to find t such that numerical_integrate_density(density_field, ray_origin, ray_direction, t_near, t) == target_density_integral
        
        target_density_integral = -math.log(1.0 - y_target)
        
        # [G] In practice, this inversion is done via root-finding (e.g., binary search)
        # over the ray segment [t_near, t_far], evaluating the cumulative density integral.
        t_i = invert_opacity_function(density_field, ray_origin, ray_direction, t_near, t_far, target_density_integral)
        
        # Step 7: Query radiance
        radiance = radiance_field(t_i)
        
        radiance_sum += radiance
        
    # Step 8: Compute estimate
    estimate = (y_f / num_samples) * radiance_sum
    
    return estimate

def numerical_integrate_density(density_field, ray_origin, ray_direction, t_start, t_end):
    """
    Approximates the integral of density along the ray from t_start to t_end.
    [G] The sources mention piecewise constant or linear approximations but do not specify the exact implementation.
    This is a placeholder for a numerical integration method.
    """
    # [G] Simple trapezoidal rule with a fixed number of steps for demonstration.
    # In a real implementation, the number of steps and method would depend on the
    # density field's smoothness and the required accuracy.
    num_steps = 100
    dt = (t_end - t_start) / num_steps
    integral = 0.0
    for i in range(num_steps):
        t1 = t_start + i * dt
        t2 = t_start + (i + 1) * dt
        # Sample density at midpoints for better accuracy
        t_mid1 = t1 + dt / 2
        t_mid2 = t2 + dt / 2
        sigma1 = density_field(ray_origin + t_mid1 * ray_direction)
        sigma2 = density_field(ray_origin + t_mid2 * ray_direction)
        integral += (sigma1 + sigma2) / 2 * dt
    return integral

def invert_opacity_function(density_field, ray_origin, ray_direction, t_near, t_far, target_integral):
    """
    Finds t such that the integral of density from t_near to t equals target_integral.
    [G] Uses binary search as a robust root-finding method for monotonic functions.
    """
    t_low = t_near
    t_high = t_far
    # Binary search for a fixed number of iterations
    for _ in range(50):
        t_mid = (t_low + t_high) / 2
        current_integral = numerical_integrate_density(density_field, ray_origin, ray_direction, t_near, t_mid)
        if current_integral < target_integral:
            t_low = t_mid
        else:
            t_high = t_mid
    return (t_low + t_high) / 2
```

## Subtopics

### Reparameterized Stratified Monte Carlo (R/SMC)
Uses stratified sampling to reduce variance compared to R/MC's uniform sampling, while maintaining the same differentiability properties [S1, unofficial]. See [Reparameterized Stratified Monte Carlo Volume Rendering](3309-reparameterized-volume-sampling.md) (Note: The sibling link provided in the prompt is for "Reparameterized Volume Sampling", but the text describes R/SMC as a variant of R/MC. The prompt lists "Reparameterized Volume Sampling" as a sibling. The text does not provide a separate article for R/SMC, so we link to the sibling that is most closely related, which is "Reparameterized Volume Sampling" if it covers stratified methods, or note that R/SMC is a variant. However, the prompt asks for subtopics of *this* method. The text describes R/SMC as an improvement. We will list it as a subtopic if it has a separate article. The prompt's sibling list includes "Reparameterized Volume Sampling". The text says "We further improve the estimate using stratified sampling... derive the reparameterized stratified Monte Carlo (R/SMC) estimate". It does not say R/SMC is a separate article. It is a variant. The prompt asks for "Subtopics: one entry per child". If there are no child articles, this section is empty. The prompt's sibling list does not include a specific "R/SMC" article. It includes "Reparameterized Volume Sampling". The text does not explicitly link R/SMC to a separate article. Therefore, there are no child articles to list. The section remains empty.)

## Operating characteristics
*   **Time Complexity:** $O(k \cdot N_{steps})$ per ray, where $k$ is the number of samples and $N_{steps}$ is the number of steps in the numerical integration and root-finding procedures. The sources do not specify $N_{steps}$.
*   **Memory Complexity:** $O(1)$ per ray, assuming the density and radiance fields are queried on the fly.
*   **Measured Performance:** The sources do not provide measured performance metrics.
*   **Scaling:** The method scales linearly with the number of samples $k$.
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
