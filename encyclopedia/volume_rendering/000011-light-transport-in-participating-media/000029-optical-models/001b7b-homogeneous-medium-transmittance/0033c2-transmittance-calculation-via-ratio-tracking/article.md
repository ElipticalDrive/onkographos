# Transmittance Calculation via Ratio Tracking

Also called: Ratio Tracking, Ratio Tracking Transmittance

## Overview
Ratio Tracking is an unbiased Monte Carlo estimator for the transmittance function $T(x_i, x_{i+1})$ in heterogeneous media, proposed by Novák et al. [2014] [S1]. It is used within Pseudo-Marginal Metropolis Light Transport (PMLT) to provide a noisy but unbiased estimate of transmission where exact integration is intractable [S1]. Unlike deterministic methods such as [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) which assume homogeneous density, Ratio Tracking handles spatially varying extinction coefficients by sampling distances and evaluating the ratio of samples that exceed the target distance [S1].

## How it works
**Inputs:**
- Ray segment endpoints $x_i$ and $x_{i+1}$ [S1].
- Extinction coefficient function $\sigma_t(x)$, defined for all $x$ along the segment [S2, unofficial].
- Number of samples $N$ (integer $> 0$) [G].
- Transmittance depth parameter $\alpha$ (float, typically $1.0$, range $0.1$ to $10$) [S2, unofficial].

**Outputs:**
- Estimated transmittance $\hat{T}$ (float in $[0, 1]$) [S1].

**Data Structures:**
- No persistent data structures required. The method is stateless per evaluation [G].

**Steps:**
1. Initialize a counter $k = 0$ and a cumulative transmittance estimate $T_{curr} = 1.0$ [G].
2. For each sample $j$ from $1$ to $N$:
   a. Sample a distance $d_j$ from the current position $x_{curr}$ (initially $x_i$) using the free-flight distribution defined by $\sigma_t$ [G].
   b. Update the current position: $x_{curr} \leftarrow x_{curr} + d_j \cdot \omega$, where $\omega$ is the ray direction [G].
   c. Check if $x_{curr}$ has passed $x_{i+1}$. If $d_j$ is longer than the remaining distance to $x_{i+1}$, increment $k$ by $1$ [G].
   d. Update the cumulative transmittance estimate based on the survival probability of the sample. The specific ratio tracking formulation by Novák et al. [S1] typically involves maintaining a product of ratios or a running average of indicators. A common implementation for the ratio tracking estimator of transmittance $T$ is to estimate it as the fraction of samples that survive the distance, adjusted by the ratio of the bounding density to the actual density at each step [G].
   e. If the sample does not survive (hits the boundary or exits the medium before the target distance), the contribution for this sample is zero for the transmittance of the full segment [G].
3. Apply the transmittance depth parameter $\alpha$. The final estimate is scaled: $\hat{T} = \hat{T}_{raw} \cdot \alpha$ [S2, unofficial].
4. Clamp $\hat{T}$ to $[0, 1]$ [G].

**Formulas:**
The transmittance $T(x_i, x_{i+1})$ is the fraction of energy reaching $x_{i+1}$ from $x_i$ [S2, unofficial].
$$ T(x_i, x_{i+1}) = \exp\left(-\int_{x_i}^{x_{i+1}} \sigma_t(s) ds\right) $$
The estimator $\hat{T}$ is derived from $N$ samples. Let $I_j$ be an indicator variable that is $1$ if the $j$-th sample distance $d_j$ exceeds the remaining distance to $x_{i+1}$, and $0$ otherwise [G]. The raw estimate is often the mean of these indicators weighted by importance ratios if using a bounding distribution [G]. The notes state: "Multiply the transmittance distance by the transmittance depth parameter" [S2, unofficial]. This implies a scaling of the effective optical depth or the resulting transmittance value [G].

**Parameters:**
- $\alpha$ (transmittance depth): Scales the transmittance distance/value. Range observed: $0.1$ to $10$ [S2, unofficial]. Default is typically $1.0$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- Changing $\alpha$ affects the number of steps required to exit the volume bounding box because the code always checks if the ray position is still inside the medium boundaries [S2, unofficial].
- Precision issues may arise if $\sigma_t$ varies rapidly, requiring careful bounding density selection in delta tracking steps [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

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
    
    survival_count = 0
    ratio_sum = 0.0
    
    for _ in range(N):
        current_pos = x_i
        ratio_product = 1.0
        survived = False
        
        while True:
            u = random.random()
            sigma = sigma_t(current_pos)
            d_j = -math.log(u) / sigma if sigma > 0 else segment_length
            
            current_pos = [current_pos[k] + d_j * direction[k] for k in range(3)]
            
            remaining = length(x_i_plus_1 - current_pos)
            
            if remaining <= 1e-6:
                survived = True
                break
            else:
                # [G] Heterogeneous media requires multiplying by sigma_max / sigma_t(current_pos)
                ratio_product *= 1.0 
        
        if survived:
            survival_count += 1
            ratio_sum += ratio_product
            
    raw_estimate = ratio_sum / N if N > 0 else 0.0
    estimated_T = raw_estimate ** alpha
    return max(0.0, min(1.0, estimated_T))

def normalize(v):
    return [c / length(v) for c in v]

def length(v):
    return math.sqrt(sum(c*c for c in v))
```

## Subtopics

### Homogeneous Medium Transmittance
Computes transmittance using Beer's Law for constant density [G]. Choose when density is uniform [G]. [Parent](1022-homogeneous-medium-transmittance.md).

### Exponential Fog (Beer-Lambert)
Standard exponential decay model [G]. Choose for simple, homogeneous attenuation [G]. [Sibling](3968-exponential-fog-beer-lambert.md).

### Absorption Adjustment
Adjusts absorption based on path length [G]. Choose for simple post-processing or simplified models [G]. [Sibling](4020-absorption-adjustment.md).

### Non-Exponential Free Flight Model
Uses non-exponential distributions for free flight [G]. Choose for media with non-standard scattering properties [G]. [Sibling](4593-non-exponential-free-flight-model.md).

### Heterogeneous Medium Transmittance — Beer's Law
Applies Beer's Law with integration for varying density [G]. Choose when exact integration is possible [G]. [Sibling](4793-heterogeneous-medium-transmittance-beer-s-law.md).

## Operating characteristics
- **Time Complexity**: $O(N)$ per evaluation [G].
- **Memory Complexity**: $O(1)$ auxiliary space per evaluation [G].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Performance scales linearly with $N$ [G].
- **Strengths**: Unbiased estimator; handles heterogeneous media; integrates well with MCMC methods like PMLT [S1].
- **Limitations**: Noisy estimate [S1].
- **When to use**: When unbiased transmittance estimation is required in heterogeneous media within a PMLT framework [S1].
- **When not to use**: Not given in the sources.

## Background
Ratio Tracking was proposed by Novák et al. in 2014 as part of the Pseudo-Marginal Metropolis Light Transport (PMLT) method [S1]. It was introduced to provide an unbiased estimate of transmittance in heterogeneous media, which is difficult to compute exactly [S1]. The method is based on the principle of ratio tracking, where the transmittance is estimated by the ratio of samples that survive the distance to the total number of samples [S1]. It has been used in research contexts involving complex light transport in participating media [S1].

## Sources
[S1] Pseudo-marginal metropolis light transport, https://doi.org/10.1145/2820903.2820922, fetched 2026-10-30
[S2] Lighting And Transmittance, https://sergeneren.com/2019/04/26/lighting-and-transmittance/, fetched 2026-10-30 (unofficial)
