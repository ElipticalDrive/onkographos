# Reparameterized Volume Sampling

Also called: Differentiable Volume Sampling, Inverse CDF Sampling

## Overview
Reparameterized Volume Sampling is a differentiable volume rendering technique that computes color estimates by sampling points on a ray using the inverse of the cumulative density function (CDF), allowing gradients to back-propagate through the sampling locations with respect to the density field [S1, unofficial]. Unlike NeRF's hierarchical sampling which uses an auxiliary coarse model trained separately, this method learns sampling points end-to-end without auxiliary networks [S1, unofficial]. Compared to standard importance sampling with uniform distribution, this stratified reparameterized sampling exhibits lower variance [S1, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   Ray segment bounds $[t_{r_1}, t_{r_2}]$.
*   Density field $\sigma_{\mathbf{r}}(t)$ parameterized by neural network weights.
*   Radiance field $c_{\mathbf{r}}(t)$ parameterized by neural network weights.
*   Number of samples per ray $k$ (typically 32) [S1, unofficial].
*   Number of knots $m$ for piecewise linear density approximation (typically 256) [S1, unofficial].

**Outputs:**
*   Rendered color $C(\mathbf{r})$.
*   Gradients $\nabla_{\theta} C(\mathbf{r})$ with respect to network parameters $\theta$.

**Data Structures:**
*   **Knots:** A set of $m+1$ depth values $t_0, t_1, \dots, t_m$ spanning the ray segment $[t_{r_1}, t_{r_2}]$. In the source, these are constructed by slightly perturbing a uniform ray grid [S1, unofficial].
*   **Density Approximation:** The density $\sigma_{\mathbf{r}}(t)$ is approximated as piecewise linear between knots. For $t \in [t_i, t_{i+1}]$:
    $$ \sigma_{\mathbf{r}}(t) \approx \sigma_{\mathbf{r}}(t_i) + \frac{\sigma_{\mathbf{r}}(t_{i+1}) - \sigma_{\mathbf{r}}(t_i)}{t_{i+1} - t_i} (t - t_i) $$
*   **CDF Approximation:** The cumulative distribution function $F_{\mathbf{r}}(t)$ is approximated as piecewise quadratic based on the piecewise linear density [S1, unofficial].

**Algorithm Steps:**
1.  **Evaluate Density at Knots:** Compute $\sigma_{\mathbf{r}}(t_i)$ for all $i \in \{0, \dots, m\}$ using the neural network. The network output activation must be Softplus with $\beta=10$ to avoid zero gradients [S1, unofficial].
2.  **Construct CDF:** Compute the cumulative opacity integral $I(t) = \int_{t_{r_1}}^t \sigma_{\mathbf{r}}(s) ds$ at each knot. Since $\sigma_{\mathbf{r}}$ is piecewise linear, $I(t)$ is piecewise quadratic. The CDF is $F_{\mathbf{r}}(t) = 1 - \exp(-I(t))$ [S1, unofficial].
3.  **Generate Random Samples:** For each sample $j \in \{1, \dots, k\}$, draw a uniform random variable $u_j \sim U[0, 1]$ [S1, unofficial].
4.  **Target Opacity:** Compute the target cumulative opacity value $y_j = F_{\mathbf{r}}(t_{r_2}) \cdot u_j$. Note: The source uses $y_f u$ where $y_f$ represents the total opacity of the segment [S1, unofficial].
5.  **Find Bin:** Identify the interval $[t_n, t_{n+1}]$ such that $F_{\mathbf{r}}(t_n) \le y_j < F_{\mathbf{r}}(t_{n+1})$. This is done by evaluating the CDF approximation at the knots [S1, unofficial].
6.  **Solve for $t$:** Solve the equation $-\log(1 - y_j) = \int_{t_n}^t \sigma_{\mathbf{r}}(s) ds$ for $t$ within the identified bin. Since $\sigma_{\mathbf{r}}$ is linear in the bin, the integral is quadratic in $t$. The source provides explicit formulas for solving this quadratic equation, ensuring numerical stability [S1, unofficial]. The solution $t_j$ is a differentiable function of the density field $\sigma_{\mathbf{r}}$ [S1, unofficial].
7.  **Evaluate Radiance:** Compute radiance $c_{\mathbf{r}}(t_j)$ at the sampled position $t_j$ [S1, unofficial].
8.  **Accumulate Color:** Compute the color estimate using the standard volume rendering integral approximation with the sampled points and their associated weights derived from the CDF differences [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def reparameterized_volume_sampling(ray_origin, ray_direction, t_near, t_far, 
                                    density_net, radiance_net, k=32, m=256):
    """
    Renders a ray using reparameterized volume sampling.
    
    Args:
        ray_origin: (3,) array
        ray_direction: (3,) array
        t_near: float
        t_far: float
        density_net: function mapping (N, 3) positions to (N,) densities
        radiance_net: function mapping (N, 3) positions to (N, 3) radiance
        k: int, number of samples
        m: int, number of knots for density approximation
        
    Returns:
        color: (3,) array
        gradients: dict of gradients w.r.t. network parameters
    """
    # 1. Create knots
    # Slightly perturb uniform grid for stability
    t_knots = np.linspace(t_near, t_far, m + 1)
    # Perturbation [G]: Add small random noise to avoid degenerate cases
    t_knots += np.random.uniform(-1e-6, 1e-6, size=t_knots.shape)
    
    # 2. Evaluate density at knots
    # Positions along the ray
    positions_knots = ray_origin + t_knots[:, None] * ray_direction
    sigma_knots = density_net(positions_knots)
    # Ensure non-negative density [G]: Softplus activation handles this, but clamp for safety
    sigma_knots = np.maximum(sigma_knots, 1e-6)
    
    # 3. Compute CDF at knots
    # Integral of piecewise linear density is piecewise quadratic
    # I(t) = integral from t_near to t of sigma(s) ds
    # For interval [t_i, t_{i+1}], sigma(s) = sigma_i + (sigma_{i+1}-sigma_i)/(t_{i+1}-t_i) * (s - t_i)
    # Integral from t_near to t_{i+1} can be computed cumulatively
    
    cum_integral = np.zeros(m + 1)
    for i in range(m):
        t_i = t_knots[i]
        t_ip1 = t_knots[i+1]
        sigma_i = sigma_knots[i]
        sigma_ip1 = sigma_knots[i+1]
        
        # Area of trapezoid under linear density
        area = 0.5 * (sigma_i + sigma_ip1) * (t_ip1 - t_i)
        cum_integral[i+1] = cum_integral[i] + area
        
    # CDF F(t) = 1 - exp(-I(t))
    # Normalize CDF to [0, 1] relative to total opacity
    total_opacity = 1.0 - math.exp(-cum_integral[-1])
    if total_opacity < 1e-6:
        # Degenerate case: no density
        return np.zeros(3), {}
        
    cdf_knots = 1.0 - np.exp(-cum_integral)
    cdf_knots /= total_opacity # Normalize to [0, 1]
    
    # 4. Generate samples
    u = np.random.uniform(0, 1, k)
    target_cdf = cdf_knots[-1] * u # y_f * u
    
    # 5. Find bins and solve for t
    t_samples = np.zeros(k)
    
    for j in range(k):
        y = target_cdf[j]
        
        # Find bin index n such that cdf_knots[n] <= y < cdf_knots[n+1]
        # Use searchsorted for efficiency
        n = np.searchsorted(cdf_knots, y, side='right') - 1
        n = np.clip(n, 0, m - 1)
        
        t_n = t_knots[n]
        t_np1 = t_knots[n+1]
        cdf_n = cdf_knots[n]
        cdf_np1 = cdf_knots[n+1]
        
        sigma_n = sigma_knots[n]
        sigma_np1 = sigma_knots[n+1]
        
        # We need to solve: -log(1 - y * total_opacity) = integral from t_n to t of sigma(s) ds
        # Let target_integral = -log(1 - y * total_opacity)
        # Note: cdf_knots[n] = 1 - exp(-cum_integral[n]) / total_opacity
        # So 1 - cdf_knots[n] = exp(-cum_integral[n]) / total_opacity
        # -log(1 - cdf_knots[n]) = cum_integral[n] - log(total_opacity)
        
        # Target integral relative to t_n
        # Integral from t_near to t is cum_integral[t]
        # We want cum_integral[t] - cum_integral[t_n] = -log(1 - y * total_opacity)
        # Let target_val = -log(1 - y * total_opacity) + cum_integral[t_n]
        
        target_val = -math.log(1.0 - y * total_opacity) + cum_integral[n]
        
        # In bin [t_n, t_np1], sigma(s) is linear
        # Integral from t_n to t is:
        # sigma_n * (t - t_n) + 0.5 * (sigma_np1 - sigma_n)/(t_np1 - t_n) * (t - t_n)^2
        # Let dt = t - t_n
        # A * dt^2 + B * dt - target_val = 0
        # A = 0.5 * (sigma_np1 - sigma_n) / (t_np1 - t_n)
        # B = sigma_n
        
        dt = t_np1 - t_n
        if dt < 1e-8:
            t_samples[j] = t_n
            continue
            
        A = 0.5 * (sigma_np1 - sigma_n) / dt
        B = sigma_n
        
        # Solve A * dt^2 + B * dt - target_val = 0
        # dt = (-B + sqrt(B^2 + 4*A*target_val)) / (2*A) if A != 0
        # If A == 0, linear: dt = target_val / B
        
        if abs(A) < 1e-8:
            dt_sol = target_val / B
        else:
            discriminant = B*B + 4*A*target_val
            if discriminant < 0:
                # Numerical issue, fallback
                dt_sol = 0.0
            else:
                dt_sol = (-B + math.sqrt(discriminant)) / (2*A)
                
        t_samples[j] = t_n + dt_sol
        
    # 6. Evaluate radiance and accumulate
    positions_samples = ray_origin + t_samples[:, None] * ray_direction
    radiance_samples = radiance_net(positions_samples)
    
    # Compute weights based on CDF differences
    # Sort samples to compute intervals correctly
    sorted_indices = np.argsort(t_samples)
    t_sorted = t_samples[sorted_indices]
    radiance_sorted = radiance_samples[sorted_indices]
    
    # Add boundaries
    t_all = np.concatenate([[t_near], t_sorted, [t_far]])
    
    # Compute opacity for each segment
    # alpha_i = exp(-integral(t_i, t_{i+1})) * (1 - exp(-integral(t_i, t_{i+1})))
    # This is complex for arbitrary points. 
    # Standard volume rendering: C = sum T_i * alpha_i * c_i
    # T_i = exp(-integral(t_near, t_i))
    # alpha_i = 1 - exp(-integral(t_i, t_{i+1}))
    
    # Approximate integral using trapezoidal rule between sorted samples
    # This is an approximation. The source implies using the CDF directly for weights.
    # Weight w_i = F(t_{i+1}) - F(t_i)
    
    # Recompute CDF at sample points for accurate weights
    # This requires evaluating density at sample points and integrating
    # For simplicity and consistency with source's "sparse set of points" claim:
    # We use the CDF values at the sorted sample points.
    
    # Evaluate density at sample points
    sigma_samples = density_net(positions_samples)
    sigma_samples = np.maximum(sigma_samples, 1e-6)
    
    # Compute cumulative integral at sample points
    # Integrate from t_near to t_j
    cum_integral_samples = np.zeros(k)
    for j in range(k-1):
        t_j = t_sorted[j]
        t_jp1 = t_sorted[j+1]
        sigma_j = sigma_samples[sorted_indices[j]]
        sigma_jp1 = sigma_samples[sorted_indices[j+1]]
        dt = t_jp1 - t_j
        area = 0.5 * (sigma_j + sigma_jp1) * dt
        cum_integral_samples[j+1] = cum_integral_samples[j] + area
        
    # CDF at samples
    cdf_samples = 1.0 - np.exp(-cum_integral_samples)
    cdf_samples /= total_opacity
    
    # Weights
    weights = np.diff(cdf_samples)
    # Handle first and last segments if necessary, but source focuses on samples
    
    # Color accumulation
    # C = sum w_j * c_j
    color = np.sum(weights[:, None] * radiance_sorted, axis=0)
    
    return color, {} # Gradients handled by autograd framework
```

## Operating characteristics
*   **Time Complexity:** $O(m + k \log m)$ or $O(m + k)$ depending on bin search implementation, where $m$ is the number of knots and $k$ is the number of samples. Evaluating density and radiance at knots and samples dominates.
*   **Memory Complexity:** $O(m + k)$ to store knot densities, CDF values, and sample positions.
*   **Measured Performance:** In the 'foggy' density field test, 32 samples matched the baseline performance of 128 samples [S1, unofficial]. In 'Glass and wall' and 'Wall' density fields, 4 samples matched the baseline performance [S1, unofficial].
*   **Strengths:** Lower variance compared to naive importance sampling with uniform distribution [S1, unofficial]. End-to-end differentiable without auxiliary networks [S1, unofficial].
*   **Limitations:** Requires piecewise linear approximation of density, which may introduce artifacts if $m$ is too small [S1, unofficial]. Piecewise constant approximation was faster but delivered worse rendering quality [S1, unofficial].
*   **When to use:** When differentiable sampling is required and variance reduction is important. Suitable for 3D scene reconstruction [S1, unofficial].
*   **When not to use:** When computational resources are extremely limited and $m$ cannot be kept large enough for accurate CDF approximation.

## Background
Reparameterized Volume Sampling was introduced in the paper "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It is part of the family of differentiable volume rendering methods that use the reparameterization trick to estimate gradients [S1, unofficial]. The method was applied to 3D scene reconstruction using synthetic data from NeRF [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
