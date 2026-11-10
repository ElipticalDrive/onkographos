# Transmittance Calculation via Ratio Tracking

Also called: Ratio Tracking, Ratio Tracking Transmittance

## Overview
Ratio Tracking is an unbiased Monte Carlo estimator for the transmittance function $T(x_i, x_{i+1})$ in heterogeneous media, proposed by Novák et al. [2014] [S1]. It is used within Pseudo-Marginal Metropolis Light Transport (PMLT) to provide a noisy but unbiased estimate of transmission where exact integration is intractable [S1]. Unlike deterministic methods such as [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) which assume homogeneous density, Ratio Tracking handles spatially varying extinction coefficients by sampling distances and evaluating the ratio of samples that exceed the target distance [S1].

## How it works
**Inputs:**
- Ray segment endpoints $x_i$ and $x_{i+1}$.
- Extinction coefficient function $\sigma_t(x)$, defined for all $x$ along the segment.
- Number of samples $N$ (integer $> 0$).
- Transmittance depth parameter $\alpha$ (float, typically $1.0$, range $0.1$ to $10$) [S2, unofficial].

**Outputs:**
- Estimated transmittance $\hat{T}$ (float in $[0, 1]$).

**Data Structures:**
- No persistent data structures required. The method is stateless per evaluation.

**Steps:**
1. Initialize a counter $k = 0$ and a cumulative transmittance estimate $T_{curr} = 1.0$ [G].
2. For each sample $j$ from $1$ to $N$:
   a. Sample a distance $d_j$ from the current position $x_{curr}$ (initially $x_i$) using the free-flight distribution defined by $\sigma_t$. In practice, this often involves sampling from a bounding distribution or using delta tracking if $\sigma_t$ is heterogeneous [G].
   b. Update the current position: $x_{curr} \leftarrow x_{curr} + d_j \cdot \omega$, where $\omega$ is the ray direction.
   c. Check if $x_{curr}$ has passed $x_{i+1}$. If $d_j$ is longer than the remaining distance to $x_{i+1}$, increment $k$ by 1.
   d. Update the cumulative transmittance estimate based on the survival probability of the sample. The specific ratio tracking formulation by Novák et al. [S1] typically involves maintaining a product of ratios or a running average of indicators. A common implementation for the ratio tracking estimator of transmittance $T$ is to estimate it as the fraction of samples that survive the distance, adjusted by the ratio of the bounding density to the actual density at each step. However, the provided notes simplify the core concept: "evaluate the ratio of samples corresponding to a distance longer than d" [S1].
   e. If the sample does not survive (hits the boundary or exits the medium before the target distance), the contribution for this sample is zero for the transmittance of the full segment.
3. Apply the transmittance depth parameter $\alpha$. The final estimate is scaled: $\hat{T} = \hat{T}_{raw} \cdot \alpha$ [S2, unofficial].
4. Clamp $\hat{T}$ to $[0, 1]$ [G].

**Formulas:**
The transmittance $T(x_i, x_{i+1})$ is the fraction of energy reaching $x_{i+1}$ from $x_i$ [S2, unofficial].
$$ T(x_i, x_{i+1}) = \exp\left(-\int_{x_i}^{x_{i+1}} \sigma_t(s) ds\right) $$
The estimator $\hat{T}$ is derived from $N$ samples. Let $I_j$ be an indicator variable that is 1 if the $j$-th sample distance $d_j$ exceeds the remaining distance to $x_{i+1}$, and 0 otherwise. The raw estimate is often the mean of these indicators weighted by importance ratios if using a bounding distribution. The notes state: "Multiply the transmittance distance by the transmittance depth parameter" [S2, unofficial]. This implies a scaling of the effective optical depth or the resulting transmittance value.

**Parameters:**
- $\alpha$ (transmittance depth): Scales the transmittance distance/value. Range observed: $0.1$ to $10$ [S2, unofficial]. Default is typically $1.0$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- Changing $\alpha$ affects the number of steps required to exit the volume bounding box because the code always checks if the ray position is still inside the medium boundaries [S2, unofficial].
- Precision issues may arise if $\sigma_t$ varies rapidly, requiring careful bounding density selection in delta tracking steps [G].
- If $N$ is too small, the estimator variance is high, leading to noisy results [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ratio_tracking_transmittance(x_i, x_i_plus_1, sigma_t, N, alpha=1.0):
    """
    Estimate transmittance using Ratio Tracking.
    
    Args:
        x_i: Start point of the ray segment.
        x_i_plus_1: End point of the ray segment.
        sigma_t: Function returning extinction coefficient at a point.
        N: Number of Monte Carlo samples.
        alpha: Transmittance depth parameter.
        
    Returns:
        Estimated transmittance.
    """
    direction = normalize(x_i_plus_1 - x_i)
    segment_length = length(x_i_plus_1 - x_i)
    
    # Note: The exact sampling of d_j from sigma_t requires a bounding density
    # if sigma_t is heterogeneous. This is a placeholder for the delta tracking step.
    # In a full implementation, one would sample d_j such that P(D > d) = exp(-int_0^d sigma_t(s) ds)
    # or use a bounding function sigma_max >= sigma_t(x).
    
    survival_count = 0
    
    for j in range(N):
        # Sample a distance d_j. 
        # For homogeneous media, d_j = -ln(u) / sigma_t(x_i)
        # For heterogeneous, use delta tracking with a bounding density.
        # Here we assume a helper function sample_free_flight exists.
        d_j = sample_free_flight(x_i, direction, sigma_t, segment_length)
        
        # Check if the sample distance exceeds the remaining distance to the target
        if d_j >= segment_length:
            survival_count += 1
            
    # Raw estimate is the fraction of samples that survived the full distance
    raw_estimate = survival_count / N
    
    # Apply transmittance depth parameter
    # Note: The source [S2, unofficial] says "Multiply the transmittance distance by the transmittance depth parameter".
    # This is ambiguous. It could mean scaling the optical depth or the result.
    # Given the context of "changing transmittance without changing density", 
    # scaling the result is the most direct interpretation of the provided text.
    # However, physically, transmittance is exp(-optical_depth). 
    # If alpha scales the distance, it scales the optical depth.
    # T_new = exp(-alpha * optical_depth) = T_old^alpha.
    # The source says "Multiply the transmittance distance", which might mean 
    # the estimated distance d_j is scaled? Or the resulting T?
    # Let's follow the literal "Multiply the transmittance distance" if it refers to the input distance.
    # But the output is transmittance. 
    # Re-reading [S2, unofficial]: "I’m just multiplying the transmittance distance with the parameter."
    # This likely refers to the optical depth calculation or the distance in the exponent.
    # If we scale the distance in the exponent: T = exp(-alpha * integral(sigma_t)).
    # So, raw_estimate is an estimator for T. 
    # If we want to apply alpha to the distance, we effectively compute T^alpha.
    
    # Implementation based on "Multiply the transmittance distance":
    # If the estimator estimates T, and we want to scale the distance, 
    # we raise the estimate to the power of alpha? Or multiply the distance by alpha?
    # Let's assume the standard interpretation: alpha scales the effective extinction/distance.
    # T_est = raw_estimate ** alpha
    
    # However, the source says "Multiply... distance". 
    # If d is the distance, and we multiply d by alpha, the optical depth increases by alpha.
    # T = exp(-alpha * tau). 
    # So we should compute the estimator for alpha * tau? 
    # Or simply scale the result? 
    # Given the ambiguity, and the note "changes the number of steps", 
    # it implies the sampling process is affected.
    # Let's stick to the simplest interpretation of the text: 
    # The transmittance value is scaled or the distance in the exponent is scaled.
    # If we scale the distance in the exponent, T_new = T_old^alpha.
    
    # But wait, [S2, unofficial] says "Multiply the transmittance distance with the parameter".
    # If "transmittance distance" means the distance d_j, then we are sampling differently.
    # Let's assume the parameter alpha is applied to the final transmittance estimate 
    # as a power, which is equivalent to scaling the optical depth (distance).
    
    estimated_T = raw_estimate ** alpha
    
    return clamp(estimated_T, 0.0, 1.0)

def sample_free_flight(x, direction, sigma_t, max_dist):
    """
    Placeholder for sampling a free-flight distance from a heterogeneous medium.
    In a real implementation, this would use delta tracking.
    """
    # For homogeneous media:
    # sigma = sigma_t(x)
    # u = random()
    # d = -log(u) / sigma
    # return min(d, max_dist)
    
    # For heterogeneous media, this is complex and not fully detailed in sources.
    # [G] This function requires a bounding density sigma_max >= sigma_t(x) for all x.
    # It uses the rejection sampling method of delta tracking.
    raise NotImplementedError("Heterogeneous free-flight sampling requires delta tracking implementation details not provided in sources.")
```

## Subtopics

### Homogeneous Medium Transmittance
Computes transmittance using Beer's Law for constant density. Choose when density is uniform. [Parent](1022-homogeneous-medium-transmittance.md).

### Exponential Fog (Beer-Lambert)
Standard exponential decay model. Choose for simple, homogeneous attenuation. [Sibling](3968-exponential-fog-beer-lambert.md).

### Absorption Adjustment
Adjusts absorption based on path length. Choose for simple post-processing or simplified models. [Sibling](4020-absorption-adjustment.md).

### Non-Exponential Free Flight Model
Uses non-exponential distributions for free flight. Choose for media with non-standard scattering properties. [Sibling](4593-non-exponential-free-flight-model.md).

### Heterogeneous Medium Transmittance — Beer's Law
Applies Beer's Law with integration for varying density. Choose when exact integration is possible. [Sibling](4793-heterogeneous-medium-transmittance-beer-s-law.md).

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of samples. Each sample requires a free-flight distance calculation, which can be $O(1)$ for homogeneous media or more complex for heterogeneous media using delta tracking [S1].
- **Memory Complexity**: $O(1)$ auxiliary space per evaluation [S1].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Performance scales linearly with $N$. Variance decreases as $1/\sqrt{N}$ [S1].
- **Strengths**: Unbiased estimator; handles heterogeneous media; integrates well with MCMC methods like PMLT [S1].
- **Limitations**: Noisy estimate; requires careful implementation of delta tracking for heterogeneous media; parameter $\alpha$ affects convergence behavior [S2, unofficial].
- **When to use**: When unbiased transmittance estimation is required in heterogeneous media within a PMLT framework [S1].
- **When not to use**: When deterministic, noise-free results are needed, or when the medium is homogeneous and Beer's Law is sufficient [S2, unofficial].

## Background
Ratio Tracking was proposed by Novák et al. in 2014 as part of the Pseudo-Marginal Metropolis Light Transport (PMLT) method [S1]. It was introduced to provide an unbiased estimate of transmittance in heterogeneous media, which is difficult to compute exactly. The method is based on the principle of ratio tracking, where the transmittance is estimated by the ratio of samples that survive the distance to the total number of samples [S1]. It has been used in research contexts involving complex light transport in participating media [S1].

## Sources
[S1] Pseudo-marginal metropolis light transport, https://doi.org/10.1145/2820903.2820922, fetched 2026-10-30
[S2] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
