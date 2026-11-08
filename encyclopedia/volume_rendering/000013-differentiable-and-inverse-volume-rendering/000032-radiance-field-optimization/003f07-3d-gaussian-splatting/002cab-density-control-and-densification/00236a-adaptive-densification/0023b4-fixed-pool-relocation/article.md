# Fixed-Pool Relocation

Also called: Component Recycling, Dynamic Operations, Relocating low opacity components to high opacity component

## Overview
Fixed-Pool Relocation is a mechanism to redistribute splats every 50 iterations to improve coverage, using a fixed pool of splats [S1, unofficial]. It differs from standard densification by targeting low-residual, low-importance splats for relocation to high-error regions rather than cloning or splitting existing structures, thereby maintaining a strict budget while correcting positional errors [S1, unofficial].

## How it works
**Inputs:** Current set of Gaussian splats (position $\mu$, covariance $\Sigma$, opacity $O$, color $c$, Adam moments $m, v$), reconstruction residual $R = (V - \hat{V})_+$, iteration counter $t$.
**Outputs:** Updated set of splats with relocated components.

**Data Structures:**
*   **Splat:** Struct containing $\mu \in \mathbb{R}^3$, $\Sigma \in \mathbb{R}^{3 \times 3}$ (or quaternion/scale), $O \in [0,1]$, $c \in \mathbb{R}^3$, $m \in \mathbb{R}^3$, $v \in \mathbb{R}^3$.
*   **Residual Volume:** Voxel grid storing $R(x)$.

**Steps:**
1.  **Identify Candidates:** Calculate per-splat importance score $a_k = L_{k,k} \cdot \text{amplitude}$ (amplitude times approximate volume) [S1, unofficial]. Select the bottom 1% of splats that sit at low residual. Limit to at most 64 splats per step, ensuring at most one per detected residual peak [S1, unofficial].
2.  **Find Targets:** Perform non-maximum suppression over the residual volume to find local maxima of $R = (V - \hat{V})_+$ [S1, unofficial].
3.  **Relocate:** For each selected splat $i$ with target location $\mu_{target}$:
    *   Set new position: $\mu_{new} = \mu_{target}$ [S2].
    *   Reinitialize covariance isotropically: $\sigma = 0.5$ voxels [S1, unofficial].
    *   Zero Adam first- and second-moment estimates: $m = 0, v = 0$ [S1, unofficial].
    *   Apply cooldown: Mark splat as ineligible for relocation for $\ge 1$ subsequent steps [S1, unofficial].
4.  **Opacity Preservation:** To preserve the total distribution when moving $N$ new components to the location of an old component ($\mu_{new} = \mu_{old}$), update opacity using:
    $$ (1 - O_{new})^N = (1 - O_{old}) $$
    where $O_{new}$ is the opacity of the new components, $N$ is the number of new components, and $O_{old}$ is the opacity of the old component [S2].
5.  **Covariance Preservation:** Minimize the difference between integrated covariances:
    $$ \text{minimize } \left\| \int_{-\infty}^{\infty} C_{new}(x) - \int_{-\infty}^{\infty} C_{old}(x) \right\| dx $$
    This ensures the distribution before and after relocation is unchanged [S2].
6.  **Cache Optimization:** Every 1,000 iterations, reorder splats along a Morton (Z-order) curve to improve GPU cache locality [S1, unofficial].
7.  **Post-Fit Culling:** After fitting completes, remove splats accounting for the lowest fraction of cumulative amplitude. Default is 5% removal (95% retention), but benchmarks use 0.1% removal (99.9% retention) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fixed_pool_relocation(splats, residual_volume, iteration, max_relocations=64):
    if iteration % 50 != 0:
        return splats

    # Step 1: Identify bottom 1% low-residual splats
    scores = [compute_importance(s) for s in splats]
    sorted_indices = np.argsort(scores)
    num_candidates = max(1, len(splats) // 100)
    candidates = sorted_indices[:num_candidates]
    
    # Filter by cooldown and limit to max_relocations
    valid_candidates = [s for s in candidates if not s.cooldown_active]
    if len(valid_candidates) > max_relocations:
        valid_candidates = valid_candidates[:max_relocations]

    # Step 2: Find residual peaks via non-maximum suppression
    peaks = non_maximum_suppression(residual_volume)

    # Step 3: Relocate
    for i, splat in enumerate(valid_candidates):
        if i >= len(peaks):
            break
        target_pos = peaks[i]
        
        # Relocate position
        splat.mu = target_pos
        
        # Reinitialize covariance isotropically
        splat.sigma = 0.5  # voxels
        
        # Zero Adam moments
        splat.m = np.zeros(3)
        splat.v = np.zeros(3)
        
        # Set cooldown
        splat.cooldown_active = True
        splat.cooldown_timer = 1

    # Step 6: Cache optimization every 1000 iterations
    if iteration % 1000 == 0:
        splats = reorder_morton(splats)

    return splats

def compute_importance(splat):
    # Amplitude times approximate volume
    return splat.amplitude * splat.volume

def non_maximum_suppression(volume):
    # Find local maxima in positive residual volume
    peaks = []
    for x in volume:
        if is_local_max(x, volume):
            peaks.append(x.position)
    return peaks

def is_local_max(point, volume):
    # Check if point is a local maximum in its neighborhood
    neighbors = get_neighbors(point)
    return all(volume[p] <= volume[point] for p in neighbors)
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ per relocation step for candidate selection, where $N$ is the number of splats. Residual peak detection depends on voxel grid resolution. Morton reordering is $O(N \log N)$ or $O(N)$ with efficient sorting.
*   **Memory Complexity:** $O(N)$ for splat storage. Additional $O(V)$ for residual volume, where $V$ is the number of voxels.
*   **Performance:** Improves reconstruction quality in under-sampled regions by targeting high-error areas [S1, unofficial].
*   **Strengths:** Maintains fixed budget, corrects positional errors, improves cache locality [S1, unofficial].
*   **Limitations:** Requires residual volume computation, may relocate splats to suboptimal positions if residual peaks are noisy [S1, unofficial].
*   **When to use:** When reconstruction quality is limited by positional errors rather than density [S1, unofficial].
*   **When not to use:** When computational resources are extremely limited, as residual computation adds overhead [S1, unofficial].

## Background
Originated in Luxar, a Gaussian splatting framework for microscopy and scalable interactive web visualization [S1, unofficial]. The technique is also referred to as "Component Recycling" in the context of 3D Student Splatting [S2]. It addresses the need for dynamic adjustment of Gaussian distributions to maintain coverage and accuracy during optimization [S1, unofficial][S2].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
