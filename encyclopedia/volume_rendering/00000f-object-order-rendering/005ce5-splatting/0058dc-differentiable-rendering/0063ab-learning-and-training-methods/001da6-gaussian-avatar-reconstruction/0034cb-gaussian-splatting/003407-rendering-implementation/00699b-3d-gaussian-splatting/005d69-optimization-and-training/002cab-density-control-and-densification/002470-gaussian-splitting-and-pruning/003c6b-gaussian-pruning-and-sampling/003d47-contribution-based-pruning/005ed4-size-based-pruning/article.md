# Size-based Pruning

Also called: Scale-based Pruning

## Overview
Size-based Pruning eliminates 3D Gaussians that are structurally redundant due to their small physical size, as they contribute minimally to the scene reconstruction [S1]. It differs from [Opacity Pruning](14627-opacity-pruning.md) by targeting geometric scale rather than alpha transparency, and from [Contribution-based Pruning](9445-contribution-based-pruning.md) by using a single attribute (scale) instead of a composite error metric [S1].

## How it works
**Inputs and Outputs:**
- Input: A set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has a scale parameter $s_i$ (or covariance matrix $\Sigma_i$).
- Output: A pruned subset $\mathcal{G}' \subset \mathcal{G}$ containing only Gaussians with significant size.

**Data Structures:**
Each Gaussian $g_i$ is represented by its attributes, including the scale vector $\mathbf{s}_i = [s_{i,x}, s_{i,y}, s_{i,z}]$ or the diagonal elements of the covariance matrix. The scale determines the spatial extent of the Gaussian's influence.

**Steps:**
1. **Identify Small Gaussians:** For each Gaussian $g_i$, compute its effective size. This is typically derived from the scale parameters. A Gaussian is considered "small" if its scale values fall below a predefined threshold $\tau$ [S1].
2. **Remove Redundant Gaussians:** Eliminate all Gaussians identified in Step 1 from the scene representation [S1].

**Formulas:**
Let $s_{i,k}$ be the $k$-th component of the scale vector for Gaussian $i$. A Gaussian is pruned if:
$$ \min_k(s_{i,k}) < \tau $$
where $\tau$ is the pruning threshold [S1].

**Parameters:**
- $\tau$: The minimum scale threshold. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Scale values close to zero may cause numerical instability in rendering or optimization if not handled [G].
- **Boundaries:** Aggressive pruning may remove fine details if $\tau$ is set too high [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def size_based_pruning(gaussians, threshold):
    """
    Prunes Gaussians with scale components below the threshold.
    
    Args:
        gaussians: List of Gaussian objects, each having a 'scale' attribute 
                   (a vector or array of size 3).
        threshold: Float, the minimum scale value for a Gaussian to survive.
        
    Returns:
        List of pruned Gaussian objects.
    """
    pruned_gaussians = []
    for g in gaussians:
        # Check if all scale components are above the threshold
        # Assuming g.scale is an iterable of scale values
        if all(s >= threshold for s in g.scale):
            pruned_gaussians.append(g)
        # Else: Gaussian is removed (structurally redundant due to small size)
    return pruned_gaussians
```

## Subtopics

### Spatial-based Pruning
Removes Gaussians based on scene location, preserving detail in important regions while reducing redundancy elsewhere [S1]. PUP 3D-GS [29] employs a second-order reconstruction error approximation to selectively prune Gaussians with minimal impact on scene reconstruction [S1]. RTGS [31], using a foveated rendering (FR) approach for Point-Based Neural Rendering (PBNR) [90], prunes Gaussians based on pixel eccentricity [93], maintaining high-density in critical regions while sparsifying peripheral areas [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$, where $N$ is the number of Gaussians, as it requires a single pass to check scale values [S1].
- Memory: Reduces memory usage by removing redundant Gaussians [S1].

**Measured Performance:**
- CompGS [22] and Papantonakis et al. [24] apply size-based pruning to improve efficiency while maintaining reconstruction quality [S1].
- Specific metrics (PSNR, FPS) are not provided for size-based pruning alone in the sources, but general compression techniques show FPS improvements on NVIDIA A40 GPU [S1].

**Scaling:**
- Linear scaling with the number of Gaussians [S1].

**Strengths and Limitations:**
- Strengths: Simple to implement, effective at removing structurally redundant Gaussians [S1].
- Limitations: May not account for the visual importance of small Gaussians in certain contexts [S1].

**When to Use:**
- When computational efficiency is a priority and the scene contains many small, insignificant Gaussians [S1].

**When Not to Use:**
- When fine details are critical and small Gaussians contribute significantly to the visual quality [S1].

## Background
Size-based Pruning is a technique used in 3D Gaussian Splatting to optimize scene representation by removing redundant Gaussians [S1]. It is part of a broader category of pruning strategies including gradient-based, opacity-based, spatial-based, and significance-scoring-based techniques [S1]. The method is applied in works such as CompGS [22] and Papantonakis et al. [24] [S1].

## Sources
[S1] Compression in 3D Gaussian Splatting: A Survey of Methods, Trends, and Future Directions, https://doi.org/10.1109/TCSVT.2026.3676048, fetched 2026-10-26
