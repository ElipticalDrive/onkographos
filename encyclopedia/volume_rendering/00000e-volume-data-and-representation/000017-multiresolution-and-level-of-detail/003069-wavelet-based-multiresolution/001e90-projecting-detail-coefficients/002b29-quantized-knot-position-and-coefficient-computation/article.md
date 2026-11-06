# Quantized Knot Position and Coefficient Computation

Also called: Quantized Knot and Coefficient Setup

## Overview
This method computes quantized knot positions ($\bar{t}_k$) and localized difference coefficients ($\bar{a}_{kd}$) from floating-point particle attributes to ensure compact support and neutralize errors in integer-only computations [S1]. It transforms continuous particle contributions into a piecewise polynomial function with shifted evenness, preventing non-zero coefficients at the boundary of the support from propagating error into regions where the contribution should be zero [S1]. Unlike direct use of floating-point coefficients, this quantization introduces a controlled error dependent on quantum values $\tau$ and $\varsigma$, but allows exact integer arithmetic operations, avoiding floating-point precision issues during rendering [S1].

## How it works
**Inputs:**
*   Particle attributes: mass $\mu$, density $\rho$, scale $\zeta$, amplitude $\alpha$ [S1].
*   Ray intersection data: distance from ray closest point $\Lambda\zeta$, and the base knot parameter $t_k$ [S1].
*   Method parameters: length quantum $\tau$, value quantum scaling $\varsigma$ (denoted $\zeta$ in some source contexts for coefficient scaling, distinct from particle scale $\zeta$), maximum number of non-trivial polynomial pieces $K$, and maximal polynomial degree $D$ [S1].
*   Look-up table: containing optimal normalized positive knot positions $\theta_k$ and localized difference coefficients $\hat{s}_{kd}$ indexed by normalized distance $\Lambda$ [S1].

**Outputs:**
*   Quantized knot counts $\bar{t}_k$ for $k \in \{-\lceil K/2 \rceil, \dots, \lceil K/2 \rceil\}$ [S1].
*   Quantized localized difference coefficients $\bar{a}_{kd}$ for $k \in \{-\lceil K/2 \rceil, \dots, \lceil K/2 \rceil\}$ and $d \in \{0, \dots, D\}$ [S1].

**Data Structures:**
*   Knots $\bar{t}_k$: Integer values representing knot positions in units of $\tau$.
*   Coefficients $\bar{a}_{kd}$: Integer values representing the localized difference coefficients.
*   Look-up Table (LUT): A precomputed table mapping normalized distance $\Lambda$ to tuples $(\theta_k, \hat{s}_{kd})$.

**Steps:**

1.  **Lookup:** Determine the normalized distance $\Lambda$ from the ray closest point. Retrieve the optimal normalized positive knot positions $\theta_k$ and localized difference coefficients $\hat{s}_{kd}$ from the look-up table corresponding to the normalized distance closest to $\Lambda$ [S1].

2.  **Compute Quantized Knots:** Calculate the integer knot counts $\bar{t}_k$ using nearest integer rounding (denoted $\lceil \cdot \rceil$, where $z + 0.5$ rounds to either $z$ or $z+1$ without affecting correctness) [S1]:
    $$ \bar{t}_0 = \left\lceil \frac{t_k}{\tau} \right\rceil $$
    $$ \bar{t}_k = \bar{t}_0 + \left\lceil \frac{\zeta \theta_k}{\tau} \right\rceil \quad \text{for } k = 1, \dots, \lceil K/2 \rceil $$
    $$ \bar{t}_{-k} = 2\bar{t}_0 - \bar{t}_k \quad \text{for } k = 1, \dots, \lceil K/2 \rceil $$
    Here, $\zeta$ is the particle scale parameter [S1].

3.  **Compute Primary Coefficients:** For $k \geq 1$, except for the case where $k=1$ and $d$ is uneven if $K$ is uneven, compute the quantized localized difference coefficients $\bar{a}_{kd}$:
    $$ \bar{a}_{kd} = \left\lceil \frac{\tau^d \mu \alpha \hat{s}_{kd}}{\varsigma \rho \zeta^{d+3}} \right\rceil $$
    Note: The source text contains a typo in the notes section ($\zeta \rho \zeta^{d+3}$) and the main text ($\varsigma \rho \zeta^{d+3}$). The main text derivation (Eq. 19) uses $\varsigma$ in the denominator to scale the coefficient, consistent with the definition of $\varsigma$ as the value quantum scaling factor. The notes section's $\zeta \rho \zeta^{d+3}$ is likely a transcription error for $\varsigma \rho \zeta^{d+3}$ or similar. We follow the explicit formula in Section 4.3 and Eq. 19: $\frac{\tau^d \mu \alpha \hat{s}_{kd}}{\varsigma \rho \zeta^{d+3}}$ [S1].
    
    Compute the symmetric coefficients using shifted evenness:
    $$ \bar{a}_{-kd} = (-1)^{d+1} \bar{a}_{kd} $$

4.  **Handle Middle Knot (Even $K$):** If $K$ is even, there is a middle knot at index $0$. For uneven $d$, compute $\bar{a}_{0d}$ to satisfy the shifted evenness condition:
    $$ \bar{a}_{0d} = -2 \sum_{k=-\lceil K/2 \rceil}^{-1} \sum_{j=d}^D \binom{j}{d} \bar{a}_{kj} (\bar{t}_0 - \bar{t}_k)^{j-d} $$
    If there is no middle knot (i.e., $K$ is odd or the middle knot coefficient is zero), set $\bar{a}_{0d} = 0$ for all $d$ [S1].

5.  **Handle Boundary Coefficients (Odd $K$ or No Middle Knot):** If $K$ is odd, or if the middle knot coefficients are zero, the coefficients for the boundary knots must be computed to ensure compact support. For uneven $d$, compute $\bar{a}_{-1d}$ in decreasing order of $d$:
    $$ \bar{a}_{-1d} = - \sum_{k=-\lceil K/2 \rceil}^2 \bar{a}_{kd} - \sum_{j=d+1}^D \binom{j}{d} \sum_{k=-\lceil K/2 \rceil}^{-1} \bar{a}_{kj} (\bar{t}_0 - \bar{t}_k)^{j-d} $$
    Then compute the symmetric counterpart:
    $$ \bar{a}_{1d} = (-1)^{d+1} \bar{a}_{-1d} $$
    This step ensures that the sum of contributions neutralizes outside the support [S1].

6.  **Recursive Specification (Optional/Alternative View):** The localized coefficients $\tilde{a}_{kd}$ can also be viewed as recursively computed from previous coefficients to define the piecewise polynomial $A_k(t)$:
    $$ \tilde{a}_{(-\lceil K/2 \rceil - 1)d} = 0 $$
    $$ \tilde{a}_{kd} = \sum_{l=-\lceil K/2 \rceil}^k \sum_{j=d}^D \binom{j}{d} \tilde{a}_{lj} (\tilde{t}_k - \tilde{t}_l)^{j-d} $$
    This recursion ensures continuity and the correct polynomial representation within each knot interval [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_quantized_knots_and_coefficients(mu, rho, zeta_particle, alpha, t_k, Lambda, tau, varsigma, K, D, LUT):
    """
    Computes quantized knot positions and coefficients for a particle.
    
    Args:
        mu: Particle mass
        rho: Particle density
        zeta_particle: Particle scale
        alpha: Particle amplitude
        t_k: Base knot parameter
        Lambda: Normalized distance from ray closest point
        tau: Length quantum value
        varsigma: Value quantum scaling factor
        K: Maximum number of non-trivial polynomial pieces
        D: Maximal polynomial degree
        LUT: Look-up table function LUT(Lambda) -> (theta_k, s_hat_kd)
        
    Returns:
        t_bar: Dictionary of quantized knot counts {k: int}
        a_bar: Dictionary of quantized coefficients {(k, d): int}
    """
    
    # Step 1: Lookup
    # Retrieve theta_k and s_hat_kd for the normalized distance closest to Lambda
    theta_k, s_hat_kd = LUT(Lambda)
    
    # Step 2: Compute Quantized Knots
    # Nearest integer rounding function
    def round_nearest(x):
        return int(math.floor(x + 0.5))
    
    t_bar_0 = round_nearest(t_k / tau)
    
    t_bar = {}
    t_bar[0] = t_bar_0
    
    half_K = math.ceil(K / 2)
    
    for k in range(1, half_K + 1):
        # theta_k is assumed to be indexed by k in the LUT
        theta = theta_k[k]
        t_bar[k] = t_bar_0 + round_nearest((zeta_particle * theta) / tau)
        t_bar[-k] = 2 * t_bar_0 - t_bar[k]
        
    # Step 3: Compute Primary Coefficients
    a_bar = {}
    
    for k in range(1, half_K + 1):
        for d in range(D + 1):
            # Exception: k=1 and uneven d if K is uneven
            if k == 1 and K % 2 != 0 and d % 2 != 0:
                continue
                
            s_hat = s_hat_kd[k][d]
            # Formula: ceil( (tau^d * mu * alpha * s_hat_kd) / (varsigma * rho * zeta_particle^(d+3)) )
            numerator = (tau ** d) * mu * alpha * s_hat
            denominator = varsigma * rho * (zeta_particle ** (d + 3))
            
            a_bar[(k, d)] = round_nearest(numerator / denominator)
            a_bar[(-k, d)] = ((-1) ** (d + 1)) * a_bar[(k, d)]
            
    # Step 4: Handle Middle Knot (Even K)
    if K % 2 == 0:
        # Middle knot at index 0
        for d in range(D + 1):
            if d % 2 != 0: # Uneven d
                sum_val = 0.0
                for k in range(-half_K, 0): # -ceil(K/2) to -1
                    for j in range(d, D + 1):
                        binom = math.comb(j, d)
                        diff_t = t_bar[0] - t_bar[k]
                        sum_val += binom * a_bar[(k, j)] * (diff_t ** (j - d))
                a_bar[(0, d)] = -2 * sum_val
            else:
                a_bar[(0, d)] = 0 # Even d coefficients for middle knot are 0? 
                # Source says "for uneven d". It implies even d are handled differently or are 0.
                # Eq 20 is for uneven d. 
                # Note: The source derivation for even K focuses on uneven d for the middle knot constraint.
                # We assume even d are 0 or handled by symmetry if not specified.
                # However, shifted evenness A(t0+t) = A(t0-t) implies specific symmetries.
                # The source explicitly calculates a_0d for uneven d.
                # For even d, the source does not explicitly set a_0d in the "middle knot" section,
                # but Eq 12 defines the recursive structure. 
                # Given the constraint "compact support" and "shifted evenness", 
                # and the fact that a_0d is computed for uneven d to satisfy the condition,
                # we leave even d for middle knot as 0 or derived from recursion if needed.
                # The source text: "for uneven d" is the condition for the formula.
                # We will set even d to 0 for the middle knot as per the explicit constraint focus.
                a_bar[(0, d)] = 0

    # Step 5: Handle Boundary Coefficients (Odd K or No Middle Knot)
    if K % 2 != 0:
        # Odd K: No middle knot, boundary knots at -1 and 1 need adjustment
        # Compute for uneven d in decreasing order
        uneven_d_list = [d for d in range(D, -1, -1) if d % 2 != 0]
        
        for d in uneven_d_list:
            # Formula: a_-1d = - sum_{k=-ceil(K/2)}^2 a_kd - sum_{j=d+1}^D binom(j,d) sum_{k=-ceil(K/2)}^{-1} a_kj (t0 - tk)^(j-d)
            
            # First term: sum_{k=-ceil(K/2)}^2 a_kd
            # Note: The source index goes up to 2. This implies k=2 is included.
            # However, our range is -half_K to half_K.
            # If K=3, half_K=2. Range -2 to 2.
            # The sum is over k from -ceil(K/2) to 2.
            
            sum1 = 0.0
            for k in range(-half_K, 3): # -ceil(K/2) to 2
                if (k, d) in a_bar:
                    sum1 += a_bar[(k, d)]
                else:
                    # If k is out of bounds or not computed, it is 0
                    pass
            
            # Second term: sum_{j=d+1}^D binom(j,d) sum_{k=-ceil(K/2)}^{-1} a_kj (t0 - tk)^(j-d)
            sum2 = 0.0
            for j in range(d + 1, D + 1):
                binom = math.comb(j, d)
                inner_sum = 0.0
                for k in range(-half_K, 0): # -ceil(K/2) to -1
                    if (k, j) in a_bar:
                        diff_t = t_bar[0] - t_bar[k]
                        inner_sum += a_bar[(k, j)] * (diff_t ** (j - d))
                sum2 += binom * inner_sum
                
            a_bar[(-1, d)] = -sum1 - sum2
            a_bar[(1, d)] = ((-1) ** (d + 1)) * a_bar[(-1, d)]
            
    return t_bar, a_bar
```

**Edge Cases and Pitfalls:**
*   **Boundary Error Propagation:** Non-zero coefficients at the boundary of the support ($\tilde{a}_{[\kappa/2]d}$) propagate error to regions where the contribution should be zero, annihilating quantization benefits. This is prevented by enforcing shifted evenness and compact support via the specific computation of boundary coefficients [S1].
*   **Rounding Ambiguity:** The rounding function $\lceil \cdot \rceil$ is defined as nearest integer. The choice of rounding $z + 0.5$ to $z$ or $z+1$ is irrelevant for correctness [S1].
*   **Exception Handling:** The formula for $\bar{a}_{kd}$ excludes $k=1$ and uneven $d$ if $K$ is uneven. These must be handled separately via the boundary coefficient formulas [S1].

## Operating characteristics
*   **Time Complexity:** The method involves a constant-time lookup followed by $O(K \cdot D^2)$ arithmetic operations to compute the coefficients, as it involves nested sums over $k$ and $j$.
*   **Memory Complexity:** Requires storage for $O(K \cdot D)$ coefficients and knots. The look-up table size depends on the number of discrete $\Lambda$ values and $K, D$.
*   **Performance:** Allows exact integer arithmetic operations, avoiding floating-point precision issues during rendering [S1]. This is a significant strength for hardware implementations or fixed-point arithmetic units.
*   **Limitations:** Introduces a quantization error dependent on $\tau$ and $\varsigma$. The error must be managed by choosing appropriate quantum values [S1].
*   **When to use:** When integer-only arithmetic is required or preferred for performance/precision stability in volume rendering.
*   **When not to use:** When floating-point precision is sufficient and the overhead of quantization error management is undesirable.

## Background
This method originates from the work on Particle-Wise Higher-Order SPH Field Approximation for DVR [S1]. It is part of a broader effort to represent Smoothed Particle Hydrodynamics (SPH) fields for Direct Volume Rendering (DVR) using higher-order polynomials. The quantization approach is designed to maintain the mathematical properties (compact support, shifted evenness) of the continuous approximation while mapping them to a discrete, integer-friendly representation.

## Sources
[S1] Particle-Wise Higher-Order SPH Field Approximation for DVR, https://arxiv.org/abs/2401.02896v1, fetched 2026-10-26
