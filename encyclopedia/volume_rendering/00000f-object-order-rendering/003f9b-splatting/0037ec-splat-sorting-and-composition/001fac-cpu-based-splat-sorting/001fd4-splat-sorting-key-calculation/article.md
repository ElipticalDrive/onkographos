# Splat Sorting Key Calculation

Also called: Importance Sorting, Splat ordering metric, Splat sort key

## Overview
This method calculates a scalar key for each 3D Gaussian splat to determine the rendering order, typically for front-to-back alpha blending [S1, unofficial][S2, unofficial]. It combines the splat's projected size and opacity to prioritize "important" splats (large and opaque) to be drawn first, ensuring correct accumulation of color and alpha [S2, unofficial].

## How it works
**Inputs:**
- A list of $N$ splats [S2, unofficial].
- For each splat $i$:
  - Scale components: $s_{0,i}, s_{1,i}, s_{2,i}$ (log-space scales) [S1, unofficial][S2, unofficial].
  - Opacity: $\alpha_i$ (logit-space opacity) [S1, unofficial][S2, unofficial].
  - [G] Position and rotation are not used in the key calculation itself but are required for subsequent rendering steps.

**Outputs:**
- An array of indices `sorted_indices` of length $N$, such that splats are ordered from highest key to lowest key [G].

**Data Structures:**
- `sizeList`: Float32Array of length $N$. Stores the calculated key for each splat [S2, unofficial].
- `sizeIndex`: Uint32Array of length $N$. Stores the original index of each splat [S2, unofficial].

**Formula:**
The key $K_i$ for splat $i$ is defined as [S1, unofficial]:

$$
K_i = \frac{\exp(s_{0,i} + s_{1,i} + s_{2,i})}{1 + \exp(-\alpha_i)}
$$

Where:
- $\exp(s_{0,i} + s_{1,i} + s_{2,i})$ represents the volume or area metric derived from the log-scales [S1, unofficial].
- $1 + \exp(-\alpha_i)$ is the sigmoid denominator, converting logit opacity to probability space [S1, unofficial].

**Steps:**
1. Initialize `sizeList` as a Float32Array of length $N$ and `sizeIndex` as a Uint32Array of length $N$ [S2, unofficial].
2. For each splat index $i$ from $0$ to $N-1$:
   a. Read $s_{0,i}, s_{1,i}, s_{2,i}$ and $\alpha_i$ [S2, unofficial].
   b. Calculate the numerator: $V_i = \exp(s_{0,i} + s_{1,i} + s_{2,i})$ [S1, unofficial].
   c. Calculate the denominator: $O_i = 1 + \exp(-\alpha_i)$ [S1, unofficial].
   d. Compute key: $K_i = V_i / O_i$ [S1, unofficial].
   e. Set `sizeList[i] = K_i` [S2, unofficial].
   f. Set `sizeIndex[i] = i` [S2, unofficial].
3. Sort `sizeIndex` in descending order based on the values in `sizeList` [S2, unofficial]. Specifically, if $K_a > K_b$, then index $a$ should precede index $b$ in the sorted array [G].
4. Return `sizeIndex` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_splat_sort_key(splats):
    """
    splats: list of dicts or objects with keys 'scale_0', 'scale_1', 'scale_2', 'opacity'
    Returns: list of indices sorted by importance (descending)
    """
    import math

    n = len(splats)
    if n == 0:
        return []

    # Arrays to store keys and original indices
    size_list = [0.0] * n
    size_index = list(range(n))

    for i in range(n):
        # Read raw vertex properties
        s0 = splats[i].get('scale_0')
        s1 = splats[i].get('scale_1')
        s2 = splats[i].get('scale_2')
        alpha = splats[i].get('opacity')

        # If scale_0 does not exist, key remains 0 (or default)
        if s0 is None:
            size_list[i] = 0.0
            size_index[i] = i
            continue

        # Calculate volume metric from log-scales
        volume = math.exp(s0 + s1 + s2)

        # Calculate opacity from logit
        opacity = 1.0 / (1.0 + math.exp(-alpha))

        # Key is volume * opacity
        key = volume * opacity

        size_list[i] = key
        size_index[i] = i

    # Sort indices based on size_list in descending order
    size_index.sort(key=lambda idx: size_list[idx], reverse=True)

    return size_index
```

**Edge Cases and Pitfalls:**
- **Missing Scales:** If scale components are missing, the key defaults to 0 or a minimal value [S2, unofficial].
- **Precision:** Using `exp()` on large scale values can cause overflow [G]. In practice, scales are usually small, but clamping or using log-sum-exp tricks may be necessary for extreme cases [G].
- **Degenerate Input:** If all splats have identical keys, the sort order is arbitrary [G]. This is acceptable as long as the rendering is consistent [G].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ due to the sorting step [G]. The key calculation is $O(N)$ [G].
- **Memory Complexity:** $O(N)$ for storing `sizeList` and `sizeIndex` [G].
- **Performance:** Sorting on the CPU (e.g., in a WebWorker) offloads the GPU, allowing it to focus on rendering [S1, unofficial].
- **Strengths:** Allows progressive loading by sorting only loaded splats [S1, unofficial].
- **Limitations:** Does not account for view-dependent effects like spherical harmonics in the key calculation [S1, unofficial]. The key is static per splat and does not update with camera movement, so it is most effective when the camera is relatively stable or when splats are sorted per frame if performance allows [G].

## Background
This method is used in 3D Gaussian Splatting renderers, particularly in web-based implementations where CPU-based sorting is preferred for simplicity and compatibility [S1, unofficial][S2, unofficial]. The approach prioritizes visual importance over geometric depth, which is crucial for Gaussian splats where the "size" of a splat can vary dramatically [S2, unofficial].

## Sources
[S1] antimatter15/splat, https://github.com/antimatter15/splat, fetched 2026-10-27 (unofficial)
[S2] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
