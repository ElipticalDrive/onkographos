# Weighted Delta Tracking

Also called: Free-Path-Sampling Residual Ratio Tracking, Non-bounding Delta Tracking, Weighted Monte Carlo Free-Path Sampling, WgtDeltaTracking

## Overview
Weighted Delta Tracking is a Monte Carlo free-path-sampling method that removes the requirement for the sampling coefficient $\bar{\mu}(\mathbf{x})$ to be a majorant of the true extinction coefficient $\mu_t(\mathbf{x})$ [S1]. Unlike standard Delta Tracking, which requires $\bar{\mu}(\mathbf{x}) \ge \mu_t(\mathbf{x})$ to ensure unbiased rejection sampling, Weighted Delta Tracking allows $\bar{\mu}(\mathbf{x}) < \mu_t(\mathbf{x})$ by compensating for the resulting stretching of the free-path probability density function (PDF) with statistical weights [S1]. This enables the use of simpler, analytically tractable sampling distributions at the cost of increased variance due to noisy weights [S1]. It builds on [Delta tracking](985-delta-tracking.md) but relaxes the majorant constraint [S1].

## How it works
**Inputs:**
- Current position $\mathbf{x}$ and direction $\omega$ [G].
- True extinction coefficient function $\mu_t(\mathbf{x}) = \mu_a(\mathbf{x}) + \mu_s(\mathbf{x})$ [G].
- Bounding extinction coefficient function $\bar{\mu}(\mathbf{x})$, which is not required to be a majorant ($\bar{\mu}(\mathbf{x})$ may be less than $\mu_t(\mathbf{x})$) [S1].
- Null interaction coefficient $\mu_n(\mathbf{x})$ (optional, used to define branching probabilities) [S1].
- Emission coefficient $\mu_e(\mathbf{x})$ (or source radiance $L_e$) [G].
- Scattering phase function $p_s(\omega' \to \omega)$ [G].
- Current path throughput weight $w_{in}$ [G].

**Outputs:**
- Updated position $\mathbf{x}'$ [G].
- New direction $\omega'$ (if scattered) [G].
- Updated throughput weight $w_{out}$ [G].
- Boolean indicating if the ray terminated (absorbed) or continued [G].

**Data Structures:**
- No specific complex data structures are required beyond standard scalar/vector storage for coefficients and random numbers [G]. The method relies on evaluating $\mu_t(\mathbf{x})$, $\bar{\mu}(\mathbf{x})$, and $\mu_n(\mathbf{x})$ at arbitrary points [G].

**Parameters:**
- $\mu_t^c(\mathbf{x})$: Control extinction coefficient. Typically a constant, e.g., $avg(\mu_t(\mathbf{x}))$ [S1].
- $\bar{\mu}^r(\mathbf{x})$: Residual bounding coefficient. Typically a constant, e.g., $max(|\mu_t^r(\mathbf{x})|)$ [S1].
- $\bar{\mu}(\mathbf{x}) = \mu_t^c(\mathbf{x}) + \bar{\mu}^r(\mathbf{x})$ [S1].

**Formulas:**
The branching probabilities are defined as [S1]:
$$ P_a(\mathbf{x}) = \frac{\mu_a(\mathbf{x})}{\mu_t(\mathbf{x}) + |\mu_n(\mathbf{x})|} $$
$$ P_s(\mathbf{x}) = \frac{\mu_s(\mathbf{x})}{\mu_t(\mathbf{x}) + |\mu_n(\mathbf{x})|} $$
$$ P_n(\mathbf{x}) = \frac{|\mu_n(\mathbf{x})|}{\mu_t(\mathbf{x}) + |\mu_n(\mathbf{x})|} $$

The probability of sampling a real collision (absorption or scattering) is [S1]:
$$ P_t(\mathbf{x}) = \frac{\mu_t^c(\mathbf{x})}{\bar{\mu}(\mathbf{x})} $$

The probability of sampling a null collision is [S1]:
$$ P_n(\mathbf{x}) = \frac{\bar{\mu}^r(\mathbf{x})}{\bar{\mu}(\mathbf{x})} $$

Note: In the general weighted formulation, $P_a, P_s, P_n$ sum to 1. In the specific "Free-Path-Sampling Residual Ratio Tracking" variant (a special case), the branching is often simplified to a binary choice between "real collision" (from control volume) and "null collision" (from residual volume) [S1].

The local collision weight for event type $\star \in \{a, s, n\}$ is [S1]:
$$ w_\star(\mathbf{x}) = \frac{\mu_\star(\mathbf{x})}{\bar{\mu}(\mathbf{x}) P_\star(\mathbf{x})} $$

The path throughput is updated as [S1]:
$$ w(\mathbf{x}) = L_e(\mathbf{x}_k, \omega_k) w_a(\mathbf{x}_k) \prod_{j=1}^{k-1} w_j(\mathbf{x}_j) $$
where the product accumulates weights from previous steps [S1]. For a single step, if a collision of type $\star$ occurs at $\mathbf{x}$, the weight is multiplied by $w_\star(\mathbf{x})$ [S1].

**Steps:**
1. Sample a free path distance $t$ from the PDF defined by $\bar{\mu}(\mathbf{x})$ along direction $\omega$ [S1]. Since $\bar{\mu}$ is often constant or simple, this is typically done analytically:
   $$ t = -\frac{\ln(1 - \zeta)}{\bar{\mu}(\mathbf{x})} $$
   where $\zeta \sim U[0, 1]$ [S1].
2. Update position:
   $$ \mathbf{x}_{new} = \mathbf{x} - t \times \omega $$
   [S1]
3. Evaluate $\mu_t(\mathbf{x}_{new})$, $\bar{\mu}(\mathbf{x}_{new})$, and $\mu_n(\mathbf{x}_{new})$ at the new position [S1].
4. Determine the collision type.
   - If using the general branching probabilities $P_a, P_s, P_n$:
     - Generate $\xi \sim U[0, 1]$.
     - If $\xi < P_a(\mathbf{x}_{new})$, event is Absorption [S1].
     - Else if $\xi < P_a(\mathbf{x}_{new}) + P_s(\mathbf{x}_{new})$, event is Scattering [S1].
     - Else, event is Null Collision [S1].
   - If using the Residual Ratio Tracking variant (binary):
     - Generate $\xi \sim U[0, 1]$.
     - If $\xi < P_t(\mathbf{x}_{new})$, event is Real Collision (Absorption or Scattering) [S1].
       - Further branch between Absorption and Scattering based on the ratio $\mu_a / \mu_t$ or similar physical ratio, weighted appropriately [S1].
     - Else, event is Null Collision [S1].
5. Update weight and handle event:
   - **Absorption**: Multiply current throughput $w$ by $w_a(\mathbf{x}_{new})$. Add contribution $w \cdot L_e(\mathbf{x}_{new}, \omega)$ to the pixel/radiance estimate. Terminate the path [S1].
   - **Scattering**: Multiply current throughput $w$ by $w_s(\mathbf{x}_{new})$. Update direction $\omega$ according to phase function. Continue tracking from $\mathbf{x}_{new}$ with new $\omega$ and updated $w$ [S1].
   - **Null Collision**: Multiply current throughput $w$ by $w_n(\mathbf{x}_{new})$. Note that $w_n$ can be negative if $\mu_n$ is negative, causing sign flips in throughput. Continue tracking from $\mathbf{x}_{new}$ with same $\omega$ and updated $w$ [S1].
6. Repeat from Step 1 until termination [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def weighted_delta_tracking(x, omega, w_in, mu_t_func, mu_bar_func, mu_n_func, mu_a_func, mu_s_func, L_e_func, phase_sample_func):
    w = w_in
    x_curr = x
    
    while True:
        # 1. Sample free path from bounding coefficient
        mu_bar = mu_bar_func(x_curr)
        if mu_bar <= 0:
            break 
            
        zeta = random.random()
        t = -math.log(1 - zeta) / mu_bar
        
        # 2. Update position
        x_next = x_curr - t * omega
        
        # 3. Evaluate coefficients at new position
        mu_t = mu_t_func(x_next)
        mu_a = mu_a_func(x_next)
        mu_s = mu_s_func(x_next)
        mu_n = mu_n_func(x_next)
        
        # 4. Determine collision type using general branching probabilities
        denom = mu_t + abs(mu_n)
        if denom == 0:
            if L_e_func(x_next, omega) == 0:
                break
            else:
                w *= L_e_func(x_next, omega)
                break
                
        P_a = mu_a / denom
        P_s = mu_s / denom
        P_n = abs(mu_n) / denom
        
        xi = random.random()
        
        # 5. Handle event
        if xi < P_a:
            # Absorption
            w_a = mu_a / (mu_bar * P_a)
            w *= w_a
            w *= L_e_func(x_next, omega)
            break 
            
        elif xi < P_a + P_s:
            # Scattering
            w_s = mu_s / (mu_bar * P_s)
            w *= w_s
            omega_new = phase_sample_func(omega)
            x_curr = x_next
            omega = omega_new
            
        else:
            # Null Collision
            w_n = abs(mu_n) / (mu_bar * P_n)
            w *= w_n
            x_curr = x_next
```

**Edge Cases and Pitfalls:**
- If $\bar{\mu}(\mathbf{x}) < \mu_t(\mathbf{x})$, the weight $w_\star$ can become large, leading to high variance [S1].
- If $\mu_n(\mathbf{x})$ is negative, the weight $w_n$ is negative, causing the throughput to alternate signs, which increases noise [S1].
- If $\mu_t^c(\mathbf{x})$ differs drastically from $\mu_t(\mathbf{x})$, extreme weights result [S1].
- Precision issues may arise if $\bar{\mu}(\mathbf{x})$ is very small.

## Operating characteristics
- **Time Complexity:** Proportional to the number of collisions (real and null). Since $\bar{\mu}$ can be smaller than $\mu_t$, the number of null collisions may increase or decrease depending on the choice of $\bar{\mu}$, but generally, the method allows for fewer null collisions than standard delta tracking if $\bar{\mu}$ is chosen to be closer to $\mu_t$ but not a majorant. However, the cost per step is similar.
- **Memory Complexity:** $O(1)$ per ray, as no large data structures are needed beyond the coefficient functions.
- **Variance:** Variance increases rapidly if $\bar{\mu}(\mathbf{x})$ is not a tight bound or if $\mu_n$ is negative [S1]. The statistical error is higher than standard delta tracking when $\bar{\mu}$ is significantly smaller than $\mu_t$ due to noisy weights [S1].
- **Strengths:** Allows use of simple, analytically tractable sampling distributions (e.g., constant $\bar{\mu}$) even in highly heterogeneous media where finding a tight majorant is difficult or expensive [S1]. Reduces the average number of null collisions compared to standard delta tracking with a loose majorant [S1].
- **Limitations:** Higher variance due to weight fluctuations. Requires careful tuning of $\bar{\mu}$ and $\mu_n$ to balance null collision rate and weight variance [S1].
- **When to use:** When finding a tight majorant for $\mu_t$ is computationally prohibitive or impossible, and a simple approximation $\bar{\mu}$ is available.
- **When not to use:** When high precision is required and the variance from weights is unacceptable, or when a tight majorant is easily available.

## Background
Weighted Delta Tracking was introduced as a way to remove the burden of finding tight majorants for the extinction coefficient in heterogeneous volumes [S1]. It is based on the concept of weighted Monte Carlo schemes, with origins in nuclear transport literature (Carter et al. 1972; Cramer 1978) and later applied to rendering (Galtier et al. 2013) [S1]. Novák et al. [2014] discussed a specific variant called Free-Path-Sampling Residual Ratio Tracking, which is a special case of Weighted Delta Tracking [S1]. The method is used in volume rendering to handle participating media with complex, heterogeneous density distributions.

## Sources
[S1] Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
