# Visibility-Based Pruning

Also called: Gradient-based Pruning, Zero-gradient Pruning

## Overview
Visibility-Based Pruning removes surfels that failed to contribute to the training signal during the backward pass, indicating they were persistently occluded, outside camera frustums, or failed kernel cutoff tests [S1, unofficial]. It differs from opacity-based pruning by relying on gradient flow rather than static opacity thresholds, ensuring only geometrically relevant primitives are retained [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Set of surfels $S$, cumulative screen-space gradient magnitudes $g_i$ for each surfel $i$, and training epoch counter.
- Output: Pruned set of surfels $S' \subset S$.

**Data Structures:**
- `surfel`: Contains position, orientation, scale, opacity, and `cumulative_gradient` (float).
- `cumulative_gradient`: Accumulates the magnitude of the gradient of the loss with respect to the surfel's parameters projected onto screen space across all views in an epoch.

**Steps:**
1. **Accumulate Gradients:** During the training epoch, for each surfel $i$ rendered in each view $v$, compute the gradient of the loss $\mathcal{L}$ with respect to the surfel's parameters. Project this gradient to screen space and compute its magnitude $\|\nabla_{screen} \mathcal{L}_i\|$. Add this magnitude to `surfel[i].cumulative_gradient`.
2. **Check Zero Gradient:** At the end of the epoch, iterate through all surfels $i \in S$.
3. **Prune:** If `surfel[i].cumulative_gradient` is exactly zero, remove surfel $i$ from $S$.

**Formula:**
$$ S' = \{ s_i \in S \mid \sum_{v \in \text{Views}} \|\nabla_{screen} \mathcal{L}_i^{(v)}\| > 0 \} $$

Where:
- $S$: Set of surfels at start of epoch.
- $S'$: Set of surfels after pruning.
- $s_i$: The $i$-th surfel.
- $\nabla_{screen} \mathcal{L}_i^{(v)}$: Screen-space gradient of the loss for surfel $i$ in view $v$.

**Parameters:**
- Threshold: 0.0 (exact zero check).

**Edge Cases and Pitfalls:**
- **Numerical Precision:** Ensure that "zero" is checked against floating-point epsilon if gradients are extremely small but non-zero due to numerical noise. However, the source specifies "zero gradient," implying exact zero from no backward pass reach [S1, unofficial].
- **First Epoch:** If a surfel is initialized but never rendered (e.g., completely outside all frustums), its gradient will be zero, and it will be pruned immediately. This is intended behavior.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def visibility_based_pruning(surfels, epoch_end_flag):
    """
    Removes surfels with zero cumulative screen-space gradient magnitude.
    """
    if not epoch_end_flag:
        return surfels

    pruned_surfels = []
    for surfel in surfels:
        # cumulative_gradient is accumulated during forward/backward passes
        if surfel.cumulative_gradient > 0.0:
            pruned_surfels.append(surfel)
        # Else: surfel is removed (never reached in backward pass)
    
    return pruned_surfels
```

## Subtopics

### Isolation-Based Pruning
Removes surfels that have no neighbors within their kernel support radius, identified as isolated primitives with low accumulated weight and coverage [S1, unofficial]. It complements visibility pruning by removing geometrically degenerate single-surfel groups that do not form coherent surface layers [S1, unofficial]. Choose this when surface coherence is critical and isolated noise points are present [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per epoch, where $N$ is the number of surfels, for the pruning step. Gradient accumulation is part of the standard rendering cost.
- **Memory Complexity:** $O(N)$ to store cumulative gradients.
- **Strengths:** Efficiently removes geometrically irrelevant surfels without requiring opacity thresholds. Ensures only surfels contributing to the image are kept.
- **Limitations:** Requires full backward pass accumulation. May be sensitive to gradient precision.
- **When to Use:** When training inverse rendering models with surfels and occlusion is common.
- **When Not to Use:** If real-time inference without training is the only goal (pruning is a training-time optimization).

## Background
Originated in the 3DSS (3D Surface Splatting for Inverse Rendering) paper [S1, unofficial]. It is part of the adaptive density control strategy for 3D Gaussian Splatting variants that use surfels. The method leverages the fact that surfels not reached in the backward pass carry no scene information [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
