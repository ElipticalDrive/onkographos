# Quantized Knot Position and Coefficient Computation

Also called: Quantized Knot and Coefficient Setup

## Overview
This method computes quantized knot positions ($\bar{t}_k$) and localized difference coefficients ($\bar{a}_{kd}$) from floating-point particle attributes to ensure compact support and neutralize errors in integer-only computations [S1]. It transforms continuous particle contributions into a piecewise polynomial function with shifted evenness, preventing non-zero coefficients at the boundary of the support from propagating error into regions where the contribution should be zero [S1]. Unlike direct use of floating-point coefficients, this quantization introduces a controlled error dependent on quantum values $\tau$ and $\varsigma$, but allows exact integer arithmetic operations, avoiding floating-point precision issues during rendering [S1].

## How it works
**Inputs:**
*   Particle attributes: mass $\mu$, density $\rho$, scale $\zeta$, amplitude $\alpha$ [S1].
*   Ray intersection data: normalized distance $\Lambda$ and base knot parameter $t_k$ [S1].
*   Method parameters: length quantum $\tau$, value quantum scaling $\varsigma$, maximum number of non-trivial polynomial pieces $K$, and maximal polynomial degree $D$ [S1].
*   Look-up table (LUT): Precomputed floating-point tuples $(\theta_k, \hat{s}_{kd})$ indexed by discrete normalized distance values $\Lambda$ [S1].

**Outputs:**
*   Quantized knot counts $\bar{t}_k$ for $k \in \{-\lceil K/2 \rceil, \dots, \lceil K/2 \rceil\}$ [S1].
*   Quantized localized difference coefficients $\bar{a}_{kd}$ for $k \in \{-\lceil K/2 \rceil, \dots, \lceil K/2 \rceil\}$ and $d \in \{0, \dots, D\}$ [S1].

**Data Structures:**
*   Knots $\bar{t}_k$: Integer arrays or dictionaries storing positions in units of $\tau$ [S1].
*   Coefficients $\bar{a}_{kd}$: Integer arrays or dictionaries storing localized difference coefficients [S1].
*   Look-up Table: A fixed-size table mapping each discrete $\Lambda$ to arrays of $\theta_k$ (length $\lceil K/2 \rceil$) and $\hat{s}_{kd}$ (dimensions $\lceil K/2 \rceil \times (D+1)$) [S1].

**Steps:**

1.  **Lookup:** Retrieve the optimal normalized positive knot positions $\theta_k$ and localized difference coefficients $\hat{s}_{kd}$ from the LUT corresponding to the normalized distance closest to $\Lambda$ [S1].
2.  **Compute Quantized Knots:** Calculate the integer knot counts $\bar{t}_k$ using nearest integer rounding (denoted $\lceil \cdot \rceil$, where $z + 0.5$ rounds to either $z$ or $z+1$ without affecting correctness) [S1]:
    $$ \bar{t}_0 = \left\lceil \frac{t_k}{\tau} \right\rceil $$
    $$ \bar{t}_k = \bar{t}_0 + \left\lceil \frac{\zeta \theta_k}{\tau} \right\rceil \quad \text{for } k = 1, \dots, \lceil K/2 \rceil $$
    $$ \bar{t}_{-k} = 2\bar{t}_0 - \bar{t}_k \quad \text{for } k = 1, \dots, \lceil K/2 \rceil $$
3.  **Compute Primary Coefficients:** For $k \geq 1$, except for the case where $k=1$ and $d$ is uneven if $K$ is uneven, compute the quantized localized difference coefficients:
    $$ \bar{a}_{kd} = \left\lceil \frac{\tau^d \mu \alpha \hat{s}_{kd}}{\varsigma \rho \zeta^{d+3}} \right\rceil $$
    Compute the symmetric coefficients using shifted evenness:
    $$ \bar{a}_{-kd} = (-1)^{d+1} \bar{a}_{kd} $$
4.  **Handle Middle Knot (Even $K$):** If $K$ is even, there is a middle knot at index $0$ [S1]. For uneven $d$, compute $\bar{a}_{0d}$ to satisfy the shifted evenness condition:
    $$ \bar{a}_{0d} = -2 \sum_{k=-\lceil K/2 \rceil}^{-1} \sum_{j=d}^D \binom{j}{d} \bar{a}_{kj} (\bar{t}_0 - \bar{t}_k)^{j-d} $$
    For even $d$, set $\bar{a}_{0d} = 0$ [S1].
5.  **Handle Boundary Coefficients (Odd $K$):** If $K$ is odd, there is no middle knot ($\bar{a}_{0d}=0$) [S1]. For uneven $d$, compute $\bar{a}_{-1d}$ in descending order of $d$:
    $$ \bar{a}_{-1d} = - \sum_{k=-\lceil K/2 \rceil}^2 \bar{a}_{kd} - \sum_{j=d+1}^D \binom{j}{d} \sum_{k=-\lceil K/2 \rceil}^{-1} \bar{a}_{kj} (\bar{t}_0 - \bar{t}_k)^{j-d} $$
    Compute the symmetric counterpart:
    $$ \bar{a}_{1d} = (-1)^{d+1} \bar{a}_{-1d} $$
6.  **Recursive Specification:** The localized coefficients $\tilde{a}_{kd}$ defining the piecewise polynomial $A_k(t)$ are recursively computed from previous coefficients with base case $\tilde{a}_{(-\lceil K/2 \rceil - 1)d} = 0$:
    $$ \tilde{a}_{kd} = \tilde{a}_{kd} + \sum_{l=-\lceil K/2 \rceil}^{k-1} \sum_{j=d}^D \binom{j}{d} \tilde{a}_{lj} (\tilde{t}_k - \tilde{t}_l)^{j-d} $$
    This recursion ensures continuity and the correct polynomial representation within each knot interval [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_quantized_knots_and_coefficients(mu, rho, zeta, alpha, t_k, Lambda, tau, varsigma, K, D, LUT):
    """
    Computes quantized knot positions and coefficients for a particle.
    
    Args:
        mu: Particle mass
        rho: Particle density
        zeta: Particle scale
        alpha: Particle amplitude
        t_k: Base knot parameter
        Lambda: Normalized distance from ray closest point
        tau: Length quantum value
        varsigma: Value quantum scaling factor
        K: Maximum number of non-trivial polynomial pieces
        D: Maximal polynomial degree
        LUT: Look-up table function LUT(Lambda) -> (theta_k, s_hat_kd)
             theta_k: list of length ceil(K/2) containing normalized knot positions
             s_hat_kd: list of lists of shape (ceil(K/2), D+1) containing coefficients
            
    Returns:
        t_bar: Dictionary of quantized knot counts {k: int}
        a_bar: Dictionary of quantized coefficients {(k, d): int}
        a_tilde: Dictionary of recursive polynomial coefficients {(k, d): float}
    """
    
    # Step 1: Lookup
    theta_k, s_hat_kd = LUT(Lambda)
    
    def round_nearest(x):
        return int(math.floor(x + 0.5))
    
    half_K = math.ceil(K / 2)
    t_bar = {0: round_nearest(t_k / tau)}
    
    # Step 2: Compute Quantized Knots
    for k in range(1, half_K + 1):
        t_bar[k] = t_bar[0] + round_nearest((zeta * theta_k[k - 1]) / tau)
        t_bar[-k] = 2 * t_bar[0] - t_bar[k]
        
    a_bar = {}
    
    # Step 3: Compute Primary Coefficients
    for k in range(1, half_K + 1):
        for d in range(D + 1):
            # Exception: k=1 and uneven d if K is uneven
            if k == 1 and K % 2 != 0 and d % 2 != 0:
                continue
                
            s_hat = s_hat_kd[k - 1][d]
            numerator = (tau ** d) * mu * alpha * s_hat
            denominator = varsigma * rho * (zeta ** (d + 3))
            
            a_bar[(k, d)] = round_nearest(numerator / denominator)
            a_bar[(-k, d)] = ((-1) ** (d + 1)) * a_bar[(k, d)]
            
    # Step 4: Handle Middle Knot (Even K)
    if K % 2 == 0:
        for d in range(D + 1):
            if d % 2 != 0:
                sum_val = 0.0
                for k in range(-half_K, 0):
                    for j in range(d, D + 1):
                        binom = math.comb(j, d)
                        diff_t = t_bar[0] - t_bar[k]
                        sum_val += binom * a_bar.get((k, j), 0) * (diff_t ** (j - d))
                a_bar[(0, d)] = -2 * sum_val
            else:
                a_bar[(0, d)] = 0

    # Step 5: Handle Boundary Coefficients (Odd K)
    if K % 2 != 0:
        uneven_d_list = [d for d in range(D, -1, -1) if d % 2 != 0]
        
        for d in uneven_d_list:
            # First term: sum_{k=-ceil(K/2)}^2 a_kd
            sum1 = 0.0
            for k in range(-half_K, 3):
                sum1 += a_bar.get((k, d), 0)
            
            # Second term: sum_{j=d+1}^D binom(j,d) sum_{k=-ceil(K/2)}^{-1} a_kj (t0 - tk)^(j-d)
            sum2 = 0.0
            for j in range(d + 1, D + 1):
                binom = math.comb(j, d)
                inner_sum = 0.0
                for k in range(-half_K, 0):
                    diff_t = t_bar[0] - t_bar[k]
                    inner_sum += a_bar.get((k, j), 0) * (diff_t ** (j - d))
                sum2 += binom * inner_sum
                
            a_bar[(-1, d)] = -sum1 - sum2
            a_bar[(1, d)] = ((-1) ** (d + 1)) * a_bar[(-1, d)]
            
    # Step 6: Recursive Specification for polynomial coefficients
    a_tilde = {k: {d: a_bar.get((k, d), 0) for d in range(D + 1)} for k in t_bar}
    for k in range(-half_K, half_K + 1):
        for d in range(D + 1):
            acc = 0.0
            for l in range(-half_K, k):
                for j in range(d, D + 1):
                    binom = math.comb(j, d)
                    diff_t = t_bar[k] - t_bar[l]
                    acc += binom * a_tilde[l][j] * (diff_t ** (j - d))
            a_tilde[k][d] += acc
            
    return t_bar, a_bar, a_tilde
```

**Edge Cases and Pitfalls:**
*   **Boundary Error Propagation:** Non-zero coefficients at the boundary of the support ($\tilde{a}_{[\kappa/2]d}$) propagate error to regions where the contribution should be zero, annihilating quantization benefits [S1]. This is prevented by enforcing shifted evenness and compact support via the specific computation of boundary coefficients [S1].
*   **Rounding Ambiguity:** The rounding function $\lceil \cdot \rceil$ is defined as nearest integer [S1]. The choice of rounding $z + 0.5$ to $z$ or $z+1$ is irrelevant for correctness [S1].
*   **Exception Handling:** The formula for $\bar{a}_{kd}$ excludes $k=1$ and uneven $d$ if $K$ is uneven [S1]. These must be handled separately via the boundary coefficient formulas [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Performance:** Allows exact integer arithmetic operations, avoiding floating-point precision issues during rendering [S1].
*   **Limitations:** Introduces a quantization error dependent on $\tau$ and $\varsigma$ [S1].
*   **When to use:** When integer-only arithmetic is required or preferred for performance or precision stability in volume rendering [S1].
*   **When not to use:** When floating-point precision is sufficient and the overhead of quantization error management is undesirable [S1].

## Background
This method originates from the work on Particle-Wise Higher-Order SPH Field Approximation for DVR [S1]. It is part of a broader effort to represent Smoothed Particle Hydrodynamics (SPH) fields for Direct Volume Rendering (DVR) using higher-order polynomials [S1]. The quantization approach is designed to maintain the mathematical properties (compact support, shifted evenness) of the continuous approximation while mapping them to a discrete, integer-friendly representation [S1].

## Sources
[S1] Particle-Wise Higher-Order SPH Field Approximation for DVR, https://arxiv.org/abs/2401.02896v1, fetched 2026-10-26
