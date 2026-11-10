# Reparameterized Monte Carlo Volume Rendering

Also called: R/MC, R/MC estimate, R/SMC estimate, Reparameterized Stratified Monte Carlo (R/SMC) Volume Rendering

## Overview
Reparameterized Monte Carlo Volume Rendering (R/MC) is a differentiable volume rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick [S1, unofficial]. It solves the problem of computing unbiased gradients for volume rendering integrals where the sampling distribution depends on the scene parameters (density field), which causes standard differentiation to fail due to moving boundaries or support [S1, unofficial]. It sets itself apart from its siblings by explicitly inverting the cumulative opacity function to sample ray distances, thereby decoupling the random variable generation from the density field $\sigma$ [S1, unofficial]. Unlike [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) which may use different sampling strategies, R/MC focuses on the inverse CDF approach for stratified or uniform sampling.

## How it works
**Inputs:**
- Ray $r = o + td$ with near bound $t_n$ and far bound $t_f$.
- Density field $\sigma_r(t)$, queried at discrete points.
- Radiance field $c_r(t)$, queried at discrete points.
- Number of samples $k$.

**Outputs:**
- Estimated radiance $\hat{C}_{MC}^R(r)$ or $\hat{C}_{SMC}^R(r)$.

**Data Structures:**
- The density field $\sigma$ is typically stored as a neural network or a discrete grid. In practice, $\sigma_r(t)$ is only queried at specific points, so analytical integration is impossible [S1, unofficial].
- The opacity function $F_r(t)$ and its inverse $F_r^{-1}(y)$ must be approximated numerically [S1, unofficial].

**Steps:**
1. Define the ray $r = o + td$ with bounds $t_n, t_f$.
2. Define the density field $\sigma$ and radiance field $c$.
3. Compute the total opacity $y_f = F_r(t_f) = 1 - \exp\left(-\int_{t_n}^{t_f} \sigma_r(s) ds\right)$ [S1, unofficial]. Note that $y_f$ represents the overall pixel opaqueness and is generally not equal to 1 [S1, unofficial].
4. Generate $k$ independent and identically distributed (i.i.d.) samples $u_i \sim U[0,1]$ for R/MC, or stratified samples $v_i$ for R/SMC.
5. For each sample, compute the target cumulative opacity $y_i = y_f u_i$ (for R/MC) or $y_i = y_f v_i$ (for R/SMC).
6. Solve for the ray distance $t_i$ such that $F_r(t_i) = y_i$. This requires inverting the opacity function [S1, unofficial].
7. Query the radiance $c_r(t_i)$ at the computed distance.
8. Compute the estimate $\hat{C} = \frac{y_f}{k} \sum_{i=1}^k c_r(t_i)$ [S1, unofficial].

**Formulas:**
The rendering integral is transformed by changing integration boundaries to $[0, 1]$ using the substitution $y = y_f u$ where $u \sim U[0,1]$ [S1, unofficial]:
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
Both estimates are unbiased, and their gradients are unbiased estimates of the gradient of the expected color [S1, unofficial].

**Parameters:**
- $k$: Number of samples per ray. Typical values are not specified in the sources, but increasing $k$ reduces variance [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method cannot compute $F_r$ analytically in practice because $\sigma_r$ is only queried at discrete points [S1, unofficial]. Approximations of $F_r$ and its inverse $F_r^{-1}$ are required [S1, unofficial].
- Piecewise constant or piecewise linear approximations of the density field are used to facilitate opacity inversion [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_mc_volume_rendering(ray_o, ray_d, t_n, t_f, density_field, radiance_field, k):
    """
    Estimates radiance using Reparameterized Monte Carlo Volume Rendering.
    
    Args:
        ray_o: Origin of the ray.
        ray_d: Direction of the ray.
        t_n: Near bound of the ray.
        t_f: Far bound of the ray.
        density_field: Function or structure to query density sigma(t).
        radiance_field: Function or structure to query radiance c(t).
        k: Number of samples.
        
    Returns:
        Estimated radiance.
    """
    # Step 1-3: Compute total opacity y_f
    # Note: In practice, F_r(t_f) must be approximated numerically because
    # sigma is only available at discrete points.
    # We assume a helper function compute_opacity(t_n, t_f, density_field) exists
    # which integrates sigma along the ray.
    y_f = compute_opacity(t_n, t_f, density_field)
    
    # Step 4: Generate k i.i.d. uniform samples
    u_samples = random_uniform(k)
    
    # Step 5-6: Map to opacity space and invert to get t
    # Note: F_r^{-1}(y) must be approximated numerically.
    # We assume a helper function invert_opacity(y, t_n, t_f, density_field) exists
    # which solves F_r(t) = y for t.
    t_samples = []
    for u in u_samples:
        y_target = y_f * u
        t_i = invert_opacity(y_target, t_n, t_f, density_field)
        t_samples.append(t_i)
        
    # Step 7-8: Query radiance and compute estimate
    radiance_sum = 0.0
    for t_i in t_samples:
        c_i = radiance_field(ray_o, ray_d, t_i)
        radiance_sum += c_i
        
    estimate = (y_f / k) * radiance_sum
    return estimate

def reparameterized_smc_volume_rendering(ray_o, ray_d, t_n, t_f, density_field, radiance_field, k):
    """
    Estimates radiance using Reparameterized Stratified Monte Carlo Volume Rendering.
    """
    y_f = compute_opacity(t_n, t_f, density_field)
    
    # Step 4: Generate stratified samples
    v_samples = []
    for i in range(1, k + 1):
        lower = (i - 1) / (k + 1)
        upper = i / (k + 1)
        v_i = random_uniform(lower, upper)
        v_samples.append(v_i)
        
    t_samples = []
    for v in v_samples:
        y_target = y_f * v
        t_i = invert_opacity(y_target, t_n, t_f, density_field)
        t_samples.append(t_i)
        
    radiance_sum = 0.0
    for t_i in t_samples:
        c_i = radiance_field(ray_o, ray_d, t_i)
        radiance_sum += c_i
        
    estimate = (y_f / k) * radiance_sum
    return estimate
```

## Subtopics

### Reparameterized Stratified Monte Carlo (R/SMC)
Uses stratified sampling to reduce variance compared to R/MC's uniform sampling, while maintaining the same differentiability properties [S1, unofficial]. See [Reparameterized Stratified Monte Carlo Volume Rendering](3309-reparameterized-volume-sampling.md).

## Operating characteristics
- **Time Complexity:** Depends on the cost of querying the density and radiance fields and the numerical inversion of the opacity function. Each sample requires one radiance query and one opacity inversion [S1, unofficial].
- **Memory Complexity:** Depends on the storage of the density and radiance fields [S1, unofficial].
- **Strengths:** Allows end-to-end learning with gradient descent without auxiliary models. It alleviates the costs associated with the color component of the neural radiance field at the additional cost of the density sampling algorithm [S1, unofficial].
- **Limitations:** Requires numerical approximation of $F_r$ and $F_r^{-1}$, which can be computationally expensive and introduce approximation errors [S1, unofficial].
- **When to use:** When differentiable volume rendering is needed and the density field is available, particularly for optimizing neural radiance fields [S1, unofficial].

## Background
The method was proposed in "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It builds on the reparameterization trick to enable gradient-based optimization of volume rendering integrals. The approach allows optimizing a neural radiance field with just a few radiance field evaluations per ray [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
