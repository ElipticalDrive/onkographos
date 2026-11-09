# Whole Volume Distance Sampling

Also called: Histogram-based Transmittance Sampling, Importance Sampling for Volume Rendering, Monte Carlo with Whole Volume Distance Sampling

## Overview
Whole Volume Distance Sampling (WVDS) is a Monte Carlo importance sampling strategy for estimating the Radiative Transfer Equation (VRE) in participating media. It addresses the limitation of standard distance sampling, which only accounts for transmittance along the current path segment, by distributing samples proportional to the transmittance integrand across the entire camera line segment geometry [S1]. This approach reduces variance in occluded areas compared to ray marching and converges in a single sample in unoccluded sections [S1].

## How it works
**Inputs:** Camera ray origin $\mathbf{o}$ and direction $\tilde{\omega}_o$, light direction $\tilde{\omega}_i$, medium extinction coefficient $\sigma_t$, and the geometric definition of the medium boundaries (typically axis-aligned boxes or trapezoidal segments formed by the intersection of camera and light rays) [S1].
**Outputs:** A sampled distance $t$ along the camera ray and the corresponding transmittance/weight [S1].

**Data Structures:**
The method relies on a histogram of trapezoid segment integrals. Each segment $i$ is defined by:
- $\tilde{\mathbf{p}}_i$: Corner positions of the trapezoid segment [S1].
- $\tilde{\mathbf{w}}_i$: Distance vectors between corners [S1].
- $\mathcal{I}_i$: The integral of transmittance for segment $i$ [S1].
- $t_{C,i-1}$: Cumulative distance or offset for previous segments [S1].
- $d_i$: Distance to the box boundary for segment $i$ [S1].

**Steps:**

1.  **Build Histogram:** Integrate the transmittance of each trapezoid segment using the trapezoid integration algorithm [S1]. Let $\mathcal{I}_i$ be the integral value for segment $i$.
2.  **Compute CDF:** Calculate the Cumulative Distribution Function (CDF) by summing the integrals. The total unoccluded integral is $T_{\text{total}} = T_{\text{box}}(\tilde{\omega}_i, \tilde{\omega}_o, t_S)$ [S1]. The cumulative integral up to segment $k$ is $\sum_{k} \mathcal{I}_k$ [S1].
3.  **Select Segment:** Generate a uniform random number $r \in [0, 1]$. Select the trapezoid segment $i$ such that:
    $$ \sum_{k=1}^{i-1} \mathcal{I}_k < T_{\text{total}} \cdot r \le \sum_{k=1}^{i} \mathcal{I}_k $$
    [S1]
4.  **Transform Random Value:** Transform $r$ to an intermediate value $T_i(r) \in [0, 1]$ for the selected segment $i$ using Equation (10) from [S1]:
    $$ T_i(r) = \frac{\mathcal{I}_{\text{total}} r - \sum_{k=1}^{i} \mathcal{I}_{k-1}}{\mathcal{I}_i} $$
    where $\mathcal{I}_0 = 0$ [S1].
5.  **Determine Distance:** Use the inversion method to derive the sampling component. Transform $T_i(r)$ to distance using a scaled logarithm function [S1]:
    $$ t = -\frac{\log(1 - (1 - E_u) T_i(r))}{\sigma_t c} $$
    where $c$ is the slope and $E_u$ is the upper bound component of the trapezoid segment radiance integral [S1]. The final distance offset is adjusted by the starting distance $t_{C,S}$ [S1].
    *Note:* The position of corners ($\tilde{\mathbf{p}}_1, \tilde{\mathbf{p}}_2, \tilde{\mathbf{p}}_3$) and distance vectors ($\tilde{\mathbf{w}}_1, \tilde{\mathbf{w}}_2, \tilde{\mathbf{w}}_3$) are required to determine the distance between the camera ray and box boundary [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def whole_volume_distance_sampling(r, segments, sigma_t):
    """
    r: uniform random float in [0, 1]
    segments: list of dicts, each containing:
        - 'integral': float, the transmittance integral I_i
        - 'p': tuple, corner positions (p1, p2, p3)
        - 'w': tuple, distance vectors (w1, w2, w3)
        - 't_C_prev': float, cumulative distance offset from previous segment
        - 'd': float, distance to box boundary
        - 'c': float, slope
        - 'E_u': float, upper bound component
    sigma_t: float, extinction coefficient
    """
    # Step 1 & 2: Compute total integral and find segment
    total_integral = sum(seg['integral'] for seg in segments)
    
    current_cumulative = 0.0
    selected_segment = None
    
    for seg in segments:
        if current_cumulative + seg['integral'] >= total_integral * r:
            selected_segment = seg
            break
        current_cumulative += seg['integral']
        
    if selected_segment is None:
        # Fallback or error handling if r is exactly 1.0 and precision issues occur
        selected_segment = segments[-1]
        
    # Step 4: Transform random value
    # I_total is total_integral
    # Sum_{k=1}^{i-1} I_{k-1} is current_cumulative (since current_cumulative holds sum of previous I's)
    # Note: The formula uses sum_{k=1}^i I_{k-1}. In the loop, current_cumulative is sum_{k=1}^{i-1} I_k.
    # The source formula (10) is: T_i(r) = (I_total * r - sum_{k=1}^i I_{k-1}) / I_i
    # Here, sum_{k=1}^i I_{k-1} corresponds to the cumulative integral BEFORE the current segment's integral I_i is added?
    # Let's trace: For i=1, sum_{k=1}^1 I_{k-1} = I_0 = 0.
    # For i=2, sum_{k=1}^2 I_{k-1} = I_0 + I_1.
    # In the loop, when we are at segment i, current_cumulative is sum_{k=1}^{i-1} I_k.
    # The formula requires sum_{k=1}^i I_{k-1}.
    # I_{k-1} for k=1 is I_0. For k=2 is I_1.
    # So sum_{k=1}^i I_{k-1} = I_0 + I_1 + ... + I_{i-1}.
    # This is exactly current_cumulative.
    
    prev_integral_sum = current_cumulative
    I_i = selected_segment['integral']
    
    numerator = total_integral * r - prev_integral_sum
    T_i_r = numerator / I_i
    
    # Clamp to [0, 1] to handle floating point errors
    T_i_r = max(0.0, min(1.0, T_i_r))
    
    # Step 5: Determine distance
    c = selected_segment['c']
    E_u = selected_segment['E_u']
    
    # Pitfall: When slope approaches zero (c -> 0), sampling simplifies
    if abs(c) < 1e-6:
        # Simplified multiplication by distance between starting and end point
        # The source says "multiplication by the distance between starting and end point"
        # This likely refers to a linear interpolation or direct scaling if transmittance is constant/linear
        # However, standard distance sampling with c=0 implies constant extinction or specific geometry.
        # Given the formula -log(...)/(sigma_t * c), c=0 is a singularity.
        # The source states: "sampling simplifies to multiplication by the distance between starting and end point."
        # We assume this means a linear distribution or specific geometric distance.
        # Without explicit formula for c=0, we use a safe fallback or linear interpolation based on T_i_r
        # But strictly, if c->0, the exponential becomes linear? 
        # Let's stick to the source's description: "multiplication by the distance between starting and end point"
        # This is ambiguous. We will use a standard linear interpolation for the distance within the segment as a [G] guard.
        dist = selected_segment['t_C_prev'] + T_i_r * (selected_segment['d']) 
    else:
        # Scaled logarithm function
        # -log(1 - (1 - E_u) * r) / (sigma_t * c)
        # Note: Source uses r in the formula description but T_i(r) in the context of the segment.
        # The formula is: -log(1 - (1 - E_u) * T_i(r)) / (sigma_t * c)
        term = 1.0 - (1.0 - E_u) * T_i_r
        if term <= 0:
            term = 1e-10 # Guard against log(0)
        dist = -math.log(term) / (sigma_t * c)
        
    # Offset by starting distance
    final_distance = dist + selected_segment['t_C_prev']
    
    return final_distance
```

**Parameters:**
- $\sigma_t$: Extinction coefficient.
- $c$: Slope of the trapezoid segment.
- $E_u$: Upper bound component of the trapezoid segment radiance integral.
- Typical values are scene-dependent. The method is implemented following the optimized algorithm explained in Sec. 6 of [S1].

**Edge Cases and Pitfalls:**
- When the slope $c$ approaches zero ($c \to 0$), the logarithmic function is undefined. The sampling simplifies to multiplication by the distance between the starting and end point [S1].
- The algorithm depends on a logarithmic function which is not directly supported by the GPU; in corner cases, it falls back to linear segment sampling through branching [S1].
- Precision issues may arise when $r$ is very close to 1.0 or when integrals are very small.

## Subtopics

### [Whole Volume Importance Sampling](9008-whole-volume-importance-sampling.md)
An importance sampling scheme for Monte Carlo integrators that samples according to the geometry of the entire medium, incorporating both camera and light rays, rather than limiting sampling to transmittance along a single ray.

## Operating characteristics
**Time and Memory Complexity:**
The method requires building a histogram of trapezoid segments, which involves integrating transmittance for each segment. The selection of a segment is $O(N)$ where $N$ is the number of segments, or $O(\log N)$ with a precomputed CDF tree. The distance calculation is $O(1)$ per sample. Memory usage is proportional to the number of segments to store the histogram and CDF [S1].

**Measured Performance:**
On an RTX 8000 GPU at 1024 spp, MCWVDS on a room scene took $10.4 \pm 0.1$ ms, compared to $11.0 \pm 0.5$ ms for Ray Marching (RM) [S1].
On a GTX 980 Ti GPU at 256 spp, MCWVDS on a room scene took $9.2 \pm 0.1$ ms, compared to $10.3 \pm 0.2$ ms for RM [S1].
On a Pro 560X GPU at 128 spp, MCWVDS on a room scene took $40 \pm 4$ ms, compared to $37 \pm 4$ ms for RM [S1].

**Strengths and Limitations:**
- **Strengths:** Converges in a single sample in unoccluded sections. Leads to noticeable variance reduction in occluded areas compared to ray marching and distance sampling at low sample counts [S1].
- **Limitations:** Depends on a logarithmic function not directly supported by GPU hardware, requiring fallbacks [S1]. Equivalent to distance sampling only when evaluating a single segment where distance to boundary is constant ($d_{E,S} = d_{E,E}$) [S1].

**When to Use:**
Use when variance reduction in occluded areas is critical and low sample counts are used. It is particularly effective for single-scattering integrators [S1].

## Background
Whole Volume Distance Sampling, also known as Monte Carlo with Whole Volume Distance Sampling (MCWVDS), was introduced in the paper "Collimated Whole Volume Light Scattering in Homogeneous Finite Media" [S1]. It generalizes rendering light scattered in finite media by incorporating a sampling strategy that considers the geometry along the entire camera line segment, unlike existing distance sampling techniques which account only for transmittance along the current path segment [S1]. The method is implemented as part of a suite of stochastic integration algorithms including Ratio Estimator with Distance Sampling (REDS) and Ratio Estimator with Equidistant Sampling (REES) [S1].

## Sources
[S1] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
