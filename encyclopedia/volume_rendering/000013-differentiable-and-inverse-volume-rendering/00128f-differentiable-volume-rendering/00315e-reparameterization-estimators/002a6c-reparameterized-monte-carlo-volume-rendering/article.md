# Reparameterized Monte Carlo Volume Rendering

Also called: R/MC, R/MC estimate, R/SMC estimate, Reparameterized Stratified Monte Carlo (R/SMC) Volume Rendering

## Overview
Reparameterized Monte Carlo Volume Rendering is a differentiable rendering method that estimates expected radiance by sampling from the density-induced probability distribution using the reparameterization trick, allowing end-to-end gradient-based optimization without auxiliary models [S1, unofficial]. It sets itself apart from standard Monte Carlo volume rendering by decoupling the randomness from the scene parameters via the inverse cumulative distribution function (CDF) of the opacity, ensuring that gradients flow directly through the sampling process [S1, unofficial]. Compared to its sibling [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md), which samples uniformly in volume space, R/MC samples in opacity space, concentrating samples in denser regions of the scene.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray $r = o + td$ defined by origin $o$ and direction $d$, near bound $t_n$, far bound $t_f$, density field $\sigma_r(t)$, and radiance field $c_r(t)$. Number of samples $k$.
*   **Outputs:** Estimated radiance $\hat{C}$ for the ray.

**Data Structures:**
*   **Density/Radiance Fields:** Typically neural networks or voxel grids. $\sigma_r(t)$ and $c_r(t)$ are queried at arbitrary $t$.
*   **Opacity Approximation:** Since $\sigma_r$ is queried at discrete points, $F_r(t)$ must be approximated. A common approach is piecewise constant or piecewise linear approximation of $\sigma_r$ along the ray segment $[t_n, t_f]$ [S1, unofficial].

**Steps:**
1.  **Define Ray Bounds:** Establish the integration interval $[t_n, t_f]$ for the ray $r$.
2.  **Compute Total Opacity:** Calculate the total opacity $y_f$ at the far bound:
    $$ y_f = F_r(t_f) = 1 - \exp\left(-\int_{t_n}^{t_f} \sigma_r(s) ds\right) $$
    In practice, this integral is approximated using the chosen density approximation (e.g., summing $\sigma \Delta t$ for piecewise constant) [S1, unofficial].
3.  **Generate Samples:**
    *   **For R/MC:** Generate $k$ independent and identically distributed (i.i.d.) samples $u_i \sim U[0, 1]$.
    *   **For R/SMC:** Generate stratified samples $v_i$ from regular grid bins: $v_i \sim U\left[\frac{i-1}{k+1}, \frac{i}{k+1}\right]$ for $i=1, \dots, k$.
4.  **Map to Opacity Space:** Compute the target cumulative opacity $y_i$ for each sample:
    $$ y_i = y_f \cdot u_i \quad (\text{or } y_f \cdot v_i \text{ for R/SMC}) $$
5.  **Inverse Opacity (Ray Distance):** Solve for the ray distance $t_i$ such that:
    $$ F_r(t_i) = y_i $$
    This requires inverting the opacity function. If $F_r$ is approximated as piecewise linear or constant, this inversion can be done analytically within each segment or via binary search [S1, unofficial].
6.  **Query Radiance:** Evaluate the radiance at the sampled positions: $c_r(t_i)$.
7.  **Compute Estimate:** Calculate the final radiance estimate:
    $$ \hat{C}_{MC}^R(r) := \frac{y_f}{k} \sum_{i=1}^k c_r(F_r^{-1}(y_f u_i)) $$
    For R/SMC:
    $$ \hat{C}_{SMC}^R(r) := \frac{y_f}{k} \sum_{i=1}^k c_r(F_r^{-1}(y_f v_i)) $$

**Formulas:**
The core reparameterized integral transformation is:
$$ \int_{y_n}^{y_f} c_r(F_r^{-1}(y)) dy = \int_0^1 y_f c_r(F_r^{-1}(y_f u)) du $$
where $y_n = 0$ and $u \sim U[0,1]$ [S1, unofficial].

**Parameters:**
*   $k$: Number of samples per ray. Typical values depend on the desired noise level and computational budget. R/SMC often achieves lower variance for the same $k$ compared to R/MC [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Analytical Inversion:** The method cannot compute $F_r$ and $F_r^{-1}$ analytically in practice because $\sigma_r$ is only queried at discrete points. Approximations are required, which can introduce bias or gradient discontinuities if not handled carefully [S1, unofficial].
*   **Zero Density:** If $\sigma_r(t) = 0$ everywhere, $y_f = 0$, and the estimate is 0. Gradients may vanish.
*   **Precision:** Numerical stability in computing $1 - \exp(-\tau)$ for large optical thickness $\tau$ should be considered.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_mc_volume_rendering(ray_origin, ray_direction, t_near, t_far, 
                                        density_field, radiance_field, k, 
                                        sampling_mode='mc'):
    """
    ray_origin: (3,) float array
    ray_direction: (3,) float array
    t_near: float
    t_far: float
    density_field: callable sigma(t) -> float
    radiance_field: callable c(t) -> (3,) float
    k: int, number of samples
    sampling_mode: 'mc' or 'smc'
    """
    # Step 1: Approximate total opacity y_f
    # Using simple Riemann sum for approximation [S1, unofficial]
    num_integration_steps = 100
    dt = (t_far - t_near) / num_integration_steps
    optical_depth = 0.0
    for i in range(num_integration_steps):
        t = t_near + (i + 0.5) * dt
        sigma = density_field(t)
        optical_depth += sigma * dt
    
    y_f = 1.0 - math.exp(-optical_depth)
    
    # Step 2: Generate samples
    if sampling_mode == 'mc':
        u_samples = random.uniform(0, 1, k)
    elif sampling_mode == 'smc':
        # Stratified sampling
        u_samples = random.uniform(0, 1, k)
        # Sort to ensure stratification order, though order doesn't matter for sum
        u_samples.sort()
        # Rescale to bins [i/(k+1), (i+1)/(k+1)]
        # Note: The source uses [ (i-1)/(k+1), i/(k+1) ] for i=1..k
        # This is equivalent to uniform samples within those bins.
        # Implementation: generate uniform in each bin
        u_samples = []
        for i in range(1, k + 1):
            lower = (i - 1) / (k + 1)
            upper = i / (k + 1)
            u_samples.append(random.uniform(lower, upper))
        u_samples = np.array(u_samples)
    else:
        raise ValueError("Unknown sampling mode")

    # Step 3: Map to opacity space
    y_samples = y_f * u_samples

    # Step 4: Invert opacity to get t samples
    # This requires a more sophisticated inversion than simple Riemann sum
    # if high accuracy is needed. Here we assume a helper function exists
    # that inverts the opacity approximation used for y_f.
    # For a simple piecewise constant approximation, this can be done analytically.
    t_samples = invert_opacity(y_samples, t_near, t_far, density_field)

    # Step 5: Query radiance and compute estimate
    radiance_sum = np.zeros(3)
    for t in t_samples:
        c = radiance_field(t)
        radiance_sum += c
    
    estimate = (y_f / k) * radiance_sum
    return estimate

def invert_opacity(y_samples, t_near, t_far, density_field):
    """
    Approximate inversion of opacity.
    Assumes piecewise constant density for simplicity in this example.
    In practice, a more robust solver or precomputed lookup table is used.
    """
    t_samples = np.zeros_like(y_samples)
    # Simple binary search or analytical inversion based on density approximation
    # This is a placeholder for the complex inversion logic described in [S1, unofficial]
    for i, y in enumerate(y_samples):
        if y == 0:
            t_samples[i] = t_near
            continue
        # Binary search for t such that F_r(t) = y
        t_low = t_near
        t_high = t_far
        for _ in range(50): # 50 iterations for precision
            t_mid = (t_low + t_high) / 2
            # Compute F_r(t_mid)
            dt = (t_mid - t_near) / 100
            optical_depth = 0.0
            for j in range(100):
                t = t_near + (j + 0.5) * dt
                optical_depth += density_field(t) * dt
            f_r_t_mid = 1.0 - math.exp(-optical_depth)
            
            if f_r_t_mid < y:
                t_low = t_mid
            else:
                t_high = t_mid
        t_samples[i] = (t_low + t_high) / 2
    return t_samples
```

## Subtopics

### Reparameterized Stratified Monte Carlo (R/SMC)
Uses stratified sampling $v_i \sim U[\frac{i-1}{k+1}, \frac{i}{k+1}]$ to reduce variance compared to R/MC, while maintaining the same differentiability properties [S1, unofficial]. Choose when lower variance is needed without increasing $k$.

### Reparameterized Volume Sampling
Samples uniformly in volume space rather than opacity space. Choose when density is uniform or when opacity-space inversion is difficult [3309-reparameterized-volume-sampling.md].

## Operating characteristics
*   **Time Complexity:** $O(k \cdot N_{inv})$ per ray, where $k$ is the number of samples and $N_{inv}$ is the cost of inverting the opacity function (e.g., binary search steps).
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, excluding the storage for the density/radiance fields.
*   **Performance:** R/SMC typically achieves lower variance than R/MC for the same $k$, allowing for fewer samples to reach a target noise level [S1, unofficial].
*   **Strengths:** Unbiased estimates for both radiance and gradients; efficient gradient computation via reparameterization; reduces cost of radiance field evaluations compared to standard path tracing [S1, unofficial].
*   **Limitations:** Requires approximation of $F_r$ and $F_r^{-1}$, which can be complex to implement and may introduce bias if not done carefully [S1, unofficial]. Performance depends heavily on the efficiency of the opacity inversion.

## Background
The method was introduced in "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It builds on the reparameterization trick for Monte Carlo integration, applying it to the volume rendering integral by changing variables from ray distance to cumulative opacity. This allows the gradient to flow through the sampling process, enabling end-to-end optimization of neural radiance fields.

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
