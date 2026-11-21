# Rejection Sampling for Arbitrary Media

Also called: Rejection sampling for transmittance, Unbiased distance sampling for arbitrary media

## Overview
Rejection Sampling for Arbitrary Media is an unbiased Monte Carlo technique to sample the distance to the next interaction point in heterogeneous participating media, avoiding the bias of classic ray marching [S1]. It generalizes standard distance sampling by allowing arbitrary non-exponential extinction distributions while maintaining the efficiency of delta tracking [S1]. Unlike [Parametric Transmittance Sampling](14526-parametric-transmittance-sampling.md), which relies on parametric approximations, this method uses a constant upper bound for rejection steps to handle heterogeneity [S1].

## How it works
**Inputs:**
- $x_0$: The starting position of the ray.
- $\omega$: The direction of the ray.
- $\sigma_t$: A constant extinction coefficient such that $\sigma_t \ge \sigma_t(x)$ for all $x$ along the ray [S1].
- $\sigma_t(x)$: The function evaluating the spatially varying extinction coefficient at position $x$ [S1].

**Outputs:**
- $t$: The sampled distance to the next interaction point, distributed according to the target density [S1].

**Data Structures:**
- No specific data structures are required beyond the ability to evaluate $\sigma_t(x)$ at arbitrary points along the ray [S1].

**Steps:**
1. Initialize the current distance $t$ to 0.
2. Generate a random number $u_1 \in [0, 1]$ [S1].
3. Sample a candidate distance $t_{cand}$ from the exponential distribution defined by the upper bound $\sigma_t$:
   $$ t_{cand} = -\frac{\ln(u_1)}{\sigma_t} $$
   [S1]
4. Generate a random number $u_2 \in [0, 1]$ [S1].
5. Evaluate the extinction coefficient at the candidate position $x_{cand} = x_0 + t_{cand}\omega$:
   $$ \sigma_{eval} = \sigma_t(x_{cand}) $$
   [S1]
6. Check the rejection condition:
   $$ \text{if } \frac{\sigma_{eval}}{\sigma_t} < u_2 $$
   [S1]
   - If the condition is met, the sample is accepted. Return $t = t_{cand}$ [S1].
   - If the condition is not met, the sample is rejected. Update the current distance $t \leftarrow t + t_{cand}$ and repeat from Step 2 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_distance(x0, omega, sigma_t_max, sigma_t_func):
    """
    Unbiased distance sampling for arbitrary media using rejection sampling.
    
    Args:
        x0 (Point): Starting position.
        omega (Direction): Ray direction.
        sigma_t_max (float): Constant upper bound for extinction coefficient.
        sigma_t_func (function): Function evaluating sigma_t(x) at position x.
        
    Returns:
        float: Distance to the next interaction point.
    """
    t = 0.0
    while True:
        # Sample candidate distance from exponential distribution with rate sigma_t_max
        u1 = random_uniform()
        t_cand = -log(u1) / sigma_t_max
        
        # Evaluate extinction at candidate position
        x_cand = x0 + t_cand * omega
        sigma_eval = sigma_t_func(x_cand)
        
        # Rejection test
        u2 = random_uniform()
        if (sigma_eval / sigma_t_max) < u2:
            return t + t_cand
        
        # Accumulate rejected distance
        t += t_cand
```

**Parameters:**
- $\sigma_t$: The constant extinction coefficient upper bound. It must satisfy $\sigma_t \ge \sigma_t(x)$ for all $x$ along the ray [S1]. A larger $\sigma_t$ increases the number of iterations but remains unbiased [S1]. Determining the global supremum $\sup_{x \in V} \sigma_t(x)$ during preprocessing is common, though determining the maximum along a single ray is more complex [S1].

**Edge Cases and Pitfalls:**
- Choosing a $\sigma_t$ larger than the true supremum increases the number of iterations in the rejection loop but does not introduce bias [S1].
- The method assumes the ability to evaluate $\sigma_t(x)$ at arbitrary points along the ray [S1].

## Subtopics

### [Transport Kernel Evaluation and Sampling](14528-transport-kernel-evaluation-and-sampling.md)
An unbiased Monte Carlo procedure for determining the next interaction point in participating media, supporting both volume scattering and surface interactions. It generalizes standard distance sampling by allowing arbitrary non-exponential extinction distributions (e.g., Gamma-distributed media) while maintaining the efficiency of delta tracking. Unlike Rejection Sampling for Arbitrary Media, which relies on rejection steps to handle heterogeneity, this method uses a pre-computed base medium model to sample optical depth directly, inverting it to find physical distance, and adjusting weights based on surface proximity.

## Operating characteristics
- **Time Complexity:** The expected number of iterations depends on the ratio of the upper bound $\sigma_t$ to the actual extinction coefficient $\sigma_t(x)$. A tighter bound reduces the expected number of rejections [S1].
- **Memory Complexity:** Minimal, requiring only storage for the current distance and random numbers [S1].
- **Strengths:** Unbiased result for arbitrary media distributions [S1]. Simple to implement [S1].
- **Limitations:** Can be inefficient if the upper bound $\sigma_t$ is much larger than the actual extinction coefficient along the ray, leading to many rejections [S1]. Not suitable for media where $\sigma_t(x)$ cannot be evaluated efficiently [S1].
- **When to use:** When unbiased sampling is required for heterogeneous media and the extinction coefficient can be evaluated at arbitrary points [S1].
- **When not to use:** When performance is critical and the media distribution allows for more efficient sampling methods like [Parametric Transmittance Sampling](14526-parametric-transmittance-sampling.md) [S1].

## Background
The method was introduced by Colvins in 1968 [S1]. It provides an unbiased alternative to the biased classic ray marching algorithm [S1]. It is used in global illumination algorithms for participating media to correctly sample the distance to the next interaction point [S1].

## Sources
[S1] Unbiased Global Illumination with Participating Media, http://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/ugiwpm.pdf, fetched 2026-10-30
