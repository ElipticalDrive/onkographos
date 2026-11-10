# Reparameterized Volume Sampling

Also called: Differentiable Volume Sampling, Inverse CDF Sampling

## Overview
Reparameterized Volume Sampling is a differentiable volume rendering technique that computes color estimates by sampling points on a ray using the inverse of the cumulative density function (CDF), allowing gradients to back-propagate through the sampling locations with respect to the density field [S1, unofficial]. It sets itself apart from NeRF's hierarchical sampling by learning sampling points end-to-end without auxiliary coarse models, and exhibits lower variance than standard importance sampling with uniform distribution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray $\mathbf{r}$ defined by origin $o$ and direction $d$, bounded by near $t_{r_1}$ and far $t_{r_n}$; a density field $\sigma_{\mathbf{r}}(t)$ parameterized by $\theta$; number of samples per ray $k$; number of knots $m$.
*   **Output:** A color estimate $C(\mathbf{r})$ and gradients $\nabla_\theta C(\mathbf{r})$.

**Data Structures:**
*   **Knots:** A set of $m$ points $t_1, \dots, t_m$ along the ray. In implementation, these are typically a uniform grid slightly perturbed to avoid degeneracy [S1, unofficial].
*   **Density Approximation:** The density $\sigma_{\mathbf{r}}(t)$ is approximated as piecewise linear between knots. For $t \in [t_i, t_{i+1}]$:
    $$ \sigma_{\mathbf{r}}(t) = \sigma_{\mathbf{r}}(t_i) + \frac{\sigma_{\mathbf{r}}(t_{i+1}) - \sigma_{\mathbf{r}}(t_i)}{t_{i+1} - t_i} (t - t_i) $$
*   **CDF Approximation:** The cumulative distribution function $F_{\mathbf{r}}(t)$ is approximated as piecewise quadratic. The integral of the piecewise linear density yields:
    $$ \int_{t_n}^t \sigma_{\mathbf{r}}(s) ds = I_0(t) + I_1(t) $$
    where $I_0$ is piecewise linear and $I_1$ is piecewise quadratic [S1, unofficial].

**Parameters:**
*   $k = 32$: Number of samples per ray [S1, unofficial].
*   $m = 256$: Number of knots for piecewise linear density approximation [S1, unofficial].
*   Activation: Softplus with $\beta = 10$ for the density field output [S1, unofficial].

**Steps:**
1.  **Evaluate Density:** Compute $\sigma_{\mathbf{r}}(t_i)$ for all knots $i=1 \dots m$.
2.  **Construct CDF:** Compute the cumulative opacity integral values at each knot. Let $A_i = \int_{t_{r_1}}^{t_i} \sigma_{\mathbf{r}}(s) ds$. Then $F_{\mathbf{r}}(t_i) = 1 - \exp(-A_i)$.
3.  **Generate Noise:** For each sample $j \in \{1, \dots, k\}$, draw a uniform random variable $u_j \sim \mathcal{U}[0, 1]$.
4.  **Target Opacity:** Compute the target cumulative opacity value $y_j = F_{\mathbf{r}}(t_{r_n}) \cdot u_j$. Note: $F_{\mathbf{r}}(t_{r_n})$ is the total opacity of the ray segment.
5.  **Find Bin:** Identify the interval $[t_n, t_{n+1}]$ such that $F_{\mathbf{r}}(t_n) \le y_j < F_{\mathbf{r}}(t_{n+1})$. This is done by evaluating the CDF approximation at the knots.
6.  **Invert CDF:** Solve for $t$ in the identified bin. The inversion equation is:
    $$ -\log(1 - y_j) = \int_{t_n}^t \sigma_{\mathbf{r}}(s) ds $$
    Since $\sigma_{\mathbf{r}}(s)$ is linear in $[t_n, t_{n+1}]$, the integral is a quadratic function of $t$. Solve this quadratic equation for $t$ within the bin bounds. The solution $t$ is a differentiable function of $\sigma_{\mathbf{r}}$ [S1, unofficial].
7.  **Evaluate Radiance:** Evaluate the radiance $c_{\mathbf{r}}(t)$ at the computed sample position $t$.
8.  **Accumulate:** Accumulate the color estimate using the standard volume rendering equation with the computed samples and their associated weights derived from the density [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reparameterized_volume_sampling(ray_origin, ray_direction, t_near, t_far, density_field, k=32, m=256):
    # 1. Define knots
    # Perturb uniform grid slightly to avoid degeneracy [S1, unofficial]
    t_knots = uniform_grid(t_near, t_far, m) + small_perturbation()
    
    # 2. Evaluate density at knots
    sigma_knots = density_field(t_knots) # Returns array of size m
    
    # 3. Compute cumulative opacity integral at knots
    # Integral of piecewise linear density is piecewise quadratic
    # A[i] = integral from t_near to t_knots[i] of sigma(s) ds
    A = compute_piecewise_quadratic_integral(t_knots, sigma_knots)
    
    # 4. Compute CDF at knots
    # F(t) = 1 - exp(-A(t))
    F_knots = 1.0 - torch.exp(-A)
    
    # 5. Total opacity
    F_total = F_knots[-1]
    
    samples = []
    for j in range(k):
        # 6. Generate noise
        u = torch.rand(1)
        
        # 7. Target cumulative opacity
        y_target = F_total * u
        
        # 8. Find bin
        # Find index n such that F_knots[n] <= y_target < F_knots[n+1]
        n = find_bin(F_knots, y_target)
        
        # 9. Invert CDF within bin [t_knots[n], t_knots[n+1]]
        # Equation: -log(1 - y_target) = integral(t_knots[n] to t) sigma(s) ds
        # sigma(s) is linear: sigma(s) = a*s + b
        # Integral is quadratic: Q(t) = c*t^2 + d*t + e
        # Solve Q(t) = -log(1 - y_target) for t in [t_knots[n], t_knots[n+1]]
        
        target_integral = -torch.log(1.0 - y_target)
        
        # Coefficients for the quadratic integral in this bin
        # Derived from piecewise linear interpolation of sigma
        sigma_n = sigma_knots[n]
        sigma_n1 = sigma_knots[n+1]
        t_n = t_knots[n]
        t_n1 = t_knots[n+1]
        
        # Linear density: sigma(s) = sigma_n + (sigma_n1 - sigma_n)/(t_n1 - t_n) * (s - t_n)
        # Let slope = (sigma_n1 - sigma_n) / (t_n1 - t_n)
        # Integral from t_n to t: sigma_n*(t-t_n) + 0.5*slope*(t-t_n)^2
        # Let dt = t - t_n
        # target_integral = sigma_n * dt + 0.5 * slope * dt^2
        # 0.5 * slope * dt^2 + sigma_n * dt - target_integral = 0
        
        slope = (sigma_n1 - sigma_n) / (t_n1 - t_n)
        a_quad = 0.5 * slope
        b_quad = sigma_n
        c_quad = -target_integral
        
        # Solve quadratic equation a*dt^2 + b*dt + c = 0
        dt = solve_quadratic(a_quad, b_quad, c_quad, lower_bound=0, upper_bound=t_n1 - t_n)
        
        t_sample = t_n + dt
        samples.append(t_sample)
        
    # 10. Evaluate radiance and accumulate
    # Standard volume rendering accumulation using samples
    color = accumulate_color(ray_origin, ray_direction, samples, density_field)
    
    return color
```

**Edge Cases and Pitfalls:**
*   **Zero Gradients:** Using ReLU activation for the density field results in zero gradients, hindering training. Softplus with $\beta=10$ is recommended [S1, unofficial].
*   **Degenerate Distributions:** Importance sampling requires only a few points for degenerate distributions, but stratified sampling is used to ensure robustness [S1, unofficial].
*   **Numerical Stability:** Explicit formulae for $t$ are provided in the supplementary materials of the source to ensure numerical stability during inversion [S1, unofficial].

## Subtopics

### Hierarchical Sampling (NeRF)
Uses an auxiliary coarse model trained separately. Unlike reparameterized volume sampling, it does not allow differentiating with respect to the coarse model parameters directly [S1, unofficial].

### Mip-NeRF
Merges coarse and fine models using a scale-aware neural field [S1, unofficial].

### Mip-NeRF 360
Distills the coarse density field from a fine field instead of training an auxiliary coarse radiance field [S1, unofficial].

### Unisurf
Treats the density field as an occupancy field and gradually incorporates root-finding algorithms [S1, unofficial].

## Operating characteristics
*   **Variance:** Stratified reparameterized sampling exhibits lower variance than naive importance sampling with uniform distribution [S1, unofficial].
*   **Efficiency:** In the 'foggy' density field test, 32 samples matched the baseline performance of 128 samples. In 'Glass and wall' and 'Wall' fields, 4 samples matched the baseline [S1, unofficial].
*   **Quality:** Piecewise linear approximation delivers better rendering quality than piecewise constant approximation, which is faster but lower quality [S1, unofficial].
*   **Comparison:** Comparable to NeRF without hierarchical sampling in terms of PSNR/LPIPS, but avoids the complexity of auxiliary networks [S1, unofficial].

## Background
Reparameterized Volume Sampling was introduced in "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It applies the reparameterization trick to volume rendering, allowing gradients to flow through the sampling process. It is used in 3D scene reconstruction based on image projections, benchmarked against NeRF [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
