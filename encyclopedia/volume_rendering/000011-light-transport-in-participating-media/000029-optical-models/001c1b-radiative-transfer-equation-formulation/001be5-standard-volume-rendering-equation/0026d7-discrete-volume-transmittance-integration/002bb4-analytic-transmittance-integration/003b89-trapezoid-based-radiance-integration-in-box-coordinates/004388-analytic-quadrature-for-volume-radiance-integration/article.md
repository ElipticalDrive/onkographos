# Analytic Quadrature for Volume Radiance Integration

Also called: Analytic integration for volume slices, Proposed quadrature method

## Overview
This method computes the radiance contribution of a volume slice by analytically integrating the product of transmittance and radiance, rather than approximating their product directly as in Newton-Cotes rules. It improves accuracy over the trapezoid rule for dense volumes with high extinction while remaining faster than Simpson's rule. It builds on [Trapezoid-based Radiance Integration in Box Coordinates](9001-trapezoid-based-radiance-integration-in-box-coordinates.md) by refining the integration step within each segment.

## How it works
**Inputs:**
- $N$: Number of intervals (slices) along the ray.
- $s_k$: Position of the $k$-th sample point, for $k=0 \dots N$.
- $L(s_k)$: Radiance at position $s_k$.
- $\bar{\sigma}_t(s_k)$: Extinction coefficient at position $s_k$.
- $\delta_k$: Length of interval $k$, defined as $|s_{k+1} - s_k|$. Typically uniform ($\delta_k = \delta$).

**Outputs:**
- $L(x_i, \omega_o)$: Total integrated radiance at point $x_i$ in direction $\omega_o$.

**Data Structures:**
- Arrays for $L$, $\bar{\sigma}_t$, and positions $s$.
- Scalar $\delta_k$.

**Steps:**

1. **Compute Average Extinction:** For each interval $k \in [0, N-1]$, calculate the average extinction coefficient $\bar{\sigma}_{t,k}$ using the arithmetic mean of the endpoints:
   $$ \bar{\sigma}_{t,k} = \frac{1}{2} (\bar{\sigma}_t(s_k) + \bar{\sigma}_t(s_{k+1})) $$
   The extinction coefficient is assumed piecewise constant within the interval to avoid complex error functions [S1].

2. **Compute Interval Radiance Contribution:** For each interval $k$, calculate the integrated radiance contribution $\bar{L}(k)$ using the analytic formula derived from piecewise linear radiance and piecewise constant extinction:
   $$ \bar{L}(k) = \frac{2L(s_k)}{\bar{\sigma}_{t,k}\delta_k} \left( e^{-\bar{\sigma}_{t,k}\delta_k} - 1 \right) + \frac{2L(s_{k+1})}{\bar{\sigma}_{t,k}\delta_k} \left( e^{\bar{\sigma}_{t,k}\delta_k} - 1 \right) e^{-\bar{\sigma}_{t,k}\delta_k} $$
   Note: If $\bar{\sigma}_{t,k} \to 0$, this formula approaches the trapezoid rule result. Implementation should handle the limit or use a small epsilon guard [G].

3. **Compute Cumulative Throughput:** Calculate the cumulative transmittance $T_r(k)$ up to the start of interval $k$. For $k=0$, $T_r(0) = 1$. For $k > 0$:
   $$ T_r(k) = \prod_{l=1}^{k} e^{-\bar{\sigma}_{t,l}\delta_l} $$
   This can be computed iteratively: $T_r(k) = T_r(k-1) \cdot e^{-\bar{\sigma}_{t,k-1}\delta_{k-1}}$.

4. **Sum Contributions:** The total radiance is the sum of the interval contributions weighted by the cumulative throughput:
   $$ L(x_i, \omega_o) = \sum_{k=0}^{N-1} \bar{L}(k) T_r(k) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytic_quadrature_radiance(N, s, L_vals, sigma_t_vals, delta):
    """
    Computes radiance using analytic quadrature for volume slices.
    
    Args:
        N: Number of intervals.
        s: Array of positions [s_0, ..., s_N].
        L_vals: Array of radiance values [L(s_0), ..., L(s_N)].
        sigma_t_vals: Array of extinction coefficients [sigma_t(s_0), ..., sigma_t(s_N)].
        delta: Interval length |s_{k+1} - s_k|. Assumed uniform.
        
    Returns:
        Total radiance L(x_i, omega_o).
    """
    total_radiance = 0.0
    cumulative_throughput = 1.0
    
    for k in range(N):
        # 1. Average extinction for interval k
        sigma_avg = 0.5 * (sigma_t_vals[k] + sigma_t_vals[k+1])
        
        # 2. Interval radiance contribution L_bar(k)
        # Handle near-zero extinction to avoid division by zero
        if sigma_avg < 1e-8:
            # Limit as sigma_avg -> 0 corresponds to trapezoid rule
            # Integral of linear L(s) over [0, delta] is delta * (L(0) + L(1)) / 2
            # Throughput is 1. So contribution is delta * (L_k + L_{k+1}) / 2
            # However, the formula below assumes throughput is factored out later.
            # The analytic form for L_bar(k) when sigma=0 is:
            # L_bar(k) = delta * (L_k + L_{k+1}) / 2
            l_bar_k = delta * (L_vals[k] + L_vals[k+1]) / 2.0
        else:
            term1 = (2.0 * L_vals[k] / (sigma_avg * delta)) * (math.exp(-sigma_avg * delta) - 1.0)
            term2 = (2.0 * L_vals[k+1] / (sigma_avg * delta)) * (math.exp(sigma_avg * delta) - 1.0) * math.exp(-sigma_avg * delta)
            l_bar_k = term1 + term2
            
        # 3. Add to total with current cumulative throughput
        total_radiance += l_bar_k * cumulative_throughput
        
        # 4. Update cumulative throughput for next interval
        # Throughput through interval k is exp(-sigma_avg * delta)
        cumulative_throughput *= math.exp(-sigma_avg * delta)
        
    return total_radiance
```

**Parameters:**
- $\bar{\sigma}_t(u)$: Piecewise constant approximation of extinction. This assumption is made to improve computational speed compared to piecewise linear extinction, which would require error functions of complex numbers [S1].
- $\delta_k$: Interval length. Typically uniform across all intervals [S1].

**Edge Cases and Pitfalls:**
- **Zero Extinction:** When $\bar{\sigma}_{t,k} \approx 0$, the division in the $\bar{L}(k)$ formula causes numerical instability. The method converges to the trapezoid rule in this limit [S1]. A guard clause or L'Hôpital's rule application is required [G].
- **High Extinction:** The method maintains accuracy for high extinction values where standard Newton-Cotes rules (trapezoid/Simpson) fail because they approximate the product $T_r(s)L(s)$ directly [S1].

## Subtopics

### Trapezoid-based Radiance Integration in Box Coordinates
- **What it is:** A method for integrating unoccluded radiance in a homogeneous finite medium by traversing the intersection of a light-view plane with a bounding box, decomposing the path into trapezoid segments.
- **Difference:** It uses simple trapezoidal approximation for the integral, whereas this method uses analytic integration per slice.
- **When to choose:** Use this parent method for simpler, homogeneous media or when computational speed is prioritized over accuracy in dense media.
- **Link:** [Trapezoid-based Radiance Integration in Box Coordinates](9001-trapezoid-based-radiance-integration-in-box-coordinates.md)

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of intervals. It is faster than Simpson's rule because it avoids the more complex quadratic interpolation and dual integral calculations [S1].
- **Memory Complexity:** $O(N)$ to store radiance and extinction samples.
- **Accuracy:** Comparable to the trapezoid rule when extinction $\bar{\sigma}_t$ is low. Significantly more accurate than the trapezoid rule when $\bar{\sigma}_t$ is high, becoming comparable to Simpson's rule in accuracy [S1].
- **Strengths:** Improved accuracy for dense volumes (e.g., flames in smoke) without the computational cost of Simpson's rule. Theoretically identical to the trapezoid rule when extinction is zero [S1].
- **Limitations:** Requires sampling of both radiance and extinction at interval endpoints. Assumes piecewise constant extinction for speed.
- **When to use:** Rendering emissive inhomogeneous volumes with high extinction coefficients where standard quadrature rules lose accuracy.
- **When not to use:** When extinction is negligible and maximum speed is required (trapezoid rule is sufficient).

## Background
- **Origin:** Proposed by T. Kuge et al. in "Real-time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights" [S1].
- **History:** Developed to address the inaccuracy of Newton-Cotes rules (trapezoid/Simpson) in dense participating media. These rules approximate the product of transmittance and radiance directly, leading to errors when extinction is high. The proposed method discretizes radiance and extinction individually to derive an analytic form for throughput.
- **Usage:** Used in real-time rendering of emissive volumes, such as flames inside dense smoke [S1].

## Sources
[S1] Real‐time Indirect Illumination of Emissive Inhomogeneous Volumes using Layered Polygonal Area Lights, https://doi.org/10.1111/cgf.13851, fetched 2026-10-26
