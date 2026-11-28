# Variance Clipping

Also called: Local Variance

## Overview
Variance clipping is a history rectification technique used in temporal anti-aliasing for volumetric rendering that constructs a tight bounding box for valid history samples based on the statistical distribution (mean and variance) of colors in the current frame's neighborhood [S1, unofficial]. It solves the problem of ghosting artifacts by discarding history samples that fall outside the statistically probable range of current-frame colors, adapting more quickly to sudden illumination changes than simple extent-based clipping [S1, unofficial]. It extends the concept of clipping against the current frame color extent by relying on the statistical distribution of values in the neighborhood to produce a smaller clipping region [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of $N$ color samples $\{c_i\}_{i=1}^N$ from the current frame's neighborhood [S1, unofficial].
- Input: A user-defined parameter $\gamma$ (gamma) controlling the clipping strength [S1, unofficial].
- Output: A clipping interval $[c_{min}, c_{max}]$ for each color component [S1, unofficial].

**Data Structures:**
- $c_i$: A vector of color values (e.g., RGB) for the $i$-th sample [G].
- $N$: The number of samples in the neighborhood [G].
- $\mu$: The sample mean vector [G].
- $\sigma$: The sample standard deviation vector [G].

**Steps:**
1. Compute the first sample moment $m_1$ by summing all color vectors component-wise:
   $$m_1 = \sum_{i=1}^N c_i \quad (2.3)$$ [S1, unofficial]
2. Compute the second sample moment $m_2$ by summing the squared color vectors component-wise:
   $$m_2 = \sum_{i=1}^N c_i^2 \quad (2.4)$$ [S1, unofficial]
3. Calculate the sample mean $\mu$ by dividing $m_1$ by $N$ component-wise:
   $$\mu = \frac{1}{N}m_1 \quad (2.5)$$ [S1, unofficial]
4. Calculate the sample standard deviation $\sigma$ component-wise:
   $$\sigma = \sqrt{\left| \frac{1}{N} m_2 - \mu^2 \right|} \quad (2.6)$$ [S1, unofficial]
   The absolute value is taken to avoid negative values caused by floating-point inaccuracies [S1, unofficial].
5. Construct the min-max clipping interval using the user-defined parameter $\gamma$:
   $$c_{min} = \mu - \gamma \sigma \quad (2.7)$$
   $$c_{max} = \mu + \gamma \sigma \quad (2.8)$$ [S1, unofficial]

**Parameters:**
- $\gamma$ (gamma): A user-defined parameter that controls the size of the clipping box and therefore the strength of clipping [S1, unofficial].

**Edge Cases and Pitfalls:**
- Floating-point inaccuracies can lead to negative values under the square root in the standard deviation calculation; the absolute value operation mitigates this [S1, unofficial].
- Variance clipping exhibits the highest Temporal Instability (TI) relative to the baseline EMA among tested techniques, making it prone to flickering in static scenes [S1, unofficial].
- Performing history rectification directly in RGB color space can produce color-related artifacts because RGB channels are highly correlated, leading to poorly aligned bounding boxes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def variance_clipping(samples, gamma):
    """
    Compute variance-based clipping bounds for a set of color samples.
    
    Args:
        samples: List of color vectors (e.g., [r, g, b]), shape (N, 3).
        gamma: User-defined parameter controlling clipping strength.
        
    Returns:
        c_min: Lower bound vector.
        c_max: Upper bound vector.
    """
    N = len(samples)
    if N == 0:
        raise ValueError("No samples provided")
        
    # Step 1: First sample moment m1 = sum(c_i)
    m1 = [0.0, 0.0, 0.0]
    for c in samples:
        for k in range(3):
            m1[k] += c[k]
            
    # Step 2: Second sample moment m2 = sum(c_i^2)
    m2 = [0.0, 0.0, 0.0]
    for c in samples:
        for k in range(3):
            m2[k] += c[k] * c[k]
            
    # Step 3: Sample mean mu = (1/N) * m1
    mu = [m1[k] / N for k in range(3)]
    
    # Step 4: Standard deviation sigma = sqrt(|(1/N) * m2 - mu^2|)
    sigma = [0.0, 0.0, 0.0]
    for k in range(3):
        variance_term = (m2[k] / N) - (mu[k] * mu[k])
        sigma[k] = (abs(variance_term) ** 0.5)
        
    # Step 5: Clipping interval
    c_min = [mu[k] - gamma * sigma[k] for k in range(3)]
    c_max = [mu[k] + gamma * sigma[k] for k in range(3)]
    
    return c_min, c_max
```

## Subtopics

### [YCoCg Color Space Rectification](1971-ycocg-color-space-rectification.md)
A variant that transforms reprojected historical color data into a luminance-chrominance color space (YCoCg or YCrCb) before applying bounding-box clipping. It sets itself apart by decoupling luminance and chrominance to reduce chroma-related ghosting, which is a known issue when applying variance clipping directly in RGB space. Choose this when color bleeding artifacts are prominent due to high channel correlation [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of samples in the neighborhood, as it requires a single pass to compute sums and squared sums [G].
- **Memory Complexity:** $O(1)$ auxiliary space if computed incrementally, or $O(N)$ if storing all samples [G].
- **Measured Performance:** In a step response scenario (sudden illumination change), variance clipping produced the lowest RMSE (0.002977) compared to AABB Clamp/Clip (0.003136) and Baseline EMA (0.009724), indicating it is the best at reconstructing the signal when light is quickly introduced [S1, unofficial]. However, it exhibits the highest Temporal Instability (TI), with a 190.23% increase in TI P95 relative to the baseline [S1, unofficial].
- **Strengths:** Tighter bounds allow the algorithm to adapt to changes much more quickly and precisely than extent-based methods [S1, unofficial]. It operates entirely on current-frame data, making it easier to implement than methods requiring historical data analysis [S1, unofficial].
- **Limitations:** High temporal instability can cause flickering. Direct application in RGB space leads to color artifacts due to channel correlation [S1, unofficial].
- **When to use:** When rapid adaptation to illumination changes is prioritized over stability, or when used in conjunction with color space transformations like YCoCg to mitigate color artifacts [G].
- **When not to use:** In static scenes where temporal stability is critical, or if RGB color space artifacts are unacceptable without rectification [G].

## Background
Variance clipping was introduced as a history rectification technique for temporal anti-aliasing in volumetric rendering [S1, unofficial]. It extends the concept of clipping against the current frame color extent by relying on the statistical distribution of values in the neighborhood [S1, unofficial]. The method was evaluated in the context of real-time froxel-based volumetric rendering, where it was found to be effective at reducing ghosting artifacts during dynamic scenes [S1, unofficial]. The term "Local Variance" is also used in the context of volume visualization to describe the variance in the neighborhood of a voxel, which is used to distinguish tissues based on spatial coherence, such as removing CT streaking artifacts by selecting high-variance voxels [S2, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time ren dering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S2] Multi-Dimensional Transfer Function Design Using Sorted Histograms, https://doi.org/10.2312/vg/vg06/001-008, fetched 2026-10-26 (unofficial)
