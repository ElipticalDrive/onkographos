# Whole Volume Distance Sampling

Also called: Histogram-based Transmittance Sampling, Importance Sampling for Volume Rendering, Monte Carlo with Whole Volume Distance Sampling

## Overview
Whole Volume Distance Sampling (WVDS) is a Monte Carlo technique for estimating the Volume Rendering Equation (VRE) that importance samples the integration domain according to the geometry-dependent transmittance of the entire participating medium, rather than limiting sampling to the transmittance along a single ray [S1]. It solves the high-variance noise inherent in standard distance sampling by incorporating both camera and light ray geometries into the probability density function (PDF), resulting in superior convergence for finite media [S1]. It differs from classic techniques by accounting for the global geometry of the medium, not just local transmittance [S1].

## How it works
**Inputs:**
*   Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   Light ray geometry defining the trapezoid segments (corners $\tilde{\mathbf{p}}_i$ and distance vectors $\tilde{\mathbf{w}}_i$).
*   Medium extinction coefficient $\sigma_t$.
*   Speed of light $c$ (used in the sampling function derivation).
*   Uniform random number $r \in [0, 1]$.

**Outputs:**
*   Distance $t$ along the camera ray from the origin to the scattering event.

**Data Structures:**
*   **Histogram of Integrals:** An array $\mathcal{I}$ where $\mathcal{I}_k$ is the integral of transmittance for the $k$-th trapezoid segment.
*   **Cumulative Distribution Function (CDF):** Derived from $\mathcal{I}$, stored as cumulative sums normalized by the total unoccluded integral $T_{\text{total}}$.
*   **Segment Geometry:** For each segment $i$, the corner positions $\tilde{\mathbf{p}}_i$ (or $\tilde{\mathbf{p}}_L, \tilde{\mathbf{p}}_1, \tilde{\mathbf{p}}_2$ depending on indexing) and distance vectors $\tilde{\mathbf{w}}_i$ (or $\tilde{\mathbf{w}}_1, \tilde{\mathbf{w}}_2, \tilde{\mathbf{w}}_3$).
*   **Boundary Distances:** $d_{E,S}$ (distance to entry boundary) and $d_{E,E}$ (distance to exit boundary) for the segment.
*   **Camera Ray Bounds:** $t_{C,S}$ (start distance) and $t_{C,E}$ (end distance) for the segment.

**Steps:**

1.  **Build Histogram:** Integrate the transmittance of each trapezoid segment using the trapezoid integration algorithm to obtain $\mathcal{I}_k$ for all segments $k$ [S1].
2.  **Compute CDF:** Calculate the total unoccluded integral $T_{\text{total}} = T_{\text{box}}(\tilde{\omega}_i, \tilde{\omega}_o, t_S)$ [S1]. Compute the cumulative integral for segment $i$ as $\sum_{k=1}^i \mathcal{I}_{k-1}$ with $\mathcal{I}_0 = 0$ [S1].
3.  **Select Segment:** Compare the random value $r$ against the normalized cumulative integrals to select the appropriate trapezoid segment $i$ [S1].
4.  **Transform Random Value:** Map $r$ to an intermediate value $T_i(r) \in [0, 1]$ specific to segment $i$ using the formula:
    $$ T_i(r) = \frac{\mathcal{I}_{\text{total}} r - \sum_{k=1}^i \mathcal{I}_{k-1}}{\mathcal{I}_i} $$
    where $\mathcal{I}_{\text{total}}$ is the sum of all segment integrals [S1].
5.  **Compute Distance:** Use the inversion method to derive the distance offset from $T_i(r)$. The sampling function $S_{\text{hist}}(r)$ selects the segment-specific function $S_s$:
    $$ S_{\text{hist}}(r) = S_s(T_i(r), \tilde{\mathbf{p}}_i, \tilde{\mathbf{w}}_i, t_{C,i-1}, d_i) $$
    The core calculation involves a scaled logarithm function:
    $$ \text{offset} = \frac{-\log(1 - (1 - E_u) T_i(r))}{\sigma_t c} $$
    where $E_u$ is the upper bound component of the trapezoid segment radiance integral [S1].
6.  **Handle Degeneracy:** If the slope $c \to 0$, the sampling simplifies to multiplication by the distance between the starting and end point [S1].
7.  **Final Distance:** Add the starting distance $t_{C,S}$ to the offset to get the final distance from the origin along the camera ray [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def whole_volume_distance_sampling(r, sigma_t, c, segments, total_integral):
    """
    r: Uniform random float in [0, 1]
    sigma_t: Extinction coefficient
    c: Slope parameter
    segments: List of segment objects, each containing:
        - integral: I_i (integral of transmittance for this segment)
        - cum_integral: Sum of I_k for k < i
        - p: Corner positions (tilde_p)
        - w: Distance vectors (tilde_w)
        - t_C_S: Start distance along camera ray
        - d_E_S: Distance to entry boundary
        - d_E_E: Distance to exit boundary
        - E_u: Upper bound component of radiance integral
    total_integral: T_total (sum of all segment integrals)
    
    Returns: Distance t along the camera ray.
    """
    # Step 1 & 2: Select segment and transform random value
    # Find segment i such that cum_integral_i / total_integral < r <= cum_integral_{i+1} / total_integral
    # In practice, iterate or use binary search on the CDF
    selected_segment = None
    for seg in segments:
        # Check if r falls within this segment's range
        # Range start: seg.cum_integral / total_integral
        # Range end: (seg.cum_integral + seg.integral) / total_integral
        range_start = seg.cum_integral / total_integral
        range_end = (seg.cum_integral + seg.integral) / total_integral
        
        if range_start <= r < range_end:
            selected_segment = seg
            break
    
    if selected_segment is None:
        # Fallback or error handling if r is exactly 1.0 or out of bounds
        selected_segment = segments[-1]
        # Adjust r to be within the last segment's range if necessary
        # This is a guard [G]
        r = range_end - 1e-9 

    # Step 4: Transform random value T_i(r)
    # T_i(r) = (I_total * r - sum(I_{k-1})) / I_i
    # Note: sum(I_{k-1}) is seg.cum_integral
    # I_i is seg.integral
    # I_total is total_integral
    
    numerator = total_integral * r - selected_segment.cum_integral
    denominator = selected_segment.integral
    
    # Guard against division by zero if integral is 0 [G]
    if denominator == 0:
        return selected_segment.t_C_S # Return start distance if no transmittance
        
    T_i_r = numerator / denominator
    
    # Clamp T_i_r to [0, 1] to handle floating point errors
    T_i_r = max(0.0, min(1.0, T_i_r))
    
    # Step 5: Compute distance using inversion method
    # Check for degenerate case c -> 0
    if abs(c) < 1e-9:
        # Simplifies to multiplication by distance between start and end point [S1]
        # The distance is proportional to the segment length in parameter space
        # Assuming linear mapping for degenerate case
        offset = T_i_r * (selected_segment.d_E_E - selected_segment.d_E_S)
    else:
        # Scaled logarithm function
        # offset = -log(1 - (1 - E_u) * T_i(r)) / (sigma_t * c)
        # Note: The source formula uses E_u. E_u is part of the segment geometry.
        E_u = selected_segment.E_u
        
        # Guard against log(0) or log(negative) [G]
        arg = 1.0 - (1.0 - E_u) * T_i_r
        if arg <= 0:
            arg = 1e-9
            
        offset = -math.log(arg) / (sigma_t * c)
        
    # Step 7: Final distance
    t = selected_segment.t_C_S + offset
    
    return t
```

## Subtopics

### [Whole Volume Importance Sampling](9008-whole-volume-importance-sampling.md)
Whole Volume Importance Sampling is a Monte Carlo strategy for volume rendering that samples the integration domain according to the geometry-dependent transmittance of the entire participating medium, rather than limiting sampling to the transmittance along a single ray. It solves the high-variance noise inherent in standard distance sampling by incorporating both camera and light ray geometries into the probability density function (PDF), resulting in superior convergence for finite media. It differs from classic techniques by accounting for the global geometry of the medium, not just local transmittance.

## Operating characteristics
*   **Time Complexity:** The algorithm requires building a histogram and CDF, which is $O(N)$ where $N$ is the number of trapezoid segments. Sampling is $O(\log N)$ with binary search or $O(N)$ with linear scan.
*   **Memory Complexity:** $O(N)$ to store the histogram and segment geometry.
*   **Measured Performance:**
    *   On an RTX 8000 GPU at 1024 spp, MCWVDS on a room scene took 10.4 ± 0.1 ms, compared to 11.0 ± 0.5 ms for Ray Marching (RM) [S1].
    *   On a GTX 980 Ti GPU at 256 spp, MCWVDS on a room scene took 9.2 ± 0.1 ms, compared to 10.3 ± 0.2 ms for RM [S1].
    *   On a Pro 560X GPU at 128 spp, MCWVDS on a room scene took 40 ± 4 ms, compared to 37 ± 4 ms for RM [S1].
*   **Strengths:** Converges in a single sample in unoccluded sections and leads to noticeable variance reduction in the rest of the scene, outperforming ray marching and distance sampling at low sample counts [S1].
*   **Limitations:** Depends on a logarithmic function which is not directly supported by the GPU, requiring fallback to linear segment sampling in corner cases, which may impact performance [S1].
*   **When to Use:** When variance reduction is critical in finite media with complex occlusion, and low sample counts are used.
*   **When Not to Use:** When hardware does not support logarithmic functions efficiently, or when the medium is simple and unoccluded, where standard distance sampling or ray marching may suffice.

## Background
Whole Volume Distance Sampling was introduced in the paper "Collimated Whole Volume Light Scattering in Homogeneous Finite Media" [S1]. It is a generalization of distance sampling that incorporates the geometry of the entire participating medium into the sampling strategy. The method is implemented as part of the Monte Carlo with Whole Volume Distance Sampling (MCWVDS) integrator [S1].

## Sources
[S1] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
