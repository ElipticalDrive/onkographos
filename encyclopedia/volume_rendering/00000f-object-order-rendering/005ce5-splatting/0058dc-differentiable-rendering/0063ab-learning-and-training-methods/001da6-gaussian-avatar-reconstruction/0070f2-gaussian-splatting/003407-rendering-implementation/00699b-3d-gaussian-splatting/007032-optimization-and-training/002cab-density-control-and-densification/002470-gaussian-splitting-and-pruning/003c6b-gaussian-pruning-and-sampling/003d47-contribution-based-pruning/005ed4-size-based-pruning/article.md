# Size-based Pruning

Also called: Scale-based Pruning

## Overview
Size-based Pruning eliminates 3D Gaussians that are structurally redundant due to their small size, as they contribute minimally to scene reconstruction [S1]. It operates as a specific strategy within [Contribution-based Pruning](9445-contribution-based-pruning.md) that evaluates geometric scale rather than opacity or composite error metrics [S1].

## How it works
**Inputs and Outputs:**
- Input: A set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has a scale parameter [G].
- Output: A pruned subset $\mathcal{G}' \subset \mathcal{G}$ containing only Gaussians with significant size [G].

**Data Structures:**
Each Gaussian $g_i$ stores a scale vector $\mathbf{s}_i = [s_{i,x}, s_{i,y}, s_{i,z}]$ representing its spatial extent [G].

**Steps:**
1. Identify Gaussians with small size (scale) [S1].
2. Remove these Gaussians [S1].

**Formulas:**
Not given in the sources [S1].

**Parameters:**
- Threshold value: Not given in the sources [S1].

**Edge Cases and Pitfalls:**
- Scale values near zero may cause numerical instability during rendering or optimization [G].

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
        # Exact thresholding criterion (e.g., max component vs magnitude) is not specified in the sources [G].
        if all(s >= threshold for s in g.scale):
            pruned_gaussians.append(g)
    return pruned_gaussians
```

## Subtopics

### Spatial-based Pruning
Removes Gaussians based on scene location, preserving detail in important regions while reducing redundancy elsewhere [S1]. PUP 3D-GS [29] employs a second-order reconstruction error approximation to selectively prune Gaussians with minimal impact on scene reconstruction [S1]. RTGS [31], using a foveated rendering (FR) approach for Point-Based Neural Rendering (PBNR) [90], prunes Gaussians based on pixel eccentricity [93], maintaining high-density in critical regions while sparsifying peripheral areas [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$, where $N$ is the number of Gaussians, as it requires a single pass to check scale values [G].
- Memory: Reduces memory usage by removing redundant Gaussians [S1].

**Measured Performance:**
- CompGS [22] and Papantonakis et al. [24] apply size-based pruning to improve efficiency while maintaining reconstruction quality [S1].
- Specific metrics (PSNR, FPS) are not provided for size-based pruning alone in the sources, but general compression techniques show FPS improvements on NVIDIA A40 GPU [S1].

**Scaling:**
- Linear scaling with the number of Gaussians [G].

**Strengths and Limitations:**
- Strengths: Simple to implement, effective at removing structurally redundant Gaussians [S1].

**When to Use:**
- When computational efficiency is a priority and the scene contains many small, insignificant Gaussians [G].

**When Not to Use:**
- When fine details are critical and small Gaussians contribute significantly to the visual quality [G].

## Background
Size-based Pruning is a technique used in 3D Gaussian Splatting to optimize scene representation by removing redundant Gaussians [S1]. It is part of a broader category of pruning strategies including gradient-based, opacity-based, spatial-based, and significance-scoring-based techniques [S1]. The method is applied in works such as CompGS [22] and Papantonakis et al. [24] [S1].

## Sources
[S1] Compression in 3D Gaussian Splatting: A Survey of Methods, Trends, and Future Directions, https://doi.org/10.1109/TCSVT.2026.3676048, fetched 2026-10-26
