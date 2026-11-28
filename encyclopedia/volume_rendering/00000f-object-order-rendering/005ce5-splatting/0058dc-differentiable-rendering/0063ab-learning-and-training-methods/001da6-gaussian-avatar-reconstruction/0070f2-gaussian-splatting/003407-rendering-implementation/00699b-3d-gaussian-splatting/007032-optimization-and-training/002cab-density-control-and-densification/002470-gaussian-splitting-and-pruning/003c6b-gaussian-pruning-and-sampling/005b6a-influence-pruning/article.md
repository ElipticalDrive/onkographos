# Influence Pruning
Also called: Accumulated Weight Pruning, Weight-based Pruning

## Overview
Influence Pruning is a density control method for 3D Gaussian Splatting that removes Gaussians with low impact on the final rasterized image, thereby reducing memory usage and training time without degrading reconstruction quality [S1]. Unlike standard opacity reset, which only removes transparent Gaussians, this method identifies and discards Gaussians that are occluded (transmittance $T=0$) or have negligible scale, even if their intrinsic opacity is high [S1].

## How it works
**Inputs and Outputs:**
- Input: The set of 3D Gaussians $\{G_i\}$ with attributes (position, covariance/scale, opacity $\alpha_i$, spherical harmonics) and the accumulated influence weights $W_i$ [S1].
- Output: A reduced set of Gaussians after removing those with the lowest accumulated weights [S1].

**Data Structures:**
- **Accumulated Weight Vector:** A 1D array $W$ of size $N_G$ (number of Gaussians), where $W_i$ stores the sum of influence weights over $N$ iterations [S1].

**Formulas:**
The influence of the $i$-th Gaussian on a specific pixel $p$ during rasterization is defined as [S1]:
$$ W_{i,p} = \alpha_i T_i = \alpha_i \prod_{j=1}^{i-1}(1-\alpha_j) $$
where $T_i$ is the transmittance up to Gaussian $i$, representing the probability that the ray reaches Gaussian $i$ without being blocked by previous Gaussians in the depth-sorted order [S1].

The total influence weight for Gaussian $i$ across the entire image is the sum of its pixel-wise influences [S1]:
$$ W_i = \sum_{p} W_{i,p} $$

**Parameters:**
- **Accumulation Interval ($N$):** The number of iterations over which weights are accumulated before pruning. Not explicitly given as a single number, but weights are accumulated over $N$ iterations to account for all training views [S1].
- **Pruning Frequency:** Every 5,000 iterations [S1].
- **Pruning Duration:** Until 25,000 iterations [S1].
- **Pruning Percentage:** 15% of Gaussians are removed at each stage [S1].
- **Total Training Iterations:** Typically 30,000, though controllable based on budget [S1].

**Steps:**
1. **Initialize:** Set accumulated weight vector $W = \mathbf{0}$ for all Gaussians [G].
2. **Training Loop:**
   a. Render the current view and compute $W_{i,p}$ for each Gaussian $i$ at each pixel $p$ during the rasterization pass [S1].
   b. Accumulate the weights: $W_i \leftarrow W_i + W_{i,p}$ for all $i, p$ [S1].
   c. Increment iteration counter $t$ [G].
3. **Pruning Check:** If $t$ is a multiple of 5,000 and $t \le 25,000$:
   a. Identify the bottom 15% of Gaussians based on their current accumulated weight $W_i$ [S1].
   b. Remove these Gaussians from the scene representation [S1].
   c. Continue accumulating weights for the remaining Gaussians without resetting $W$ [G].
4. **Continue Training:** Repeat steps 2-3 until convergence or max iterations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Parameters
max_iterations = 30000
pruning_interval = 5000
max_pruning_iter = 25000
prune_percentage = 0.15

# Initialization
gaussians = [...] # List of Gaussian objects with .alpha attribute
accumulated_weights = [0.0] * len(gaussians)

for t in range(1, max_iterations + 1):
    # 1. Rasterization & Weight Accumulation
    current_weights = [0.0] * len(gaussians)
    for p in pixels:
        T = 1.0
        for i in visible_gaussians_for_pixel(p): # Depth-sorted visible set [G]
            T *= (1.0 - gaussians[i].alpha)
            W_i_p = gaussians[i].alpha * T
            current_weights[i] += W_i_p

    # 2. Accumulate weights over N iterations
    accumulated_weights = [accumulated_weights[i] + current_weights[i] for i in range(len(gaussians))]

    # 3. Standard training step (forward/backward pass) [G]

    # 4. Pruning Stage
    if t % pruning_interval == 0 and t <= max_pruning_iter:
        num_to_prune = int(len(gaussians) * prune_percentage)
        sorted_indices = sorted(range(len(gaussians)), key=lambda k: accumulated_weights[k])
        prune_indices = set(sorted_indices[:num_to_prune])

        gaussians = [g for idx, g in enumerate(gaussians) if idx not in prune_indices]
        accumulated_weights = [accumulated_weights[idx] for idx in range(len(gaussians))]
```

**Edge Cases and Pitfalls:**
- **Occlusion:** Gaussians may have high opacity but zero influence if they are occluded by previous Gaussians ($T_i=0$). These are correctly identified as having $W_i=0$ and pruned [S1].
- **Small Scale:** Gaussians with very small scales influence few pixels, resulting in low summed weights $W_i$, and are pruned [S1].
- **Precision:** The product $\prod (1-\alpha_j)$ can suffer from underflow if many Gaussians are rendered. Standard rasterization handles this via floating-point precision [G].
- **Over-pruning:** Removing more than 15% may degrade reconstruction quality [S1].

## Subtopics

### Kernel Pruning Strategy
An adaptive density control method that deletes kernels contributing little to the final texture based on hit count and energy score [S2]. During an accumulation window $\Delta_{prune}$, it records the hit count $\nu(\kappa_s)$ (frequency of selection in the local partition of unity) and the energy score $\omega(\kappa_s)$ (aggregated filtered weights) for every kernel $\kappa_s$ [S2]. Periodically, kernels are pruned if they are rarely hit ($\nu(\kappa_s) < \gamma_{hit} \cdot \max(\{\nu(\kappa_i)\}_{i=1}^S)$) or energetically weak ($\omega(\kappa_s) < \gamma_{energy}$) [S2].

## Operating characteristics
- **Time Complexity:** The weight calculation has no computational overhead as it is computed directly during the rasterization pass [S1]. The pruning step itself requires sorting or selection, taking $O(N_G \log N_G)$ or $O(N_G)$ time [G].
- **Memory Complexity:** Reduces memory usage by decreasing the number of stored Gaussians [S1].
- **Performance:** Reduces training and rendering times significantly by eliminating redundant Gaussians [S1].
- **Strengths:** Effectively removes occluded and small Gaussians that opacity reset misses [S1].
- **Limitations:** Requires accumulating weights over multiple iterations, which may delay the removal of initially important but later redundant Gaussians [G].

## Background
Influence Pruning was introduced in the EAGLES (Efficient Accelerated 3D Gaussians with Lightweight EncodingS) paper by S. Girish et al. [S1]. It addresses the redundancy problem in 3D Gaussian Splatting, where a significant fraction of Gaussians do not contribute to the final image, leading to high memory and training costs [S1]. The method builds on the standard 3D-GS opacity reset but extends it to consider geometric occlusion and scale [S1].

## Sources
[S1] EAGLES: Efficient Accelerated 3D Gaussians with Lightweight EncodingS, https://doi.org/10.1007/978-3-031-73036-8_4, fetched 2026-10-27
[S2] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
