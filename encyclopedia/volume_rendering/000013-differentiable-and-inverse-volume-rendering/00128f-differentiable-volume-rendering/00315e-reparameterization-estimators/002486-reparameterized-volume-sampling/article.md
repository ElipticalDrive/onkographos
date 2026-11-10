# Reparameterized Volume Sampling

Also called: Differentiable Volume Sampling, Inverse CDF Sampling

## Overview
Reparameterized Volume Sampling is a differentiable rendering technique that computes color estimates by sampling points on a ray using the inverse of the cumulative density function (CDF), allowing gradients to back-propagate through the sampling locations with respect to the density field [S1, unofficial]. Unlike NeRF's hierarchical sampling which uses an auxiliary coarse model trained separately, this method learns sampling points end-to-end without auxiliary networks [S1, unofficial]. Compared to standard importance sampling with uniform distribution, this stratified reparameterized sampling exhibits lower variance [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Near and far clipping distances $t_n$ and $t_f$.
- Neural density field $\sigma_{\mathbf{r}}(t)$ parameterized by $\theta$.
- Number of samples per ray $k$ (typically 32) [S1, unofficial].
- Number of knots $m$ for piecewise linear approximation (typically 256) [S1, unofficial].

**Outputs:**
- Estimated color $C(\mathbf{r})$ along the ray.
- Gradients $\nabla_\theta C(\mathbf{r})$ with respect to the density field parameters.

**Data Structures:**
- **Knots:** A set of $m+1$ points $t_0, t_1, \dots, t_m$ uniformly distributed in $[t_n, t_f]$, optionally perturbed to avoid aliasing [S1, unofficial].
- **Density Values:** $\sigma_i = \sigma_{\mathbf{r}}(t_i)$ evaluated at each knot.
- **CDF Approximation:** A piecewise quadratic function $F_r(t)$ constructed from the piecewise linear density approximation.

**Method:**
1.  **Density Approximation:** Approximate the density field $\sigma_{\mathbf{r}}(t)$ along the ray as a piecewise linear function between knots $t_i$ and $t_{i+1}$:
    $$
    \sigma_{\mathbf{r}}(t) \approx \sigma_i + \frac{\sigma_{i+1} - \sigma_i}{t_{i+1} - t_i} (t - t_i) \quad \text{for } t \in [t_i, t_{i+1}]
    $$
    [S1, unofficial]

2.  **CDF Construction:** The cumulative distribution function is defined as:
    $$
    F_r(t) = 1 - \exp\left(-\int_{t_n}^t \sigma_{\mathbf{r}}(s) ds\right)
    $$
    [S1, unofficial]
    Using the piecewise linear density, the integral $\int_{t_n}^t \sigma_{\mathbf{r}}(s) ds$ becomes a piecewise quadratic function. Let $I(t) = \int_{t_n}^t \sigma_{\mathbf{r}}(s) ds$. Then $F_r(t) = 1 - e^{-I(t)}$.

3.  **Inverse CDF Sampling:** To sample a point $t$, generate a random number $u \in [0, 1]$. The target cumulative opacity value is $y_f u$, where $y_f = F_r(t_f)$ is the total opacity of the ray segment [S1, unofficial]. Solve for $t$ in:
    $$
    y_f u = F_r(t) = 1 - \exp\left(-\int_{t_n}^t \sigma_{\mathbf{r}}(s) ds\right)
    $$
    [S1, unofficial]
    Rewrite the equation to isolate the integral:
    $$
    -\log(1 - y_f u) = \int_{t_n}^t \sigma_{\mathbf{r}}(s) ds = I(t)
    $$
    [S1, unofficial]

4.  **Bin Search:** Find the interval $[t_i, t_{i+1}]$ such that $I(t_i) \le -\log(1 - y_f u) < I(t_{i+1})$. This is done by evaluating the cumulative integral at the knots.

5.  **Root Finding:** Within the identified bin, solve for $t$. Since $\sigma_{\mathbf{r}}(t)$ is linear in the bin, $I(t)$ is quadratic. Let $t = t_i + \delta$, where $\delta \in [0, t_{i+1} - t_i]$. The integral over this segment is:
    $$
    \int_{t_i}^{t_i+\delta} \left( \sigma_i + \frac{\sigma_{i+1} - \sigma_i}{t_{i+1} - t_i} \delta' \right) d\delta' = \sigma_i \delta + \frac{\sigma_{i+1} - \sigma_i}{2(t_{i+1} - t_i)} \delta^2
    $$
    Let $A = \sigma_i$ and $B = \frac{\sigma_{i+1} - \sigma_i}{2(t_{i+1} - t_i)}$. The equation to solve is:
    $$
    B \delta^2 + A \delta - \left( -\log(1 - y_f u) - I(t_i) \right) = 0
    $$
    [S1, unofficial]
    Solve this quadratic equation for $\delta$ using the quadratic formula, selecting the positive root. Then $t = t_i + \delta$.

6.  **Radiance Evaluation:** Evaluate the radiance $c_{\mathbf{r}}(t)$ at the computed sample position $t$.

7.  **Accumulation:** Accumulate the color estimate using the standard volume rendering equation with the sampled points. For $k$ samples, the color is estimated as:
    $$
    \hat{C}(\mathbf{r}) = \sum_{j=1}^k c_{\mathbf{r}}(t_j) \alpha_j
    $$
    where $\alpha_j$ is the transmittance-weighted contribution derived from the CDF differences between samples.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_volume_sampling(ray_origin, ray_direction, t_near, t_far, density_field, k=32, m=256):
    # 1. Define knots
    knots = np.linspace(t_near, t_far, m + 1)
    
    # 2. Evaluate density at knots
    densities = density_field(knots)
    
    # 3. Compute cumulative integral I(t) at knots
    # I(t_i) = integral from t_near to t_i of sigma(s) ds
    # Using piecewise linear approximation, I(t) is piecewise quadratic
    integrals = np.zeros(m + 1)
    for i in range(1, m + 1):
        t_prev = knots[i-1]
        t_curr = knots[i]
        sigma_prev = densities[i-1]
        sigma_curr = densities[i]
        # Integral of linear function from t_prev to t_curr
        # Area = (sigma_prev + sigma_curr) / 2 * (t_curr - t_prev)
        integrals[i] = integrals[i-1] + (sigma_prev + sigma_curr) / 2 * (t_curr - t_prev)
        
    # Total opacity factor
    total_opacity_factor = 1 - np.exp(-integrals[-1])
    
    # 4. Generate samples
    u = np.random.uniform(0, 1, k)
    samples = []
    
    for u_val in u:
        # Target integral value
        target_integral = -np.log(1 - total_opacity_factor * u_val)
        
        # Find bin
        bin_idx = np.searchsorted(integrals, target_integral, side='right') - 1
        bin_idx = np.clip(bin_idx, 0, m - 1)
        
        t_i = knots[bin_idx]
        t_i1 = knots[bin_idx + 1]
        sigma_i = densities[bin_idx]
        sigma_i1 = densities[bin_idx + 1]
        I_i = integrals[bin_idx]
        
        # Solve quadratic: B * delta^2 + A * delta - (target - I_i) = 0
        A = sigma_i
        B = (sigma_i1 - sigma_i) / (2 * (t_i1 - t_i))
        C = -(target_integral - I_i)
        
        # Quadratic formula
        discriminant = A**2 - 4 * B * C
        if discriminant < 0:
            # Fallback to linear if numerical issues occur
            delta = C / A if A != 0 else 0
        else:
            delta = (-A + np.sqrt(discriminant)) / (2 * B) if B != 0 else C / A
            
        t_sample = t_i + delta
        samples.append(t_sample)
        
    # 5. Evaluate radiance and accumulate
    # Radiance evaluation depends on the specific radiance field implementation
    # Here we assume a function evaluate_radiance(t) exists
    colors = []
    for t in samples:
        c = evaluate_radiance(t)
        colors.append(c)
        
    # Simple accumulation for demonstration; actual implementation uses alpha compositing
    # based on the CDF differences between sorted samples
    return np.mean(colors) # Placeholder for proper accumulation
```

**Parameters:**
- $k=32$ samples per ray [S1, unofficial].
- $m=256$ knots for piecewise linear density approximation [S1, unofficial].
- Softplus activation with $\beta=10$ for the density field output [S1, unofficial].

**Edge Cases and Pitfalls:**
- Using ReLU activation for the density field can result in zero gradients, hindering training [S1, unofficial].
- Numerical stability in solving the quadratic equation must be handled, particularly when $B$ is close to zero [S1, unofficial].
- The piecewise constant approximation was found to be faster but delivered worse rendering quality compared to piecewise linear [S1, unofficial].

## Subtopics

### [Reparameterization](11557-reparameterization.md)
The foundational mathematical framework using the change of variables formula to eliminate boundary terms in differentiable rendering.

### [Reparameterization Estimators](6499-reparameterization-estimators.md)
The broader class of estimators that use differentiable transformations to enable gradient flow through stochastic sampling.

### [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md)
A related method that applies reparameterization to Monte Carlo volume rendering, potentially with different sampling strategies.

### [Boundary Path Integral Formulation](3389-boundary-path-integral-formulation.md)
An alternative approach that handles boundary terms explicitly rather than eliminating them via reparameterization.

## Operating characteristics
- **Time Complexity:** The method requires evaluating the density field at $m$ knots to build the CDF approximation, and then solving a quadratic equation for each of the $k$ samples. The complexity is $O(m + k)$.
- **Memory Complexity:** $O(m)$ to store the knot positions, density values, and cumulative integrals.
- **Measured Performance:** In the 'foggy' density field test, 32 samples matched the baseline performance of 128 samples. In the 'Glass and wall' and 'Wall' density fields, 4 samples matched the baseline performance [S1, unofficial].
- **Strengths:** Lower variance compared to naive importance sampling with uniform distribution [S1, unofficial]. End-to-end differentiable without auxiliary networks [S1, unofficial].
- **Limitations:** Requires the density field to be differentiable. The piecewise linear approximation introduces some error compared to exact integration, though it was found to provide better quality than piecewise constant [S1, unofficial].
- **When to use:** When differentiable sampling is required for optimization of the density field, and lower variance is desired compared to uniform sampling.
- **When not to use:** When computational resources are extremely limited and the piecewise linear approximation overhead is prohibitive, or when the density field is not differentiable.

## Background
Reparameterized Volume Sampling was introduced in the paper "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It builds on the principles of reparameterization estimators [S1, unofficial]. The method was applied to 3D scene reconstruction using synthetic data from NeRF [S1, unofficial]. It addresses the limitations of NeRF's hierarchical sampling by providing a differentiable alternative that does not require an auxiliary coarse model [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
