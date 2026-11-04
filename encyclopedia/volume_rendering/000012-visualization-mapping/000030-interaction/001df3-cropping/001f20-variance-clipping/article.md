# Variance Clipping

Also called: Variance-based history rectification, Statistical variance clipping

## Overview
Variance clipping is a history rectification technique used in Temporal Anti-Aliasing (TAA) for volumetric rendering that constructs a tight bounding box for valid color values based on the statistical mean and variance of a neighborhood of samples [S1, unofficial]. It solves the problem of ghosting artifacts during rapid illumination changes by adapting the clipping region to the local signal distribution, allowing it to reconstruct signals faster than fixed-bound methods [S1, unofficial]. It sets itself apart from [AABB Clamping](2019-aabb-clamping.md) and [AABB Clipping](1444-clipping.md) by using a statistically derived, typically smaller, clipping region that reacts more precisely to changes, though at the cost of higher temporal instability [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ color samples $\{c_i\}$ from the current frame's neighborhood (e.g., a pixel and its spatial/temporal neighbors), where each $c_i$ is a vector (typically RGB).
*   **Output:** A clipping interval $[c_{min}, c_{max}]$ for each color component, used to clamp or clip the history value.

**Data Structures:**
*   $N$: Integer count of samples in the neighborhood.
*   $c_i$: Vector of color values for sample $i$.
*   $\mu$: Vector of sample means.
*   $\sigma$: Vector of standard deviations.
*   $\gamma$: Scalar user-defined parameter.

**Steps:**
1.  Compute the first sample moment $m_1$ by summing all color samples component-wise:
    $$m_1 = \sum_{i=1}^N c_i \quad (2.3)$$
2.  Compute the second sample moment $m_2$ by summing the squared color samples component-wise:
    $$m_2 = \sum_{i=1}^N c_i^2 \quad (2.4)$$
3.  Calculate the sample mean $\mu$ by dividing $m_1$ by $N$ component-wise:
    $$\mu = \frac{1}{N}m_1 \quad (2.5)$$
4.  Calculate the standard deviation $\sigma$ component-wise. First compute the variance estimate $\frac{1}{N} m_2 - \mu^2$, take the absolute value to handle floating-point inaccuracies that might result in negative values, and then take the square root:
    $$\sigma = \sqrt{\left| \frac{1}{N} m_2 - \mu^2 \right|} \quad (2.6)$$
5.  Construct the clipping bounds using the user-defined gamma parameter $\gamma$:
    $$c_{min} = \mu - \gamma \sigma \quad (2.7)$$
    $$c_{max} = \mu + \gamma \sigma \quad (2.8)$$
6.  Clip the history value $c_{hist}$ to the interval $[c_{min}, c_{max}]$ for each component.

**Parameters:**
*   $\gamma$ (Gamma): User-defined parameter controlling the size of the clipping box and the strength of clipping. Tighter bounds (smaller $\gamma$) allow faster adaptation to changes but increase temporal instability [S1, unofficial]. Typical values are not specified in the sources.

**Pitfalls:**
*   **Temporal Instability:** Variance clipping exhibits the highest Temporal Instability (TI) relative to the baseline EMA among tested techniques [S1, unofficial]. This is a direct trade-off for its ability to quickly reject incorrect history during rapid changes.
*   **Floating-Point Inaccuracy:** The variance calculation $\frac{1}{N} m_2 - \mu^2$ can yield negative values due to floating-point precision limits. The absolute value operation in Eq. (2.6) is required to prevent NaNs or errors [S1, unofficial].
*   **Color Correlation:** Performing this rectification directly in RGB space can produce color-related artifacts because RGB channels are highly correlated, leading to poorly aligned bounding boxes [S1, unofficial]. This is often mitigated by transforming to a luminance-chrominance space like YCoCg first [S1, unofficial].

## Subtopics

### [YCoCg Color Space Rectification](1971-ycocg-color-space-rectification.md)
A variant that transforms colors into a luminance-chrominance space (YCoCg or YCrCb) before applying variance clipping. It decouples luminance and chrominance to orient the rectification bounds along the luma direction, reducing color-related ghosting and temporal color bleeding. Choose this when RGB-based variance clipping causes color bleeding or poor alignment with the color distribution.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples in the neighborhood. The operations are simple component-wise arithmetic.
*   **Memory Complexity:** $O(1)$ auxiliary memory per pixel (storing $m_1, m_2, \mu, \sigma$).
*   **Measured Performance:** In a step response scenario (light turning on), variance clipping produced the lowest RMSE (0.002977) among tested techniques, indicating it is the best at reconstructing the signal during quick illumination introduction [S1, unofficial]. However, it exhibited the highest Temporal Instability (TI P = 0.004217, a 190.23% increase over baseline) [S1, unofficial].
*   **Strengths:** Tighter bounds allow the algorithm to adapt to changes much more quickly and precisely than fixed AABB methods. It operates entirely on current-frame data, making it easier to implement than methods requiring historical bounds [S1, unofficial].
*   **Limitations:** High temporal instability can cause flickering or jitter in static scenes or scenes with subtle changes. Sensitive to the choice of $\gamma$.
*   **When to use:** When rapid adaptation to illumination changes is prioritized over stability, or in scenes with high dynamic range lighting changes.
*   **When not to use:** In static scenes or scenes with subtle temporal variations where stability is more important than rapid convergence.

## Background
Variance clipping was introduced as an extension of clipping against the current frame color extent, relying instead on the statistical distribution of values in the neighborhood to produce a smaller clipping region [S1, unofficial]. It is identified as the best technique at reconstructing signals during quick illumination introduction [S1, unofficial]. The method is discussed in the context of Temporal Anti-Aliasing for real-time froxel-based volumetric rendering [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
