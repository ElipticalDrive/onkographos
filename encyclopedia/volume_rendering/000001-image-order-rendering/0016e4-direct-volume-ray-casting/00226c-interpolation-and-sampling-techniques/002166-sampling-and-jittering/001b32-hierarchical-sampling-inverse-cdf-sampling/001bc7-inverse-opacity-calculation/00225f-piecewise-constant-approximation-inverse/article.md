# Piecewise Constant Approximation Inverse

Also called: Inverse CDF with Piecewise Constant Density, Piecewise Constant Inverse Sampling

## Overview
Piecewise Constant Approximation Inverse is a method for computing the inverse of the density integral along a ray, used to determine sample positions in inverse opacity sampling. It assumes that the density $\sigma_{\mathbf{r}}$ is constant within each discrete sample interval $[t_{j-1}, t_j]$. This approach differs from its sibling, [Piecewise Linear Approximation Inverse](2649-piecewise-linear-approximation-inverse.md), by treating density as a step function rather than a linearly varying one, resulting in a linear equation for the ray parameter $t$ instead of a quadratic one. It builds on [Inverse Opacity Calculation](1099-inverse-opacity-calculation.md) by providing an explicit closed-form solution for the ray parameter given a target cumulative density value.

## How it works
**Inputs:**
- $y$: The target cumulative density value (integral of density) to solve for.
- $\{t_j\}_{j=0}^{N}$: The sorted array of sample positions along the ray, where $t_0$ is the entry point and $t_N$ is the exit point.
- $\{\hat{\sigma}_j\}_{j=1}^{N}$: The density values sampled at each interval, where $\hat{\sigma}_j$ represents the constant density in the interval $[t_{j-1}, t_j]$.

**Outputs:**
- $t$: The ray parameter such that the integral of density from $t_0$ to $t$ equals $y$.

**Data Structures:**
- Arrays `t` and `sigma` of size $N+1$ and $N$ respectively.
- Precomputed prefix sums of the product $\sigma \cdot \Delta t$ can be used to accelerate the summation, though the direct summation is $O(N)$ per query.

**Steps:**
1. Iterate through the sample intervals $j=1, \dots, N$ to find the index $i$ such that the cumulative integral up to $t_i$ is less than or equal to $y$, and the cumulative integral up to $t_{i+1}$ is greater than $y$. Specifically, find $i$ where:
   $$ \sum_{j=1}^{i} \hat{\sigma}_j (t_j - t_{j-1}) \le y < \sum_{j=1}^{i+1} \hat{\sigma}_j (t_j - t_{j-1}) $$
   If $y$ exceeds the total integral, clamp $t$ to $t_N$. If $y \le 0$, return $t_0$.

2. Calculate the cumulative integral up to the start of the identified interval $t_i$:
   $$ C_i = \sum_{j=1}^{i} \hat{\sigma}_j (t_j - t_{j-1}) $$

3. Solve for $t$ using the linear relationship within the interval $[t_i, t_{i+1}]$. The density is assumed constant at $\hat{\sigma}_i$. The formula is derived from:
   $$ y = C_i + \hat{\sigma}_i (t - t_i) $$
   Rearranging for $t$:
   $$ t = t_i + \frac{y - C_i}{\hat{\sigma}_i} $$

**Formula:**
The explicit solution for $t$ is:
$$ t = t_i + \frac{y - \sum_{j=1}^{i} \hat{\sigma}_j (t_j - t_{j-1})}{\hat{\sigma}_i} $$
where $\hat{\sigma}_i$ is the density in the interval containing $t$.

**Parameters:**
- Epsilon ($\epsilon$): A small positive constant added to the denominator $\hat{\sigma}_i$ to prevent division by zero or instability when density is near zero. Typical value: $10^{-6}$ or machine epsilon scaled by density range.

**Edge Cases and Pitfalls:**
- **Zero Density:** If $\hat{\sigma}_i \approx 0$, the denominator becomes unstable. The implementation must add a small epsilon value to the denominator to improve stability [S1, unofficial].
- **Boundary Conditions:** If $y$ is outside the range $[0, I_{total}]$, the result should be clamped to $[t_0, t_N]$.
- **Precision:** Floating-point errors in the summation $C_i$ can accumulate. Using Kahan summation or precomputed prefix sums can mitigate this.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def piecewise_constant_inverse(t, sigma, y, epsilon=1e-6):
    """
    Compute the ray parameter t for a given cumulative density y
    using piecewise constant density approximation.
    
    Args:
        t: Array of sample positions [t_0, t_1, ..., t_N]
        sigma: Array of densities [sigma_1, ..., sigma_N] for intervals [t_0, t_1], ..., [t_{N-1}, t_N]
        y: Target cumulative density value
        epsilon: Small value to add to denominator for stability
        
    Returns:
        t_out: The computed ray parameter
    """
    n = len(sigma)
    if n == 0:
        return t[0]
    
    # Compute cumulative integrals
    # C[i] stores the integral from t_0 to t_i
    C = [0.0] * (n + 1)
    for j in range(1, n + 1):
        delta_t = t[j] - t[j-1]
        C[j] = C[j-1] + sigma[j-1] * delta_t
        
    total_integral = C[n]
    
    # Clamp y to valid range
    if y <= 0.0:
        return t[0]
    if y >= total_integral:
        return t[n]
        
    # Find the interval i such that C[i] <= y < C[i+1]
    # Since C is monotonically increasing, we can use binary search or linear scan
    i = 0
    for j in range(1, n + 1):
        if C[j] > y:
            i = j - 1
            break
    else:
        # y is greater than total integral, handled by clamp above, but for safety
        return t[n]
        
    # Solve for t in interval [t_i, t_{i+1}]
    # y = C[i] + sigma[i] * (t - t[i])
    # t = t[i] + (y - C[i]) / sigma[i]
    
    sigma_val = sigma[i]
    if abs(sigma_val) < epsilon:
        sigma_val = epsilon if sigma_val >= 0 else -epsilon
        
    t_out = t[i] + (y - C[i]) / sigma_val
    
    return t_out
```

## Subtopics

### [Inverse CDF Interpolation](7982-inverse-cdf-interpolation.md)
Uses linear interpolation between cumulative weight boundaries rather than assuming constant density within bins, providing smoother sampling distributions. Choose this when higher accuracy in sample placement is needed and the cost of solving a linear interpolation is acceptable.

## Operating characteristics
- **Time Complexity:** $O(N)$ for a linear scan to find the interval, or $O(\log N)$ if binary search is used on the cumulative integral array. The final calculation is $O(1)$.
- **Memory Complexity:** $O(N)$ to store the cumulative integral array if precomputed, or $O(1)$ if computed on the fly (though recomputing sums is inefficient).
- **Strengths:** Simple to implement, computationally inexpensive per sample, explicit closed-form solution.
- **Limitations:** Assumes constant density within intervals, which can lead to sampling artifacts if the density varies significantly within an interval. Less accurate than piecewise linear approximations.
- **When to use:** When performance is critical and the density field is relatively smooth or the sampling rate is high enough that the piecewise constant assumption is reasonable.
- **When not to use:** When high accuracy in sample placement is required for highly varying density fields, or when the density is known to vary linearly within intervals.

## Background
The method is described in "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial], which derives explicit formulae for the density integral inverse used in inverse opacity sampling. The piecewise constant approximation is the simplest form of inverting the density integral, serving as a baseline for more complex approximations like piecewise linear.

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
