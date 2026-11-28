# AdaptiveRatioTracking

Also called: importance-sampled form of ratio tracking

## Overview
AdaptiveRatioTracking is an importance-sampled variant of ratio tracking that improves transmittance estimation by decoupling the medium's combined free-flight distribution from the sampling distribution [S1]. It uses a proxy for the null-free-flight distribution to favor larger free-flight distances, addressing the suboptimal variance of traditional ratio tracking which samples proportional to the total extinction [S1].

## How it works
**Inputs:** Current iteration majorant $\mu_t^{(k)}$, origin $o$, direction $\omega$, medium length $d$.
**Outputs:** Transmittance $T$, next iteration majorant $\mu_t^{(k+1)}$.

**Data Structures:**
*   $\mu_{MC}$: Monte Carlo extinction coefficient, initialized to $\mu_t^{(k)}$. Acts as a proxy for the unknown spatially varying null extinction $\mu_n(x)$ [S1].
*   $t$: Current distance traveled along the ray, initialized to 0.
*   $T$: Accumulated transmittance, initialized to 1.

**Algorithm Steps:**
1.  Initialize $T \leftarrow 1$, $t \leftarrow 0$, and $\mu_{MC} \leftarrow \mu_t^{(k)}$ [S1].
2.  While $t < d$ and $\mu_{MC} > 0$:
    a. Sample distance $x$ from the exponential distribution defined by $\mu_{MC}$, clamped to the remaining ray length:
       $$x \leftarrow \min\left(\frac{-\ln(1-u)}{\mu_{MC}}, d-t\right)$$
       where $u$ is a uniform random number in $[0, 1)$ [S1].
    b. Update transmittance for the null scattering component:
       $$T \leftarrow T \cdot \exp((\mu_{MC} - \mu_t^{(k)}) \cdot x)$$
       This term accounts for the difference between the sampling distribution ($\mu_{MC}$) and the true total extinction ($\mu_t^{(k)}$) over the sampled distance [S1].
    c. If $x \neq d - t$ (a null collision occurred within the medium, not at the boundary):
       i. Advance position: $t \leftarrow t + x$ [S1].
       ii. Evaluate the real extinction $\mu_r^+$ and the next iteration majorant $\mu_t^{(k+1)}$ at the new position $o + t\omega$ using `EvaluateDensity` [S1].
       iii. Update transmittance for the real scattering component:
       $$T \leftarrow T \cdot \frac{\mu_t^{(k)} - \mu_r^+}{\mu_{MC}}$$
       This weight corrects for the ratio of real extinction to the sampling extinction [S1].
       iv. Update the Monte Carlo extinction proxy for the next step:
       $$\mu_{MC} \leftarrow \mu_t^{(k)} - \mu_r^+$$
       This value represents the null extinction at the current position, serving as the majorant for the next free-flight distance [S1].
    d. If $x = d - t$ (the ray exited the medium), terminate the loop [S1].
3.  Return $T$ and $\mu_t^{(k+1)}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def AdaptiveRatioTracking(mu_t_k, o, omega, d):
    T = 1.0
    t = 0.0
    mu_MC = mu_t_k
    
    while t < d and mu_MC > 0:
        # Sample distance from exponential distribution with rate mu_MC
        u = random_uniform(0, 1)
        x = -log(1 - u) / mu_MC
        x = min(x, d - t)
        
        # Update transmittance for null scattering
        T = T * exp((mu_MC - mu_t_k) * x)
        
        if x != d - t:
            # Null collision occurred inside the medium
            t = t + x
            
            # Evaluate real extinction and update majorant
            # EvaluateDensity returns (mu_r_plus, mu_t_next)
            mu_r_plus, mu_t_next = EvaluateDensity(mu_t_k, o, omega, t)
            
            # Update transmittance for real scattering weight
            # Note: mu_t_k is the total extinction at the START of this segment
            # mu_r_plus is the real extinction at the END of this segment (position t)
            # The weight is (mu_t_k - mu_r_plus) / mu_MC
            # However, standard ratio tracking weights are typically (mu_t - mu_r) / mu_null
            # Here mu_null = mu_t - mu_r. The sampling rate is mu_MC.
            # The source formula is: T * (mu_t^(k) - mu_r^+) / mu_MC
            T = T * (mu_t_k - mu_r_plus) / mu_MC
            
            # Update MC extinction proxy for next iteration
            mu_MC = mu_t_k - mu_r_plus
            
            # Update majorant for next iteration
            mu_t_k = mu_t_next
        else:
            # Ray exited the medium
            break
            
    return T, mu_t_k
```

**Parameters:**
*   $\mu_t^{(k)}$: The bounding extinction coefficient (majorant) from the previous iteration or initialization [S1].
*   $\mu_{MC}$: The Monte Carlo extinction used for sampling. It is updated iteratively to track the local null extinction $\mu_n(x) = \mu_t(x) - \mu_r(x)$ [S1].

**Edge Cases and Pitfalls:**
*   **Bias/Instability:** The method assumes $\mu_{MC}$ acts as a proxy for the unknown spatially varying $\mu_n(x)$. If $\mu_{MC}$ is not a bounding extinction for the true density (i.e., $\mu_{MC} < \mu_n(x)$), the estimator may become biased or unstable [S1]. The progressive framework aims to converge to a valid bounding $\mu_t$ over time [S1].
*   **Zero Extinction:** The loop terminates if $\mu_{MC} \le 0$, preventing division by zero in the sampling step [S1].

## Subtopics

### Importance Sampling Extinctions
An importance sampling strategy for ratio tracking that samples distances proportional to the null density times the transmittance, derived from a theoretically zero-variance estimator [S1]. The zero-variance sampling distribution is defined as $p_{MC}(t) = \mu_n \hat{Tr}(x, t)$, where $\hat{Tr}(x, t) = e^{-\mu_n t} = \frac{\overline{\text{Tr}}(x, t)}{\text{Tr}(x, t)}$ [S1]. Unlike traditional ratio tracking which samples proportional to $\mu_t(t)$, this technique samples proportional to $\mu_n(t)$ [S1]. This approach favors larger free-flight distances because it samples from the null-free-flight distribution rather than the total free-flight distribution [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The method improves estimation over traditional ratio tracking by decoupling the sampling distribution from the total free-flight distribution, favoring larger free-flight distances [S1].
*   **Strengths:** Reduces variance compared to standard ratio tracking by using an importance-sampled distribution closer to the zero-variance solution [S1].
*   **Limitations:** Requires a progressive framework to adapt $\mu_t$ and ensure $\mu_{MC}$ remains a valid bounding extinction; incorrect bounding can lead to bias [S1].

## Background
AdaptiveRatioTracking was introduced in the context of progressive null-tracking for volumetric rendering [S1]. It builds on the concept of ratio tracking with null scattering [1044-ratio-tracking-with-null-scattering] and addresses the limitation that traditional null-tracking techniques sample distances proportional to the combined free-flight distribution, which is not the most optimal choice for variance reduction [S1]. The method is motivated by zero-variance analysis of ratio tracking and the progressive technique for adapting $\mu_t$ during renders [S1].

## Sources
[S1] Progressive null-tracking for volumetric rendering, https://www.yiningkarlli.com/projects/progressivenulltracking/progressivenulltracking.pdf, fetched 2026-10-29
