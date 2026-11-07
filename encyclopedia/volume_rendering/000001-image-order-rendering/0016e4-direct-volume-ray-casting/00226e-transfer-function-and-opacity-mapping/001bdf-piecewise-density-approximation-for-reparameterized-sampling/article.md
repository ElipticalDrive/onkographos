# Piecewise Density Approximation for Reparameterized Sampling

Also called: PWC approximation, PWL approximation, Piecewise Constant Approximation

## Overview
This method enables reparameterized sampling in volume rendering by approximating the density field $\sigma_r(t)$ along a ray as either piecewise constant or piecewise linear on a predefined grid [S1, unofficial][S2, unofficial]. This approximation allows the cumulative opacity integral to be computed analytically, permitting the inversion of the opacity function to generate samples according to a target distribution [S1, unofficial][S2, unofficial]. It differs from standard fixed-step ray casting by providing differentiable, distribution-aware sample locations [S1, unofficial].

## How it works
**Inputs:**
- Grid of $m+1$ points $t_0, t_1, \dots, t_m$ such that $t_n = t_0 < t_1 < \dots < t_m = t_f$ [S1, unofficial][S2, unofficial].
- A density query function $\sigma_r(t)$ that returns the scalar density at distance $t$ along the ray [S1, unofficial][S2, unofficial].
- A target uniform random sample $u \in [0, 1)$ [S1, unofficial][S2, unofficial].
- A scaling factor $y_f$ [S1, unofficial][S2, unofficial].

**Outputs:**
- A sample location $t \in [t_n, t_f]$ on the ray such that the cumulative opacity up to $t$ matches the target value derived from $u$ [S1, unofficial][S2, unofficial].

**Data Structures:**
- **Grid:** An array of $m+1$ floats representing the bin boundaries $t_i$.
- **Density Values:**
  - For PWC: An array of $m$ floats $\hat{\sigma}_j = \sigma_r(\hat{t}_j)$, where $\hat{t}_j$ is a representative point in bin $j$ [S1, unofficial][S2, unofficial].
  - For PWL: An array of $m+1$ floats $\sigma_j = \sigma_r(t_j)$, the density at grid points [S1, unofficial][S2, unofficial].

**Steps:**

1. **Construct the Density Approximation:**
   Choose between Piecewise Constant (PWC) or Piecewise Linear (PWL) approximation [S1, unofficial][S2, unofficial].

   *For PWC:*
   1.1. For each bin $j \in \{0, \dots, m-1\}$, select a point $\hat{t}_j \in [t_j, t_{j+1}]$ [S2, unofficial].
   1.2. Query the density at this point: $\hat{\sigma}_j = \sigma_r(\hat{t}_j)$ [S1, unofficial][S2, unofficial].

   *For PWL:*
   1.1. Query the density at all grid points: $\sigma_j = \sigma_r(t_j)$ for $j \in \{0, \dots, m\}$ [S1, unofficial][S2, unofficial].

2. **Compute the Target Integral Value:**
   Calculate the target cumulative optical depth $Y$ corresponding to the random sample $u$ [S1, unofficial][S2, unofficial]:
   $$ Y = -\log(1 - y_f u) $$
   If $u=1$, handle the edge case where $Y \to \infty$ by clamping $t$ to $t_f$ [G].

3. **Find the Containing Bin:**
   Compute the cumulative integral values at the grid boundaries to determine which bin contains the solution $t$.

   *For PWC:*
   Compute $C_i = I_0(t_i)$ for $i=0, \dots, m$.
   $$ C_0 = 0 $$
   $$ C_i = \sum_{j=0}^{i-1} \hat{\sigma}_j (t_{j+1} - t_j) \quad \text{for } i > 0 $$
   Find index $k$ such that $C_k \le Y < C_{k+1}$ [S1, unofficial][S2, unofficial].

   *For PWL:*
   Compute $C_i = I_1(t_i)$ for $i=0, \dots, m$.
   $$ C_0 = 0 $$
   $$ C_i = \sum_{j=0}^{i-1} \frac{\sigma_j + \sigma_{j+1}}{2} (t_{j+1} - t_j) \quad \text{for } i > 0 $$
   Find index $k$ such that $C_k \le Y < C_{k+1}$ [S1, unofficial][S2, unofficial].

4. **Solve for $t$ within the Bin:**
   Solve the equation $I(t) = Y$ for $t \in [t_k, t_{k+1}]$ [S1, unofficial][S2, unofficial].

   *For PWC (Linear Equation):*
   The integral in bin $k$ is linear: $I_0(t) = C_k + \hat{\sigma}_k (t - t_k)$ [S1, unofficial][S2, unofficial].
   $$ t = t_k + \frac{Y - C_k}{\hat{\sigma}_k} $$
   If $\hat{\sigma}_k = 0$, set $t = t_{k+1}$ if $Y \ge C_k$, or handle as an edge case [G].

   *For PWL (Quadratic Equation):*
   The density in bin $k$ is interpolated: $\bar{\sigma}_r(t) = \sigma_k \frac{t_{k+1}-t}{t_{k+1}-t_k} + \sigma_{k+1} \frac{t-t_k}{t_{k+1}-t_k}$ [S2, unofficial].
   Let $\Delta t_k = t_{k+1} - t_k$ and $\tau = t - t_k$.
   The integral from $t_k$ to $t$ is:
   $$ I_{bin}(\tau) = \int_{t_k}^t \bar{\sigma}_r(s) ds = \sigma_k \tau + \frac{\sigma_{k+1} - \sigma_k}{2 \Delta t_k} \tau^2 $$
   We need to solve $C_k + I_{bin}(\tau) = Y$, or $I_{bin}(\tau) = Y - C_k$. Let $R = Y - C_k$.
   $$ \frac{\sigma_{k+1} - \sigma_k}{2 \Delta t_k} \tau^2 + \sigma_k \tau - R = 0 $$
   This is a quadratic equation $A \tau^2 + B \tau + C = 0$ with:
   $$ A = \frac{\sigma_{k+1} - \sigma_k}{2 \Delta t_k}, \quad B = \sigma_k, \quad C = -R $$
   Solve for $\tau$:
   If $|A| < \epsilon$ (i.e., $\sigma_k \approx \sigma_{k+1}$), use linear solution: $\tau = R / B$ [G].
   Otherwise, use quadratic formula:
   $$ \tau = \frac{-B + \sqrt{B^2 - 4AC}}{2A} $$
   (Select the root that lies in $[0, \Delta t_k]$; since the integral is monotonic for non-negative density, exactly one root will satisfy the bounds) [S1, unofficial][S2, unofficial].
   $$ t = t_k + \tau $$

**Parameters:**
- Grid resolution $m$: Not given in the sources. Typical values depend on the scene complexity and desired accuracy [G].
- $y_f$: Scaling factor for the opacity function [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Density:** If $\hat{\sigma}_k = 0$ in PWC, division by zero occurs. The sample should be pushed to the end of the bin or the next non-zero bin [G].
- **$u=1$:** Results in $Y \to \infty$. Clamp $t$ to $t_f$ [G].
- **Negative Density:** The method assumes non-negative density fields. If $\sigma_r(t)$ can be negative, the monotonicity of $I(t)$ is not guaranteed, and the bin search may fail [S1, unofficial][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def piecewise_density_approximation_sample(
    t_grid,       # List of m+1 floats: t_0, ..., t_m
    density_func, # Function sigma_r(t) -> float
    u,            # Random sample in [0, 1)
    y_f=1.0,      # Scaling factor
    approx_type="PWC" # "PWC" or "PWL"
):
    """
    Computes sample location t on ray using piecewise density approximation.
    """
    if u >= 1.0:
        return t_grid[-1]
    
    # Target integral value
    Y = -math.log(1.0 - y_f * u)
    
    m = len(t_grid) - 1
    
    # Precompute cumulative integrals at grid points
    C = [0.0] * (m + 1)
    
    if approx_type == "PWC":
        # PWC: Precompute representative densities and cumulative sums
        current_C = 0.0
        for i in range(1, m + 1):
            t_prev = t_grid[i-1]
            t_curr = t_grid[i]
            # Pick representative point: midpoint as default [G]
            t_rep = (t_prev + t_curr) / 2.0
            sigma_rep = density_func(t_rep)
            
            delta_t = t_curr - t_prev
            current_C += sigma_rep * delta_t
            C[i] = current_C
            
    elif approx_type == "PWL":
        # PWL: Precompute densities at grid points and cumulative sums
        sigma_grid = [density_func(t) for t in t_grid]
        
        current_C = 0.0
        for i in range(1, m + 1):
            t_prev = t_grid[i-1]
            t_curr = t_grid[i]
            sigma_prev = sigma_grid[i-1]
            sigma_curr = sigma_grid[i]
            
            delta_t = t_curr - t_prev
            # Trapezoidal area
            area = 0.5 * (sigma_prev + sigma_curr) * delta_t
            current_C += area
            C[i] = current_C
    else:
        raise ValueError("Unknown approximation type")
        
    # Find bin k such that C[k] <= Y < C[k+1]
    k = 0
    for i in range(m):
        if Y < C[i+1]:
            k = i
            break
    else:
        # Y >= C[m], return far clip
        return t_grid[-1]
        
    # Solve for t in bin k
    t_k = t_grid[k]
    t_k1 = t_grid[k+1]
    delta_t_k = t_k1 - t_k
    R = Y - C[k]
    
    if approx_type == "PWC":
        # Linear equation: R = sigma_rep * (t - t_k)
        # Need sigma_rep for bin k
        t_rep = (t_k + t_k1) / 2.0
        sigma_rep = density_func(t_rep)
        
        if abs(sigma_rep) < 1e-9:
            # Zero density, push to end of bin
            return t_k1
            
        t = t_k + R / sigma_rep
        return max(t_k, min(t_k1, t))
        
    elif approx_type == "PWL":
        # Quadratic equation: A*tau^2 + B*tau + C = 0
        sigma_k = density_func(t_k)
        sigma_k1 = density_func(t_k1)
        
        A = (sigma_k1 - sigma_k) / (2.0 * delta_t_k)
        B = sigma_k
        C_const = -R
        
        if abs(A) < 1e-9:
            # Linear case
            if abs(B) < 1e-9:
                return t_k1
            tau = -C_const / B
        else:
            discriminant = B*B - 4*A*C_const
            if discriminant < 0:
                # Numerical error, clamp
                return t_k1
            sqrt_disc = math.sqrt(discriminant)
            # Two roots: (-B + sqrt) / 2A and (-B - sqrt) / 2A
            # We want the root in [0, delta_t_k]
            tau1 = (-B + sqrt_disc) / (2.0 * A)
            tau2 = (-B - sqrt_disc) / (2.0 * A)
            
            if 0 <= tau1 <= delta_t_k:
                tau = tau1
            elif 0 <= tau2 <= delta_t_k:
                tau = tau2
            else:
                # Fallback
                tau = delta_t_k
                
        t = t_k + tau
        return max(t_k, min(t_k1, t))
```

## Subtopics

### [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md)
Uses linear opacity mapping with gradient scaling. Choose when gradient-based optimization of opacity is needed.

### [Direct Volume Rendering with Interactive Piecewise Transfer Function](2068-direct-volume-rendering-with-interactive-piecewise-transfer-.md)
Interactive piecewise transfer functions. Choose for interactive exploration.

### [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md)
Uses SDFs and logistic CDF. Choose for surface-based rendering.

### [Volume Raycasting Output Generation](4277-volume-raycasting-output-generation.md)
Standard raycasting output. Choose for baseline comparison.

## Operating characteristics
- **Time Complexity:** $O(m)$ to precompute cumulative integrals, where $m$ is the number of grid bins. Finding the bin is $O(m)$ with linear scan or $O(\log m)$ with binary search. Solving for $t$ is $O(1)$.
- **Memory Complexity:** $O(m)$ to store grid points and cumulative integrals.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Differentiable with respect to density field $\sigma_r$ [S1, unofficial]. Allows analytical computation of integrals [S1, unofficial][S2, unofficial].
- **Limitations:** Requires precomputation of grid densities. Approximation error depends on grid resolution [S1, unofficial][S2, unofficial].
- **When to use:** When differentiable sampling is required, or when analytical inversion of opacity is beneficial [S1, unofficial][S2, unofficial].
- **When not to use:** When high accuracy is needed without fine grids, or when density field is not available analytically [G].

## Background
The method was proposed in "Differentiable Rendering with Reparameterized Volume Sampling" [S1, unofficial][S2, unofficial]. It builds on the idea of approximating the density field to enable analytical integration and inversion [S1, unofficial][S2, unofficial].

## Sources
[S1] Differentiable Rendering with Reparameterized Volume Sampling, https://doi.org/10.48550/arxiv.2302.10970, fetched 2026-10-26 (unofficial)
[S2] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
