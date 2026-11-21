# Piecewise Density Approximation for Reparameterized Sampling

Also called: PWC approximation, PWL approximation, Piecewise Constant Approximation

## Overview
This method enables analytical inversion of the volume rendering opacity function by approximating the density field $\sigma_r$ along a ray as either piecewise constant or piecewise linear on a predefined grid [S1, S2, unofficial]. It solves the problem of computing sample locations $t$ for reparameterized sampling (inverse opacity sampling) without numerical root-finding, allowing gradients to back-propagate through the sample generation process [S1, S2, unofficial]. It differs from [Opacity Correction for Slice Count Invariance](1277-opacity-correction-for-slice-count-invariance.md) by focusing on sample location derivation for stochastic integration, and from [Piecewise Linear Mapping and Integration](9000-piecewise-linear-mapping-and-integration.md) by explicitly constructing the density approximation to facilitate the inversion of the cumulative opacity integral [S1, S2, unofficial].

## How it works
**Inputs:**
- Ray origin $t_n$ (or $t_{r_1}$) and end $t_f$ [S1, S2, unofficial].
- A grid of $m+1$ points $t_0, t_1, \dots, t_m$ such that $t_n = t_0 < t_1 < \dots < t_m = t_f$ [S1, S2, unofficial].
- A density query function $\sigma_r(t)$ that returns the scalar density at any point $t$ [S1, S2, unofficial].
- A target uniform random sample $u \in [0, 1]$ [S1, S2, unofficial].
- A scaling factor $y_f$ (often related to the total optical depth or a normalization constant, appearing in the equation $y_f u = F_r(t)$) [S1, S2, unofficial].

**Outputs:**
- A sample location $t$ on the ray such that the cumulative opacity up to $t$ corresponds to the random sample $u$ [S1, S2, unofficial].

**Data Structures:**
- Grid array: A sorted list of floats $[t_0, t_1, \dots, t_m]$ [S1, S2, unofficial].
- Density values: Pre-computed or on-the-fly queried scalar values $\sigma_r(t_i)$ at grid points or representative points within bins [S1, S2, unofficial].

**Steps:**

1.  **Construct the Density Approximation:**
    Choose between Piecewise Constant (PWC) or Piecewise Linear (PWL) approximation [S1, S2, unofficial].

    *   **Piecewise Constant (PWC):**
        For each bin $j \in [0, m-1]$ defined by $[t_j, t_{j+1}]$, select a representative point $\hat{t}_j$ within the bin [S1, S2, unofficial]. The sources specify picking a random point $\hat{t}_j \in [t_j, t_{j+1}]$ [S2, unofficial], though [S1, unofficial] describes picking "a point" generally. The density is approximated as constant: $\sigma_r(t) \approx \sigma_r(\hat{t}_j)$ for $t \in [t_j, t_{j+1})$ [S1, S2, unofficial].

    *   **Piecewise Linear (PWL):**
        Compute the density at the grid points $\sigma_r(t_j)$ for all $j$ [S1, S2, unofficial]. The density is approximated by linear interpolation between grid points [S1, S2, unofficial]. For $t \in [t_i, t_{i+1})$, the interpolated density $\bar{\sigma}_r(t)$ is:
        $$ \bar{\sigma}_r(t) = \sigma_r(t_i) \frac{t_{i+1}-t}{t_{i+1}-t_i} + \sigma_r(t_{i+1}) \frac{t-t_i}{t_{i+1}-t_i} $$
        [S2, unofficial].

2.  **Compute the Cumulative Integral Approximation $I(t)$:**
    Calculate the integral of the approximated density from the ray start $t_n$ to a target point $t \in [t_i, t_{i+1})$ [S1, S2, unofficial].

    *   **For PWC ($I_0(t)$):**
        $$ I_0(t) = \sum_{j=1}^{i} \sigma_r(\hat{t}_j)(t_j - t_{j-1}) + \sigma_r(\hat{t}_i)(t - t_i) $$
        [S1, S2, unofficial]. Note: The summation index in [S1, S2, unofficial] starts at $j=1$, implying bins are indexed $1 \dots m$ or $0 \dots m-1$ with specific offset handling. Based on the formula structure $\sum_{j=1}^i$, it sums full bins prior to the current bin $i$, and adds the partial current bin. We assume grid indices $0 \dots m$ and bins $1 \dots m$ where bin $j$ is $[t_{j-1}, t_j]$. If $t \in [t_i, t_{i+1})$, the full bins are $1 \dots i$ (indices $0 \dots i-1$ in 0-based grid) and the partial bin is $i$ (index $i$ in 0-based grid). The formula in [S1, S2, unofficial] uses $t_j - t_{j-1}$, consistent with bin width.

    *   **For PWL ($I_1(t)$):**
        $$ I_1(t) = \sum_{j=1}^{i} \frac{\sigma_r(t_j) + \sigma_r(t_{j-1})}{2} (t_j - t_{j-1}) + \frac{\sigma_r(t_i) + \bar{\sigma}_r(t)}{2} (t - t_i) $$
        [S2, unofficial]. The summation covers full trapezoidal areas for previous bins, and the current term is the trapezoidal area for the partial bin $[t_i, t]$ using the interpolated density $\bar{\sigma}_r(t)$ [S2, unofficial].

3.  **Solve for Sample Location $t$:**
    The goal is to find $t$ such that the cumulative opacity matches the random sample [S1, S2, unofficial]. The relationship is given by:
    $$ y_f u = 1 - \exp\left(-\int_{t_n}^t \sigma_r(s) ds\right) $$
    [S1, S2, unofficial].
    Rearranging for the integral:
    $$ \int_{t_n}^t \sigma_r(s) ds = -\log(1 - y_f u) $$
    [S1, S2, unofficial].
    Let $Y = -\log(1 - y_f u)$. We solve $I(t) = Y$ for $t$ [S1, S2, unofficial].

    *   **Inversion for PWC:**
        $I_0(t)$ is piecewise linear and monotonic [S1, S2, unofficial].
        1.  Find the bin index $i$ such that $I_0(t_i) \le Y < I_0(t_{i+1})$ [S1, S2, unofficial]. This can be done via binary search on the cumulative sums of bin densities.
        2.  Within bin $i$, solve the linear equation:
            $$ I_0(t_i) + \sigma_r(\hat{t}_i)(t - t_i) = Y $$
            $$ t = t_i + \frac{Y - I_0(t_i)}{\sigma_r(\hat{t}_i)} $$
            [S1, S2, unofficial]. If $\sigma_r(\hat{t}_i) = 0$, handle as a special case (e.g., return $t_{i+1}$ or next non-zero bin) [G].

    *   **Inversion for PWL:**
        $I_1(t)$ is piecewise quadratic and monotonic [S1, S2, unofficial].
        1.  Find the bin index $i$ such that $I_1(t_i) \le Y < I_1(t_{i+1})$ [S1, S2, unofficial].
        2.  Within bin $i$, let $\Delta t = t - t_i$. The equation is:
            $$ I_1(t_i) + \frac{\sigma_r(t_i) + \bar{\sigma}_r(t_i + \Delta t)}{2} \Delta t = Y $$
            Substitute $\bar{\sigma}_r(t_i + \Delta t) = \sigma_r(t_i) + \frac{\sigma_r(t_{i+1}) - \sigma_r(t_i)}{t_{i+1} - t_i} \Delta t$.
            Let $A = \sigma_r(t_i)$ and $B = \frac{\sigma_r(t_{i+1}) - \sigma_r(t_i)}{t_{i+1} - t_i}$. Then $\bar{\sigma}_r = A + B \Delta t$.
            The equation becomes:
            $$ I_1(t_i) + \frac{A + (A + B \Delta t)}{2} \Delta t = Y $$
            $$ I_1(t_i) + A \Delta t + \frac{B}{2} \Delta t^2 = Y $$
            $$ \frac{B}{2} \Delta t^2 + A \Delta t + (I_1(t_i) - Y) = 0 $$
            [S1, S2, unofficial].
            Solve this quadratic equation for $\Delta t$. Since $I_1(t)$ is monotonic increasing (assuming non-negative density), there is a unique positive root within $[0, t_{i+1} - t_i]$ [S1, S2, unofficial].
            $$ \Delta t = \frac{-A + \sqrt{A^2 - 2B(I_1(t_i) - Y)}}{B} $$
            [G] (Derived from quadratic formula; note $B$ can be zero, reducing to linear case).
            $$ t = t_i + \Delta t $$
            [S1, S2, unofficial].

**Parameters:**
- Grid resolution $m$: Not given in the sources. Typical values depend on the scene complexity and desired accuracy [G].
- $y_f$: Not given in the sources. Likely a normalization factor or total optical depth scaling [G].

**Edge Cases and Pitfalls:**
- **Zero Density:** If $\sigma_r(\hat{t}_i) = 0$ in PWC, division by zero occurs in inversion [G]. The algorithm should skip bins with zero density or handle them as infinite optical depth gaps [G].
- **Negative Density:** The sources state that for non-negative fields, the approximations are non-negative [S1, S2, unofficial]. If density is negative, monotonicity may be violated, breaking the binary search and root-finding assumptions [G].
- **Precision:** Solving the quadratic equation for PWL requires care with floating-point precision, especially when $B$ is small [G].

## Subtopics

### Piecewise Constant Approximation (PWC)
Approximates density as constant within each grid bin. Simpler to implement and compute, but less accurate for smooth density variations. Chosen when computational efficiency is prioritized over sampling accuracy [S1, S2, unofficial].

### Piecewise Linear Approximation (PWL)
Approximates density via linear interpolation between grid points. More accurate than PWC, but requires solving a quadratic equation for inversion. Chosen when higher fidelity sample locations are needed [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(m)$ to pre-compute cumulative integrals if not cached, or $O(\log m)$ per sample if binary search is used on pre-computed cumulative sums [G]. Inversion within a bin is $O(1)$ [S1, S2, unofficial].
- **Memory Complexity:** $O(m)$ to store grid points and cumulative integral values [G].
- **Strengths:** Analytical inversion avoids iterative root-finding, enabling differentiable rendering through the sample generation step [S1, S2, unofficial]. Fully differentiable with respect to density values [S1, S2, unofficial].
- **Limitations:** Accuracy depends on grid resolution. PWC may introduce bias in sample distribution for smooth densities. PWL requires solving quadratic equations, which is slightly more complex than PWC [S1, S2, unofficial].
- **When to use:** When differentiable volume sampling is required, and the density field can be efficiently queried [S1, S2, unofficial].
- **When not to use:** When the density field is highly discontinuous or the grid resolution is insufficient to capture fine details, leading to inaccurate sample locations [G].

## Background
The method originates from the work on Differentiable Rendering with Reparameterized Volume Sampling [S1, S2, unofficial]. It builds on the idea of approximating integrals in volume rendering to enable analytical inversion of the opacity function, a technique also referenced in earlier NeRF literature [S2, unofficial]. The approach allows for gradient-based optimization of scene parameters by making the sampling process differentiable [S1, S2, unofficial].

## Sources
[S1] Differentiable Rendering with Reparameterized Volume Sampling, https://doi.org/10.48550/arxiv.2302.10970, fetched 2026-10-26 (unofficial)
[S2] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
