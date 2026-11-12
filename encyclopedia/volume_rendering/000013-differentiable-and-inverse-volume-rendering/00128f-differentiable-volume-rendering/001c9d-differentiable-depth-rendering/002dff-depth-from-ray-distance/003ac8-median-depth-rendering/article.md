# Median Depth Rendering

Also called: 50% transmittance depth, Depth Renderer (Median), Median Depth Visualization, Median termination depth

## Overview
Median Depth Rendering computes the geometric depth of a volume by identifying the sample along a ray where the accumulated transmittance drops to 0.5. It solves the bias introduced by residual density in free space that affects Expected Depth Rendering, providing a sharper and less biased geometric representation for ambiguous or sparse scenes [S2]. Unlike Expected Depth Rendering, which integrates the entire ray's contribution to find a weighted average, Median Depth Rendering selects a single threshold-crossing point, making it robust to noise in empty regions [S1, unofficial][S2].

## How it works
**Inputs:**
- Ray samples: A set of $N$ points along a ray, defined by their depths $z_i$ and densities $\sigma_i$ for $i = 0, \dots, N-1$.
- Step sizes: $\Delta z_i$ (distance between sample $i$ and $i+1$).

**Outputs:**
- Depth value $D$ for the pixel corresponding to the ray.

**Data Structures:**
- `z_vals`: Array of shape $(N,)$ containing depths $z_i$.
- `sigma`: Array of shape $(N,)$ containing densities $\sigma_i$.
- `T`: Array of shape $(N,)$ storing accumulated transmittance.

**Parameters:**
- `depth_threshold`: The transmittance threshold for median definition. Default is $0.5$ [S1, unofficial][S2].

**Steps:**
1. **Compute Transmittance:** Calculate the accumulated transmittance $T_i$ for each sample. $T_i$ represents the probability that a ray passes through all samples up to $i$ without interacting.
   $$ T_0 = 1.0 $$
   $$ T_{i+1} = T_i \cdot \exp(-\sigma_i \Delta z_i) \quad \text{for } i = 0, \dots, N-2 $$
   [G] Note: $\Delta z_i = z_{i+1} - z_i$. The last sample's transmittance is not strictly needed for the threshold check if we stop at $N-1$, but typically $T$ is computed for all $N$ samples where $T_N$ would be the transmittance after the last sample.

2. **Find Median Index:** Identify the first sample index $i^*$ such that the accumulated transmittance $T_{i^*}$ is greater than or equal to the threshold (typically 0.5).
   $$ i^* = \min \{ i \mid T_i \ge 0.5 \} $$
   [S2] If no sample satisfies $T_i \ge 0.5$ (i.e., the ray is fully opaque before the threshold), the behavior is undefined in the sources; typically, the last sample or a fallback is used [G].

3. **Assign Depth:** The rendered depth is the depth value of the identified sample.
   $$ D = z_{i^*} $$
   [S1, unofficial][S2]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def median_depth_rendering(z_vals: list[float], sigma: list[float], threshold: float = 0.5) -> float:
    """
    Compute median depth rendering for a single ray.
    
    Args:
        z_vals: List of depths along the ray, sorted ascending.
        sigma: List of densities at each depth sample.
        threshold: Transmittance threshold (default 0.5).
        
    Returns:
        The depth value at the median termination point.
    """
    n = len(z_vals)
    if n == 0:
        return 0.0  # [G] Fallback for empty ray
        
    # Step 1: Compute accumulated transmittance T
    # T[i] is the transmittance arriving at sample i
    T = [0.0] * n
    T[0] = 1.0
    
    for i in range(n - 1):
        delta_z = z_vals[i+1] - z_vals[i]
        # T[i+1] = T[i] * exp(-sigma[i] * delta_z)
        T[i+1] = T[i] * math.exp(-sigma[i] * delta_z)
        
    # Step 2: Find the first index where T[i] >= threshold
    median_index = -1
    for i in range(n):
        if T[i] >= threshold:
            median_index = i
            break
            
    # Edge case: If threshold is never reached (ray is fully opaque before threshold),
    # sources do not specify. [G] Return the depth of the last sample or handle as needed.
    if median_index == -1:
        median_index = n - 1
        
    # Step 3: Return depth at median_index
    return z_vals[median_index]
```

**Edge Cases and Pitfalls:**
- **Fully Opaque Ray:** If the accumulated transmittance drops below 0.5 immediately or never reaches 0.5 because the ray is blocked early, the definition $T_i \ge 0.5$ might fail to find an index if the ray is *more* opaque than the threshold. However, $T_0=1.0$ always satisfies $T_0 \ge 0.5$, so the first sample is always a candidate. The "first sample" logic ensures that if the object starts at the first sample, that depth is returned.
- **Free Space Bias:** In regions with no geometry (density $\approx 0$), transmittance remains near 1.0. Expected depth would pull the value towards the far plane. Median depth ignores this residual density, staying anchored to the actual surface where opacity accumulates [S2].
- **Discontinuity:** The depth output is discontinuous with respect to small changes in density, as it jumps between sample indices. This can cause artifacts in differentiable rendering if not handled carefully (e.g., via soft approximations) [G].

## Subtopics

### [Expected Depth Rendering](8816-expected-depth-rendering.md)
Expected Depth Rendering estimates scene depth by computing the weighted average of depth samples along a ray, normalized by the accumulated weight. This method provides a stable geometric signal for pseudo-normal supervision and ray-tracing initialization, offering superior reconstruction quality and downstream stability compared to inverse-depth variants. It differs from Median Depth Rendering, which identifies the sample index where cumulative opacity crosses a threshold, by integrating the entire ray's contribution rather than selecting a single median point.

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. Finding the threshold index is a linear scan.
- **Memory Complexity:** $O(N)$ to store transmittance values if computed explicitly, or $O(1)$ if computed on the fly with early termination.
- **Strengths:** Robust to residual density in empty space; provides sharper depth boundaries than expected depth [S2].
- **Limitations:** Discontinuous output can hinder gradient-based optimization in differentiable settings without smoothing [G]. Sensitive to the choice of threshold (though 0.5 is standard) [S1, unofficial].
- **When to use:** When geometric accuracy is prioritized over smoothness, or when residual density in free space biases expected depth significantly [S2].
- **When not to use:** When differentiable gradients are required without additional smoothing techniques, or when a continuous depth field is needed for downstream tasks like normal estimation [S1, unofficial].

## Background
Median Depth Rendering was introduced in the context of Deformable Neural Radiance Fields (NeRFies) to address the bias in depth visualization caused by residual density in free space [S2]. While standard NeRF uses expected termination distance for depth, NeRFies found that median depth provides a less biased geometric representation, particularly for deformable scenes [S2]. It is used in NeRFies for depth visualization and potentially for initialization or supervision in deformable reconstruction tasks [S1, unofficial][S2]. The method is also referenced in other implementations like Nerfies and GanERF for depth rendering [S1, S3, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] Nerfies: Deformable Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00581, fetched 2026-10-26
[S3] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
