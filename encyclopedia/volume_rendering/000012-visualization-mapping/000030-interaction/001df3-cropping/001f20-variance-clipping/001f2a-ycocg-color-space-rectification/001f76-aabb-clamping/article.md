# AABB Clamping
Also called: AABB Clamping History Rectification

## Overview
AABB Clamping is a history rectification technique used in Temporal Anti-Aliasing (TAA) to prevent ghosting artifacts by restricting reprojected historical color values to a valid range defined by the current frame. It solves the problem of temporal lag and ghosting during abrupt scene changes where a baseline Exponential Moving Average (EMA) fails to react quickly enough [S1, unofficial]. What sets it apart from its siblings and nearest alternatives is its use of a simple Axis-Aligned Bounding Box (AABB) approximation constructed from per-channel minimum and maximum values, whereas History Rectification encompasses more sophisticated variants like Variance Clipping that handle color distributions more accurately [S1, unofficial]. It builds on [YCoCg Color Space Rectification](1971-ycocg-color-space-rectification.md), which transforms colors into a luminance-chrominance space before applying this rectification to reduce color-related ghosting.

## How it works
**Inputs and Outputs:**
*   **Input:** The reprojected historical color $c_h$ (typically RGBA), and a set of $N$ current-frame neighborhood samples $c_i \in \mathbb{R}^k$ (where $k$ is the number of channels, e.g., 3 or 4).
*   **Output:** The rectified color $c_r$.

**Data Structures:**
*   **AABB Bounds:** Two vectors, $c_{min}$ and $c_{max}$, each of size $k$.
    *   $c_{min}[j] = \min_{i=1..N}(c_i[j])$ for each channel $j$.
    *   $c_{max}[j] = \max_{i=1..N}(c_i[j])$ for each channel $j$.

**Steps:**
1.  **Construct AABB:** Compute the component-wise minimum and maximum across all current-frame neighborhood samples to define the bounding box bounds $c_{min}$ and $c_{max}$ [S1, unofficial].
2.  **Clamp:** For each channel $j$ of the historical color $c_h$, clamp the value between the corresponding bounds $c_{min}[j]$ and $c_{max}[j]$ [S1, unofficial].

**Formula:**
The rectified color $c_r$ is obtained by clamping $c_h$ component-wise:
$$c_r = \min(\max(c_h, c_{min}), c_{max}) \quad (2.2)$$
where $\min$, $\max$, and arithmetic operations are performed component-wise [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aabb_clamp(historical_color, neighborhood_samples):
    """
    Rectifies historical color by clamping it to the AABB defined by neighborhood samples.
    
    Args:
        historical_color: List/Array of floats, shape (k,) representing ch.
        neighborhood_samples: List/Array of floats, shape (N, k) representing ci.
        
    Returns:
        List/Array of floats, shape (k,) representing cr.
    """
    # Step 1: Construct AABB bounds
    # c_min is the component-wise minimum of all neighborhood samples
    c_min = min(neighborhood_samples, key=lambda x: x) # Conceptual min over array
    
    # c_max is the component-wise maximum of all neighborhood samples
    c_max = max(neighborhood_samples, key=lambda x: x) # Conceptual max over array
    
    # Note: In implementation, this is done channel by channel:
    # c_min[j] = min(sample[j] for sample in neighborhood_samples)
    # c_max[j] = max(sample[j] for sample in neighborhood_samples)

    # Step 2: Clamp historical color to bounds
    rectified_color = []
    for j in range(len(historical_color)):
        val = historical_color[j]
        if val < c_min[j]:
            val = c_min[j]
        elif val > c_max[j]:
            val = c_max[j]
        rectified_color.append(val)
        
    return rectified_color
```

**Edge Cases and Pitfalls:**
*   **Corner Accumulation:** Because each channel is individually clamped, values tend to collect in the corners of the AABB [S1, unofficial].
*   **Color Artifacts:** This corner accumulation can lead to strange color-related artifacts in scenes with significant color information, although it is generally not an issue for grayscale scenes [S1, unofficial].
*   **Accuracy:** This method is a fast approximation of the tighter convex hull approach. It is less accurate but much cheaper to build and intersect against [S1, unofficial].

## Subtopics

### AABB Clipping
Projects the historical color onto the AABB towards the current frame sample or the center of the box, rather than clamping channels individually. This better preserves temporal color gradients and reduces corner artifacts compared to clamping [S1, unofficial].

### Variance Clipping
Uses the statistical distribution (variance) of values in the neighborhood to create the bounds of the AABB used for clipping, offering a more robust handling of color information than simple min/max [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot k)$ to compute min/max over $N$ samples with $k$ channels, followed by $O(k)$ for clamping. It is computationally efficient [S1, unofficial].
*   **Memory Complexity:** $O(k)$ to store the min and max bounds.
*   **Strengths:** Fast approximation of history rectification; easy to implement and intersect against; ensures history remains within valid color extent [S1, unofficial].
*   **Limitations:** Less accurate than convex hull methods; prone to color artifacts due to channel-wise clamping causing values to collect in AABB corners [S1, unofficial].
*   **When to use:** When performance is critical and the scene is predominantly grayscale or has limited color variation.
*   **When not to use:** In scenes with complex color information where color artifacts are noticeable, or when higher fidelity rectification (like AABB Clipping or Variance Clipping) is required [S1, unofficial].

## Background
AABB Clamping is a history rectification method implemented and evaluated in the context of temporal anti-aliasing for real-time froxel-based volumetric rendering [S1, unofficial]. It serves as a fast approximation to the computationally expensive convex hull projection, which is too costly for real-time rendering constraints [S1, unofficial]. The method is part of a class of techniques that correct historical color values before blending by restricting them to a region defined by current-frame neighborhood samples [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
