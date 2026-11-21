# Free-Flight Based Gradient Estimators

Also called: DT estimators, delta tracking gradient estimators

## Overview
Free-flight based gradient estimators compute gradients of the volume rendering equation with respect to scene parameters (such as density $\sigma_t$ or emission) by leveraging free-flight distance sampling, typically via delta tracking [S1]. They solve the problem of differentiating volumetric light transport in participating media where the medium density is variable or unknown, enabling inverse volume rendering and optimization [S1]. Unlike surface-based gradient methods, these estimators handle the continuous nature of participating media by accumulating transmittance gradients along null-scattering interactions, distinguishing them from homogeneous medium sampling which assumes constant density [S1].

## How it works
The method estimates gradients for the adjoint volume rendering equation using two primary sampling strategies: one proportional to extinction-weighted transmittance ($\sigma_t(t) T(t)$) and, in unbiased variants, one proportional to transmittance alone ($T(t)$) [S1].

**Inputs:**
*   Ray segment defined by origin, direction, and maximum distance $t_s$ (surface intersection or medium boundary).
*   Scene parameters $\theta$ (e.g., $\sigma_t$, emission coefficients).
*   Radiance quantities: $L_s(t)$ (in-scattered radiance), $L_e(t)$ (emitted radiance), $\alpha(t)$ (scattering albedo).

**Outputs:**
*   Gradient estimates $\langle \partial L \rangle$ with respect to $\theta$.

**Data Structures:**
*   Accumulator for transmittance gradients, initialized to zero.
*   Sampled free-flight distance $t$.

**Numbered Steps:**

1.  **Sample Free-Flight Distance:**
    Sample a distance $t$ from the distribution proportional to $\sigma_t(t) T(t)$. This is typically achieved using delta tracking (null-collision method) [S1].
    *   If $t < t_s$: The interaction occurs within the medium.
    *   If $t \geq t_s$: The ray hits the surface or exits the medium.

2.  **Compute Contribution for Medium Interaction ($t < t_s$):**
    If the sampled distance is within the medium, compute the in-scattering gradient contribution:
    $$ \langle \partial L_1^{\text{DT}} \rangle = \frac{\partial_\theta [\sigma_t(t) \alpha(t)]}{\sigma_t(t)} L_s(t) + \alpha(t) \partial_\theta L_s(t) $$
    [S1]

3.  **Compute Contribution for Surface Interaction ($t \geq t_s$):**
    If the ray hits the surface, compute the surface contribution:
    $$ \langle \partial L_2^{\text{DT}} \rangle = \partial_\theta L_e(t_s) + \partial_\theta L_s(t_s) $$
    [S1]

4.  **Accumulate Transmittance Gradients:**
    For all null-scattering interactions (steps taken during delta tracking) at locations $t'$ along the ray segment $(0, \min(t, t_s))$, accumulate the transmittance gradient contribution [S1]:
    $$ \langle \partial L_3^{\text{DT}} \rangle = \begin{cases} -\partial_\theta \sigma_t(t') \alpha(t) L_s(t) & \text{if } t < t_s, \\ -\partial_\theta \sigma_t(t') [L_e(t_s) + L_s(t_s)] & \text{otherwise.} \end{cases} $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_free_flight_gradient(ray, theta, max_dist_ts):
    """
    Computes gradient estimates using standard DT estimators.
    ray: object with origin, direction, and sampling functions
    theta: scene parameters (sigma_t, etc.)
    max_dist_ts: distance to surface/exit
    """
    # Step 1: Sample t ~ sigma_t(t) * T(t) using delta tracking
    t, null_collisions = sample_free_flight_distance(ray, theta, max_dist_ts)
    
    gradient_sum = 0.0
    
    # Step 4: Accumulate transmittance gradients from null collisions
    # Note: null_collisions contains positions t' where null collisions occurred
    for t_prime in null_collisions:
        # Calculate local transmittance gradient contribution
        # This depends on whether we eventually hit surface or medium
        # We determine this after sampling t, but accumulate during sampling
        pass 

    if t < max_dist_ts:
        # Step 2: Medium interaction
        # L_s(t) is in-scattered radiance at t
        # alpha(t) is scattering albedo at t
        term1 = (derivative(theta, sigma_t(t) * alpha(t)) / sigma_t(t)) * L_s(t)
        term2 = alpha(t) * derivative(theta, L_s(t))
        gradient_sum += term1 + term2
        
        # Add transmittance gradient for this path segment
        # The transmittance gradient term depends on the final outcome t
        for t_prime in null_collisions:
            gradient_sum += -derivative(theta, sigma_t(t_prime)) * alpha(t) * L_s(t)
            
    else:
        # Step 3: Surface interaction
        # L_e(t_s) is emitted radiance at surface
        # L_s(t_s) is in-scattered radiance at surface
        gradient_sum += derivative(theta, L_e(max_dist_ts)) + derivative(theta, L_s(max_dist_ts))
        
        # Add transmittance gradient for this path segment
        for t_prime in null_collisions:
            gradient_sum += -derivative(theta, sigma_t(t_prime)) * (L_e(max_dist_ts) + L_s(max_dist_ts))
            
    return gradient_sum
```

**Parameters:**
*   $\sigma_t(t)$: Total extinction coefficient.
*   $\alpha(t)$: Scattering albedo, $\sigma_s(t) / \sigma_t(t)$ [S1].
*   $L_s(t)$: In-scattered radiance [S1].
*   $L_e(t)$: Emitted radiance [S1].

**Edge Cases and Pitfalls:**
*   **Bias at Zero Density:** If $\sigma_t(t) = 0$, the estimator $\langle \partial L_1^{\text{DT}} \rangle$ is never sampled because the sampling probability is zero, even though the integrand for in-scattering gradients is non-zero. This causes bias, preventing optimization from escaping states where $\sigma_t=0$ [S1].
*   **High Variance at Low Density:** If $\sigma_t(t)$ is close to zero, the term $1/\sigma_t(t)$ in Equation (8) leads to large outliers. The variance is unbounded for a homogeneous slab [S1].
*   **Clamping Limitation:** Unlike surface BSDF optimization, clamping $\sigma_t$ to a small nonzero value is not a valid solution here. Lowering $\sigma_t$ increases brightness, which is the wrong direction if the background is darker than the target, leading to incorrect gradient updates [S1].

## Subtopics

### Unbiased Estimators (DRT)
DRT (Delta Tracking with Transmittance sampling) replaces the problematic $\langle \partial L_1^{\text{DT}} \rangle$ estimator with tailored estimators that sample distances proportional to transmittance $T(t')$ to address bias and variance issues [S1]. It samples $t' \sim T(t')$ to estimate in-scattering gradients, while keeping $t \sim \sigma_t T$ for other terms [S1]. The in-scattering gradient estimator is $\langle \partial L_{1a}^{\text{DRT}} \rangle := L_s(t') \partial_\theta [\sigma_t(t') \alpha(t')]$ and the albedo gradient estimator is $\langle \partial L_{1b}^{\text{DRT}} \rangle := \alpha \partial_\theta L_s(t)$ [S1]. A key drawback is that because $t'$ is sampled from a different strategy than the path construction, path replay backpropagation cannot efficiently produce an estimate of incident illumination $L_i$ at $t'$, potentially raising time complexity to $O(n^2)$ [S1].

## Operating characteristics
*   **Complexity:** Standard DT estimators are $O(n)$ per path, where $n$ is the number of null collisions. DRT estimators may incur $O(n^2)$ complexity in the worst case due to the need for recursive path starting at sampled points $t'$ [S1].
*   **Variance:** Standard DT estimators suffer from unbounded variance when density is low. DRT estimators reduce variance and eliminate bias in these regions [S1].
*   **Strengths:** Handles variable density media effectively; integrates naturally with delta tracking.
*   **Limitations:** Standard DT is biased and high-variance in low-density regions; DRT is computationally more expensive due to sampling strategy mismatch [S1].

## Background
The method was introduced by Tregan et al. in 2022 in the paper "Unbiased inverse volume rendering with differential trackers" [S1]. It addresses limitations in previous gradient estimation methods for volume rendering, specifically the bias and variance issues associated with applying primal sampling techniques directly to the adjoint problem [S1]. The work builds upon earlier foundations in differential volume rendering by Gkioulekas et al. [2013] [S1].

## Sources
[S1] Unbiased inverse volume rendering with differential trackers, https://doi.org/10.1145/3528223.3530073, fetched 2026-11-01
