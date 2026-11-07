# Median Depth Rendering
Also called: Depth Renderer (Median)
## Overview
Median Depth Rendering computes the depth of a ray by determining the weighted median of sample depths along the ray, where weights are derived from the sample opacities. It differs from Expected Depth Rendering, which computes the weighted mean, by providing a robust estimate that is less sensitive to outliers or multi-modal depth distributions [S1, unofficial].
## How it works
**Inputs:**
- `depths`: Array of sample depths along the ray, shape $(N,)$.
- `weights`: Array of sample weights (typically opacity contributions), shape $(N,)$.

**Outputs:**
- `depth`: Scalar depth value for the ray.

**Data Structures:**
- `depths`: Floating-point array.
- `weights`: Floating-point array, non-negative.

**Steps:**
1.  **Sort:** Sort the samples based on `depths` in ascending order. Let the sorted depths be $d_{(1)}, d_{(2)}, \dots, d_{(N)}$ and the corresponding sorted weights be $w_{(1)}, w_{(2)}, \dots, w_{(N)}$.
2.  **Cumulative Sum:** Compute the cumulative sum of the sorted weights:
    $$ W_k = \sum_{i=1}^{k} w_{(i)} $$
    for $k = 1, \dots, N$. Let $W_{total} = W_N$.
3.  **Find Median Index:** Find the smallest index $k$ such that:
    $$ W_k \ge \frac{W_{total}}{2} $$
4.  **Output:** The median depth is $d_{(k)}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def median_depth_rendering(depths: list[float], weights: list[float]) -> float:
    """
    Computes the weighted median depth.
    
    Args:
        depths: List of sample depths along the ray.
        weights: List of sample weights (opacities) corresponding to depths.
        
    Returns:
        The weighted median depth.
    """
    # Pair depths and weights, then sort by depth
    samples = sorted(zip(depths, weights), key=lambda x: x[0])
    
    # Extract sorted depths and weights
    sorted_depths = [s[0] for s in samples]
    sorted_weights = [s[1] for s in samples]
    
    # Calculate total weight
    total_weight = sum(sorted_weights)
    
    # Handle edge case of zero total weight
    if total_weight == 0:
        # [G] Return the first depth or a default value if no valid samples exist.
        return sorted_depths[0] if sorted_depths else 0.0
        
    # Find the median index
    cumulative_weight = 0.0
    half_weight = total_weight / 2.0
    
    for i, w in enumerate(sorted_weights):
        cumulative_weight += w
        if cumulative_weight >= half_weight:
            return sorted_depths[i]
            
    # [G] Fallback if loop completes without returning (should not happen if total_weight > 0)
    return sorted_depths[-1]
```

**Parameters:**
- `method`: Strategy for computing depth. Set to `'median'` to use the weighted median depth [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Weights:** If all weights are zero, the total weight is zero. The implementation must handle this, e.g., by returning the first depth or a default value [G].
- **Precision:** Floating-point precision in cumulative sum calculation may affect the exact median index in cases of very small weights.
- **Degenerate Input:** If $N=0$, the function should handle empty inputs gracefully [G].

## Subtopics

### Expected Depth Rendering
A renderer that computes the expected depth of a ray by calculating the weighted mean of sample depths. It is chosen when a smooth, average depth estimate is preferred over a robust median, but it is more sensitive to outliers in multi-modal depth distributions. Link: [Expected Depth Rendering](5666-expected-depth-rendering.md) (Note: Link placeholder as per instruction to link to siblings, though no specific file name provided for this variant, it is a sibling concept).

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ due to sorting, where $N$ is the number of samples per ray.
- **Memory Complexity:** $O(N)$ to store sorted depths and weights.
- **Strengths:** Robust to outliers; provides a single, well-defined depth value even in complex scenes with multiple surfaces.
- **Limitations:** Less smooth than expected depth; may jump between surfaces in noisy or sparse sampling scenarios.
- **When to Use:** When robustness to outliers is critical, or when the depth distribution is multi-modal and a single representative depth is needed.
- **When Not to Use:** When a smooth, average depth is required, or when computational efficiency is paramount and $N$ is large (consider approximate methods).

## Background
Median Depth Rendering is part of the differentiable volume rendering framework, specifically within the depth from ray distance methods. It is implemented in the `ganerf` repository by Barbaroessle [S1, unofficial]. The method leverages the statistical property of the median to provide a robust depth estimate from volumetric samples.

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
