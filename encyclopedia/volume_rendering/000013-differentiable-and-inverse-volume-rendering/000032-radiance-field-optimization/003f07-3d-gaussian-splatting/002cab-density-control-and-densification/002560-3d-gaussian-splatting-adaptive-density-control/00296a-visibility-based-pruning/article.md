# Visibility-Based Pruning

Also called: Gradient-based Pruning, Zero-gradient Pruning

## Overview
Visibility-Based Pruning removes surfels that were never reached during the backward pass, which occurs when they are persistently occluded, lie outside all camera frustums, or fail the kernel cutoff test at every pixel [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Set of surfels $S$ with accumulated screen-space gradient magnitudes across all training views, and a flag indicating the end of a training epoch.
- Output: Pruned set of surfels $S' \subset S$.
[S1, unofficial]

**Data Structures:**
- `surfel`: Contains geometric and appearance parameters, plus a `cumulative_gradient` field (float) that accumulates the magnitude of the loss gradient projected onto screen space across all views in an epoch.
[S1, unofficial]

**Steps:**
1. **Accumulate Gradients:** During the training epoch, for each surfel $i$ rendered in each view $v$, compute the gradient of the loss $\mathcal{L}$ with respect to the surfel's parameters, project it to screen space, and compute its magnitude $\|\nabla_{\text{screen}} \mathcal{L}_i^{(v)}\|$. Add this magnitude to `surfel[i].cumulative_gradient`.
2. **Check Zero Gradient:** At the end of the epoch, iterate through all surfels $i \in S$.
3. **Prune:** If `surfel[i].cumulative_gradient` equals zero, remove surfel $i$ from $S$.
[S1, unofficial]

**Formula:**
$$ S' = \{ s_i \in S \mid \sum_{v \in \text{Views}} \|\nabla_{\text{screen}} \mathcal{L}_i^{(v)}\| > 0 \} $$
Where:
- $S$: Set of surfels at the start of the epoch.
- $S'$: Set of surfels after pruning.
- $s_i$: The $i$-th surfel.
- $\nabla_{\text{screen}} \mathcal{L}_i^{(v)}$: Screen-space gradient of the loss for surfel $i$ in view $v$.
[S1, unofficial]

**Parameters:**
- Threshold: $0.0$ (exact zero check).
[S1, unofficial]

**Edge Cases and Pitfalls:**
- **Numerical Precision:** The source specifies an exact zero check, implying that any non-zero accumulation survives pruning [S1, unofficial]. Implementers should ensure gradients are not artificially clamped to zero during accumulation. [G]
- **First Epoch:** If a surfel is initialized but never rendered (e.g., completely outside all frustums), its gradient will be zero, and it will be pruned immediately. This is intended behavior. [G]

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
        if surfel.cumulative_gradient > 0.0:
            pruned_surfels.append(surfel)
        # Else: surfel is removed (never reached in backward pass)
        surfel.cumulative_gradient = 0.0  # Reset cumulative_gradient for next epoch [G]
    
    return pruned_surfels
```

## Subtopics

### Isolation-Based Pruning
Removes surfels that have no neighbors within their kernel support radius, identified as isolated primitives with low accumulated weight and coverage [S1, unofficial]. It complements visibility pruning by removing geometrically degenerate single-surfel groups that do not form coherent surface layers [S1, unofficial]. Choose this when surface coherence is critical and isolated noise points are present [S1, unofficial].

### Anchor Pruning with View-Frequency
Removes redundant and ineffective anchors that cause artifacts (floaters) by using 'view-frequency' instead of opacity alone [S2]. View-frequency is defined as the probability that an anchor is selected in training views, which correlates with the gradient received [S2]. Compute the average opacity of Gaussians generated over $T$ training iterations for each anchor [S2]. Calculate the view-frequency for each anchor [S2]. Remove anchors where the view-frequency is below a visibility threshold $\tau_v$ [S2]. Parameter $\epsilon$: A small user-defined quantity added to $\Delta L$ to prevent stagnation, typically $0.01$ [S2]. Pitfall: Insufficiently optimized anchors (low view-frequency) produce artifacts if pruning is based solely on opacity, as they are not visible or selected in most training view frustums and are not sufficiently optimized [S2].

## Operating characteristics
Time and memory complexity are not given in the sources. The method requires $O(N)$ memory to store cumulative gradients per surfel, where $N$ is the number of surfels [S1, unofficial]. Gradient accumulation occurs during the standard rendering cost [G]. The method is strictly a training-time optimization and is not used for real-time inference without training [G]. It efficiently removes primitives that carry no scene information [S1, unofficial].

## Background
It is part of the adaptive density control strategy for 3D Gaussian Splatting variants that use surfels [S1, unofficial]. The method leverages the fact that surfels not reached in the backward pass carry no scene information [S1, unofficial].

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S2] Octree-GS: Towards Consistent Real-time Rendering with LOD-Structured 3D Gaussians, https://doi.org/10.1109/tpami.2025.3568201, fetched 2026-10-27
