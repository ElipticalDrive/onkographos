# Reparameterized Monte Carlo Volume Rendering

Also called: R/MC, R/MC estimate, R/SMC estimate, Reparameterized Stratified Monte Carlo (R/SMC) Volume Rendering

## Overview
Reparameterized Monte Carlo Volume Rendering (R/MC) is a differentiable volume rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick [S1, unofficial]. It solves the problem of computing gradients for volume rendering integrals where the domain of integration (the ray segment) and the sampling distribution depend on the scene parameters (density field) [S1, unofficial]. Unlike standard Monte Carlo volume rendering, which suffers from discontinuous gradients when samples cross object boundaries, R/MC maps parameter-independent uniform random variables to ray distances via the inverse opacity function, ensuring the gradient estimator remains unbiased and differentiable [S1, unofficial]. It sets itself apart from [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) by explicitly normalizing the integration domain to opacity space $[0, y_f]$ and solving for ray distances through inverse CDF approximation, whereas [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md) handles boundaries via path integral formulations [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Ray $r = o + td$ with near/far bounds $t_n, t_f$; differentiable density field $\sigma_r(t)$; differentiable radiance field $c_r(t)$.
- **Output:** Estimated radiance $\hat{C}$ and its gradient $\nabla_\theta \hat{C}$.

**Data Structures:**
- **Ray:** Struct containing origin $o$, direction $d$, $t_n$, $t_f$.
- **Density/Radiance Fields:** Neural networks or voxel grids parameterized by $\theta$, supporting query $\sigma_r(t)$ and $c_r(t)$.
- **Samples:** Array of $k$ random variables.

**Steps:**
1.  **Compute Total Opacity:** Calculate the total opacity $y_f$ along the ray segment $[t_n, t_f]$. This requires approximating the integral of the density field.
    $$ y_f = F_r(t_f) = 1 - \exp\left(-\int_{t_n}^{t_f} \sigma_r(s) ds\right) $$
    Since $\sigma_r$ is queried at discrete points, $F_r(t)$ is approximated numerically (e.g., piecewise constant or linear integration) [S1, unofficial].

2.  **Generate Samples:** Generate $k$ independent and identically distributed (i.i.d.) samples.
    - For **R/MC**: $u_i \sim U[0, 1]$ for $i=1, \dots, k$.
    - For **R/SMC** (Reparameterized Stratified Monte Carlo): $v_i \sim U\left[\frac{i-1}{k+1}, \frac{i}{k+1}\right]$ for $i=1, \dots, k$. Stratified sampling reduces variance compared to uniform sampling while maintaining differentiability [S1, unofficial].

3.  **Map to Opacity Space:** Transform samples to the cumulative opacity domain.
    $$ y_i = y_f \cdot u_i \quad \text{(for R/MC)} $$
    $$ y_i = y_f \cdot v_i \quad \text{(for R/SMC)} $$
    This normalizes the integration boundaries to $[0, 1]$ in the transformed domain [S1, unofficial].

4.  **Inverse Opacity (Ray Distance):** Solve for the ray distance $t_i$ such that the cumulative opacity at $t_i$ equals $y_i$.
    $$ F_r(t_i) = y_i \implies t_i = F_r^{-1}(y_i) $$
    Since $F_r$ is approximated numerically, $F_r^{-1}$ is computed via root-finding (e.g., binary search) or analytical inversion of the piecewise approximation [S1, unofficial].

5.  **Query Radiance:** Evaluate the radiance at the sampled distances.
    $$ c_i = c_r(t_i) $$

6.  **Compute Estimate:** Calculate the final radiance estimate.
    $$ \hat{C} = \frac{y_f}{k} \sum_{i=1}^k c_r(t_i) $$

**Formulas:**
The R/MC estimate for a ray $r$ with $k$ i.i.d. uniform samples is [S1, unofficial]:
$$ \hat{C}_{MC}^R(r) := \frac{y_f}{k} \sum_{i=1}^k c_r(F_r^{-1}(y_f u_i)) $$

The R/SMC estimate with stratified samples is [S1, unofficial]:
$$ \hat{C}_{SMC}^R(r) := \frac{y_f}{k} \sum_{i=1}^k c_r(F_r^{-1}(y_f v_i)) $$

Both estimates are unbiased for the expected color $C(r)$, and their gradients are unbiased estimates of the gradient of $C(r)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_volume_rendering(ray, density_field, radiance_field, k, stratified=False):
    """
    ray: object with .origin, .direction, .t_near, .t_far
    density_field: callable sigma(t) returning density at distance t
    radiance_field: callable c(t) returning radiance at distance t
    k: number of samples
    stratified: boolean, if True use stratified sampling (R/SMC), else uniform (R/MC)
    """
    # Step 1: Approximate total opacity y_f
    # In practice, integrate sigma along the ray. 
    # Here we assume a helper function that computes F_r(t_far) numerically.
    y_f = compute_total_opacity(ray, density_field)
    
    # Step 2: Generate samples
    if stratified:
        # Stratified samples: v_i ~ U[(i-1)/(k+1), i/(k+1)]
        samples = []
        for i in range(1, k + 1):
            lower = (i - 1) / (k + 1)
            upper = i / (k + 1)
            # Sample uniformly within the bin
            u = random.uniform(lower, upper)
            samples.append(u)
    else:
        # Uniform samples: u_i ~ U[0, 1]
        samples = [random.uniform(0, 1) for _ in range(k)]
        
    # Step 3 & 4: Map to opacity and invert to ray distance
    # We need to solve F_r(t) = y_f * u_i for t
    # This requires an inverse opacity function F_r^{-1}
    # In practice, this is done via binary search on the piecewise approximation of F_r
    
    total_radiance = 0.0
    for u in samples:
        target_opacity = y_f * u
        
        # Find t such that F_r(t) = target_opacity
        # Assuming compute_inverse_opacity is implemented using binary search 
        # on the cumulative density integral
        t_sample = compute_inverse_opacity(ray, density_field, target_opacity)
        
        # Step 5: Query radiance
        radiance = radiance_field(t_sample)
        total_radiance += radiance
        
    # Step 6: Compute estimate
    estimate = (y_f / k) * total_radiance
    
    return estimate
```

**Parameters:**
- $k$: Number of samples per ray. Typical values are small (e.g., 1-4) to keep computational cost low, as the method is designed for end-to-end learning with few evaluations [S1, unofficial].
- Stratification: Using stratified sampling (R/SMC) is recommended to reduce variance [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Analytical Inversion:** The method cannot compute $F_r$ analytically in practice because $\sigma_r$ is only queried at discrete points. Approximations of $F_r$ and its inverse $F_r^{-1}$ are required [S1, unofficial].
- **Precision:** Numerical errors in the root-finding for $F_r^{-1}$ can introduce bias or gradient noise.
- **Empty Rays:** If $y_f \approx 0$, the estimate is 0. Care must be taken to avoid division by zero or undefined behavior in the inverse opacity step.

## Subtopics

### Reparameterized Stratified Monte Carlo (R/SMC)
A variant of R/MC that uses stratified sampling to reduce variance. It replaces uniform samples $u_i$ with stratified samples $v_i$ drawn from regular grid bins. It is chosen when lower variance is needed without increasing the number of samples significantly [S1, unofficial].

### Piecewise Density Approximation
A technique used to approximate the opacity function $F_r$ and its inverse. It involves discretizing the ray and approximating the density field as piecewise constant or linear. This is necessary because analytical inversion is not possible with discrete queries [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(k \cdot N)$, where $k$ is the number of samples and $N$ is the cost of computing the inverse opacity (e.g., binary search steps). Since $k$ is small, the method is efficient per ray.
- **Memory Complexity:** $O(N)$ for storing the density field and intermediate integration values.
- **Measured Performance:** The method allows optimization of neural radiance fields with just a few radiance field evaluations per ray, alleviating costs associated with the color component [S1, unofficial].
- **Strengths:** Unbiased gradient estimation; differentiable end-to-end; low sample count required.
- **Limitations:** Requires numerical approximation of opacity and its inverse; variance can be high with uniform sampling (mitigated by R/SMC).
- **When to Use:** When differentiable volume rendering is needed for gradient-based optimization of scene parameters (e.g., neural radiance fields).
- **When Not to Use:** When analytical opacity functions are available and standard Monte Carlo is sufficient, or when real-time rendering with very high sample counts is required.

## Background
Reparameterized Monte Carlo Volume Rendering was introduced in the paper "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It builds on the reparameterization trick to enable gradient-based optimization of volume rendering integrals. The method was developed to address the challenges of learning 3D scenes and generating novel views using differentiable rendering techniques. It is particularly relevant in the context of neural radiance fields (NeRFs) and other learning-based 3D representation methods.

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
