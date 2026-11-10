# Reparameterized Monte Carlo Volume Rendering

Also called: R/MC, R/MC estimate, R/SMC estimate, Reparameterized Stratified Monte Carlo (R/SMC) Volume Rendering

## Overview
Reparameterized Monte Carlo (R/MC) volume rendering is a differentiable rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick [S1, unofficial]. It solves the problem of computing gradients for volume rendering integrals where the sampling distribution depends on the scene parameters (density field), avoiding the complex boundary terms associated with the Reynolds Transport Theorem [S1, unofficial]. Unlike standard Monte Carlo volume rendering where samples are fixed in space, R/MC samples are generated in a normalized parameter space and mapped back to the ray, ensuring the sampling distribution is static with respect to the optimization variables [S1, unofficial]. It sets itself apart from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) by explicitly defining the sampling strategy via inverse opacity mapping rather than general domain transformations, and from [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) by operating on the volumetric integral directly without path tracing [S1, unofficial].

## How it works
**Inputs:** Ray $r = o + td$ with near/far bounds $t_n, t_f$; density field $\sigma(x)$; radiance field $c(x)$; number of samples $k$.
**Outputs:** Estimated radiance $\hat{C}$; gradient $\nabla_\theta \hat{C}$ with respect to parameters $\theta$ defining $\sigma$ and $c$.

**Data Structures:**
*   **Ray:** Defined by origin $o$, direction $d$, $t_n$, $t_f$.
*   **Density Field:** $\sigma_r(t) = \sigma(o + td)$.
*   **Radiance Field:** $c_r(t) = c(o + td)$.
*   **Opacity Function:** $F_r(t) = 1 - \exp\left(-\int_{t_n}^{t} \sigma_r(s) ds\right)$.
*   **Total Opacity:** $y_f = F_r(t_f)$.

**Steps:**
1.  Compute the total opacity $y_f = F_r(t_f)$ along the ray segment $[t_n, t_f]$ [S1, unofficial].
2.  Generate $k$ independent samples.
    *   For **R/MC**: Draw $u_i \sim U[0, 1]$ for $i=1, \dots, k$ [S1, unofficial].
    *   For **R/SMC**: Draw stratified samples $v_i \sim U\left[\frac{i-1}{k+1}, \frac{i}{k+1}\right]$ for $i=1, \dots, k$ [S1, unofficial].
3.  Compute the target cumulative opacity values: $y_i = y_f \cdot u_i$ (for R/MC) or $y_i = y_f \cdot v_i$ (for R/SMC) [S1, unofficial].
4.  Solve for the ray distance $t_i$ such that $F_r(t_i) = y_i$. This requires computing the inverse opacity function $F_r^{-1}(y_i)$ [S1, unofficial].
5.  Query the radiance at the sampled positions: $c_r(t_i)$ [S1, unofficial].
6.  Compute the estimate:
    $$
    \hat{C} = \frac{y_f}{k} \sum_{i=1}^{k} c_r(F_r^{-1}(y_f u_i)) \quad \text{(R/MC)}
    $$
    $$
    \hat{C} = \frac{y_f}{k} \sum_{i=1}^{k} c_r(F_r^{-1}(y_f v_i)) \quad \text{(R/SMC)}
    $$
    [S1, unofficial]

**Approximations:**
In practice, $F_r(t)$ cannot be computed analytically because $\sigma_r$ is queried at discrete points. $F_r$ and $F_r^{-1}$ must be approximated, typically using piecewise constant or piecewise linear density field approximations along the ray [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_mc_volume_rendering(ray_o, ray_d, t_n, t_f, sigma_field, c_field, k):
    """
    ray_o: origin (3,)
    ray_d: direction (3,)
    t_n: near bound (float)
    t_f: far bound (float)
    sigma_field: function sigma(t) -> density (float)
    c_field: function c(t) -> radiance (3,)
    k: number of samples (int)
    """
    # Step 1: Compute total opacity y_f
    # Note: In practice, this requires numerical integration of sigma_field
    # along the ray from t_n to t_f.
    y_f = compute_total_opacity(ray_o, ray_d, t_n, t_f, sigma_field)
    
    # Step 2: Generate samples
    # For R/MC:
    u_samples = random.uniform(0, 1, size=k)
    
    # For R/SMC, replace u_samples with stratified samples:
    # u_samples = random.uniform((i-1)/(k+1), i/(k+1)) for i in 1..k
    
    # Step 3: Compute target opacities
    y_targets = y_f * u_samples
    
    # Step 4: Inverse opacity mapping
    # Solve F_r(t_i) = y_target_i for t_i
    # This requires an approximation of F_r and its inverse.
    # Typically done via piecewise linear interpolation of the cumulative opacity.
    t_samples = inverse_opacity_mapping(y_targets, ray_o, ray_d, t_n, t_f, sigma_field)
    
    # Step 5: Query radiance
    radiance_samples = c_field(t_samples) # Shape (k, 3)
    
    # Step 6: Compute estimate
    estimate = (y_f / k) * np.sum(radiance_samples, axis=0)
    
    return estimate
```

**Subtopics:**
*   **R/SMC (Reparameterized Stratified Monte Carlo):** Uses stratified sampling to reduce variance compared to R/MC's uniform sampling, while maintaining the same differentiability properties [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k \cdot N)$, where $k$ is the number of samples and $N$ is the cost of evaluating the density/radiance fields and computing the opacity approximation per sample.
*   **Memory Complexity:** $O(N)$ for storing the density field and intermediate opacity values along the ray.
*   **Strengths:** Enables end-to-end gradient-based optimization of neural radiance fields with few radiance evaluations per ray [S1, unofficial]. Unbiased gradient estimates [S1, unofficial].
*   **Limitations:** Requires approximation of $F_r$ and $F_r^{-1}$, which can introduce bias if not handled carefully [S1, unofficial]. R/SMC reduces variance but requires more complex sampling logic [S1, unofficial].
*   **When to use:** When optimizing differentiable neural radiance fields and computational efficiency per ray is critical [S1, unofficial].

## Background
Reparameterized Monte Carlo volume rendering was introduced in the paper "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It addresses the challenges of differentiable volume rendering by reparameterizing the sampling distribution to align with the density field, allowing for efficient gradient computation without auxiliary models [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
