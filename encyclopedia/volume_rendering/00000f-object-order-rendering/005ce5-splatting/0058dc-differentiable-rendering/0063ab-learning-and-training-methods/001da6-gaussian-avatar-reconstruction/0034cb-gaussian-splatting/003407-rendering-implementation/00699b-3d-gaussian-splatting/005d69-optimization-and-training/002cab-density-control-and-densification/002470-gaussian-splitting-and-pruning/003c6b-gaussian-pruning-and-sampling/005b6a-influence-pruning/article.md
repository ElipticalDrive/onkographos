# Influence Pruning
Also called: Accumulated Weight Pruning, Weight-based Pruning

## Overview
Influence Pruning is a density control method for 3D Gaussian Splatting that removes Gaussians with low impact on the final rasterized image, thereby reducing memory usage and training time without degrading reconstruction quality [S1]. Unlike standard opacity reset, which only removes transparent Gaussians, this method identifies and discards Gaussians that are occluded (transmittance $T=0$) or have negligible scale, even if their intrinsic opacity is high [S1].

## How it works
**Inputs and Outputs:**
- Input: The set of 3D Gaussians $\{G_i\}$, their current attributes (position, covariance, opacity $\alpha$, scale), and the accumulated influence weights $W_i$.
- Output: A reduced set of Gaussians with those having the lowest accumulated weights removed.

**Data Structures:**
- **Gaussian Attributes:** Each Gaussian $i$ stores position $\mathbf{x}_i$, covariance matrix $\Sigma_i$ (or scale/rotation), opacity $\alpha_i$, and spherical harmonics coefficients for color.
- **Accumulated Weight Vector:** A 1D array $W$ of size $N_G$ (number of Gaussians), where $W_i$ stores the sum of influence weights over $N$ iterations. Initialized to zero.

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
1. **Initialize:** Set accumulated weight vector $W = \mathbf{0}$ for all Gaussians. Set iteration counter $t=0$.
2. **Training Loop:**
   a. Perform a training step (forward pass/rasterization and backward pass/gradient update) for the current view [S1].
   b. During rasterization, for each Gaussian $i$ rendered at pixel $p$, compute $W_{i,p}$ using the formula above [S1].
   c. Accumulate the weights: $W_i \leftarrow W_i + W_{i,p}$ for all $i, p$ [S1].
   d. Increment $t$.
3. **Pruning Check:** If $t$ is a multiple of 5,000 and $t \le 25,000$:
   a. Identify the bottom 15% of Gaussians based on their current accumulated weight $W_i$ [S1].
   b. Remove these Gaussians from the scene representation [S1].
   c. Reset or continue accumulating weights for the remaining Gaussians (implementation detail: weights should persist to reflect long-term influence, so they are not reset to zero, but the count of Gaussians decreases) [G].
4. **Continue Training:** Repeat steps 2-3 until convergence or max iterations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
gaussians = initialize_gaussians() # List of Gaussian objects
accumulated_weights = np.zeros(len(gaussians))
pruning_interval = 5000
max_pruning_iter = 25000
prune_percentage = 0.15

for t in range(1, max_iterations + 1):
    # 1. Forward Pass / Rasterization
    # During rasterization, for each pixel p and visible Gaussian i:
    # Compute transmittance T_i and influence W_i_p
    # Note: This happens implicitly in the CUDA kernel. 
    # We assume a function that returns per-pixel influences for all Gaussians.
    pixel_influences = rasterize(gaussians, camera) # Shape: (num_gaussians, num_pixels)
    
    # 2. Accumulate Weights
    # Sum influences across all pixels for each Gaussian
    current_weights = np.sum(pixel_influences, axis=1)
    accumulated_weights += current_weights
    
    # 3. Backward Pass / Optimization
    optimize_gaussians(gaussians, loss)
    
    # 4. Pruning Stage
    if t % pruning_interval == 0 and t <= max_pruning_iter:
        # Identify indices of the lowest weights
        num_to_prune = int(len(gaussians) * prune_percentage)
        prune_indices = np.argsort(accumulated_weights)[:num_to_prune]
        
        # Remove Gaussians
        # Note: This requires careful handling of indices if done in-place
        gaussians = remove_gaussians(gaussians, prune_indices)
        
        # Update accumulated weights to match remaining Gaussians
        # The weights for remaining Gaussians persist
        accumulated_weights = np.delete(accumulated_weights, prune_indices)
```

**Edge Cases and Pitfalls:**
- **Occlusion:** Gaussians may have high opacity but zero influence if they are occluded by previous Gaussians ($T_i=0$). These are correctly identified as having $W_i=0$ and pruned [S1].
- **Small Scale:** Gaussians with very small scales influence few pixels, resulting in low summed weights $W_i$, and are pruned [S1].
- **Precision:** The product $\prod (1-\alpha_j)$ can suffer from underflow if many Gaussians are rendered. However, standard rasterization handles this via floating-point precision [G].
- **Over-pruning:** Removing more than 15% may degrade reconstruction quality [S1].

## Operating characteristics
- **Time Complexity:** The pruning step itself is $O(N_G \log N_G)$ due to sorting, where $N_G$ is the number of Gaussians. The weight accumulation is $O(N_G \cdot N_P)$ per iteration, where $N_P$ is the number of pixels, but this is done during the standard rasterization pass, adding no extra computational overhead [S1].
- **Memory Complexity:** Reduces memory usage by decreasing the number of Gaussians stored [S1].
- **Performance:** Reduces training and rendering times significantly by eliminating redundant Gaussians [S1].
- **Strengths:** Effectively removes occluded and small Gaussians that opacity reset misses [S1].
- **Limitations:** Requires accumulating weights over multiple iterations, which may delay the removal of initially important but later redundant Gaussians [G].

## Background
Influence Pruning was introduced in the EAGLES (Efficient Accelerated 3D Gaussians with Lightweight EncodingS) paper by S. Girish et al. [S1]. It addresses the redundancy problem in 3D Gaussian Splatting, where a significant fraction of Gaussians do not contribute to the final image, leading to high memory and training costs [S1]. The method builds on the standard 3D-GS opacity reset but extends it to consider geometric occlusion and scale [S1].

## Sources
[S1] EAGLES: Efficient Accelerated 3D Gaussians with Lightweight EncodingS, https://doi.org/10.1007/978-3-031-73036-8_4, fetched 2026-10-27
