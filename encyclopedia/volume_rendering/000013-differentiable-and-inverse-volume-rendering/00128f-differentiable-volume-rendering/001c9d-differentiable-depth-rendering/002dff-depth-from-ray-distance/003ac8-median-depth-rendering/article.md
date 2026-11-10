# Median Depth Rendering

Also called: Depth Renderer (Median)

## Overview
Median Depth Rendering computes the geometric depth of a ray by identifying the sample point where the accumulated opacity crosses a specific threshold, typically 0.5 [S1, unofficial]. Unlike Expected Depth Rendering, which integrates the entire ray's contribution to produce a weighted average, Median Depth Rendering selects a single representative sample index based on cumulative weight [S2, unofficial]. This method provides a stable geometric signal for pseudo-normal supervision and ray-tracing initialization, offering superior reconstruction quality and downstream stability compared to inverse-depth variants [S1, unofficial].

## How it works
**Inputs:**
- `z_vals`: Array of shape `(N,)` containing the depth coordinates of the $N$ samples along the ray [S1, unofficial].
- `weights`: Array of shape `(N,)` containing the density weights (opacity contribution) for each sample [S1, unofficial].
- `depth_threshold`: Scalar accumulation threshold (default 0.5) [S1, unofficial].

**Outputs:**
- `depth`: Scalar depth value for the ray [S1, unofficial].

**Data Structures:**
- `cumulative_weights`: Array of shape `(N,)` storing the prefix sum of `weights`.

**Steps:**
1. Compute the cumulative sum of the weights along the ray:
   $$ W_i = \sum_{j=0}^{i} w_j $$
   where $w_j$ is the weight at sample $j$ and $W_i$ is the cumulative weight at sample $i$ [S1, unofficial].
2. Find the index $k$ such that $W_k \ge \text{depth\_threshold}$ and $W_{k-1} < \text{depth\_threshold}$ [S1, unofficial]. If no such index exists (e.g., total weight < threshold), return the last sample depth or handle as edge case [G].
3. Return the depth value at the identified index:
   $$ \text{depth} = z_k $$
   [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp

def median_depth_rendering(z_vals: jnp.ndarray, weights: jnp.ndarray, depth_threshold: float = 0.5) -> jnp.ndarray:
    """
    Compute median depth by finding the sample where cumulative opacity crosses the threshold.
    
    Args:
        z_vals: Depth samples along the ray, shape (N,).
        weights: Density weights for each sample, shape (N,).
        depth_threshold: Accumulation threshold (default 0.5).
        
    Returns:
        depth: Scalar depth value.
    """
    # Compute cumulative weights
    cumulative_weights = jnp.cumsum(weights)
    
    # Find the first index where cumulative weight >= threshold
    # jnp.argmax returns the first index where the condition is true if we use boolean mask
    # However, argmax on boolean array returns index of first True.
    # We need to ensure we handle the case where threshold is never reached.
    mask = cumulative_weights >= depth_threshold
    
    # If no sample reaches the threshold, use the last sample [G]
    if not jnp.any(mask):
        return z_vals[-1]
    
    depth_index = jnp.argmax(mask)
    
    return z_vals[depth_index]
```

**Parameters:**
- `depth_threshold`: Default 0.5 [S1, unofficial].

**Edge Cases:**
- If the total accumulated weight is less than `depth_threshold`, the method must define a fallback. The pseudocode uses the last sample [G].
- Precision issues near the threshold boundary may cause instability if weights are very small [G].

## Subtopics

### [Expected Depth Rendering](8816-expected-depth-rendering.md)
Expected Depth Rendering estimates scene depth by computing the weighted average of depth samples along a ray, normalized by the accumulated weight. This method provides a stable geometric signal for pseudo-normal supervision and ray-tracing initialization, offering superior reconstruction quality and downstream stability compared to inverse-depth variants. It differs from Median Depth Rendering, which identifies the sample index where cumulative opacity crosses a threshold, by integrating the entire ray's contribution rather than selecting a single median point.

## Operating characteristics
**Time Complexity:** $O(N)$ to compute cumulative weights and find the threshold index, where $N$ is the number of samples per ray.
**Memory Complexity:** $O(N)$ to store cumulative weights.
**Strengths:** Provides a robust geometric estimate that is less sensitive to outliers than expected depth. It is computationally cheap and differentiable if implemented with soft thresholding or if gradients are only needed for the network, not the depth selection itself [S2, unofficial].
**Limitations:** The depth value is discrete (tied to sample locations), which can cause quantization artifacts. It does not account for the distribution of weight beyond the threshold crossing.
**When to use:** When a stable, single-value depth estimate is needed for supervision or initialization, and robustness to noise is prioritized over smoothness [S1, unofficial].
**When not to use:** When a smooth, continuous depth representation is required, or when the full depth distribution is needed for uncertainty estimation.

## Background
Median Depth Rendering is used in differentiable volume rendering pipelines, particularly in NeRF-based methods like Nerfies [S1, unofficial]. It serves as an alternative to expected depth for providing geometric constraints. The method is implemented in libraries such as Nerfies [S1, unofficial] and Ganerf [S2, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
