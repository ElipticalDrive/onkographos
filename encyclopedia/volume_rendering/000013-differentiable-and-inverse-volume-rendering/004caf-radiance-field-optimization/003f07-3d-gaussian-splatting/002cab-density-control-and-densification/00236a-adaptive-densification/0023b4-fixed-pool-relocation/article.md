# Fixed-Pool Relocation

Also called: Component Recycling, Dynamic Operations, Relocating low opacity components to high opacity component

## Overview
Fixed-Pool Relocation is a mechanism to redistribute splats every 50 iterations to improve coverage using a fixed pool of splats [S1, unofficial]. It differs from standard densification by targeting low-residual, low-importance splats for relocation to high-error regions rather than cloning or splitting existing structures, thereby maintaining a strict budget while correcting positional errors [S1, unofficial].

## How it works
**Inputs:** Current set of Gaussian splats (position $\mu$, covariance $\Sigma$, opacity $O$, color $c$, Adam moments $m, v$), reconstruction residual $R = (V - \hat{V})_+$, iteration counter $t$ [S1, unofficial].
**Outputs:** Updated set of splats with relocated components [S1, unofficial].

**Data Structures:**
*   **Splat:** Struct containing $\mu \in \mathbb{R}^3$, $\Sigma \in \mathbb{R}^{3 \times 3}$ (or quaternion/scale), $O \in [0,1]$, $c \in \mathbb{R}^3$, $m \in \mathbb{R}^3$, $v \in \mathbb{R}^3$ [S1, unofficial][G].
*   **Residual Volume:** Voxel grid storing $R(x)$ [G].

**Steps:**
1.  **Identify Candidates:** Calculate per-splat importance score $a_k = L_{k,k} \cdot \text{amplitude}$ (amplitude times approximate volume) [S1, unofficial]. Select the bottom 1% of splats that sit at low residual [S1, unofficial]. Limit to at most 64 splats per step, ensuring at most one per detected residual peak [S1, unofficial].
2.  **Find Targets:** Perform non-maximum suppression over the residual volume to find local maxima of $R = (V - \hat{V})_+$ [S1, unofficial].
3.  **Relocate:** For each selected splat $i$ with target location $\mu_{target}$: Set new position to the residual peak location $\mu_{target}$ [S1, unofficial]. Reinitialize covariance isotropically: $\sigma = 0.5$ voxels [S1, unofficial]. Zero Adam first- and second-moment estimates: $m = 0, v = 0$ [S1, unofficial]. Apply cooldown: Mark splat as ineligible for relocation for $\ge 1$ subsequent steps [S1, unofficial].
4.  **Opacity Preservation:** To preserve the total distribution when moving $N$ new components to the location of an old component ($\mu_{new} = \mu_{old}$), update opacity using:
    $$ (1 - O_{new})^N = (1 - O_{old}) $$
    where $O_{new}$ is the opacity of the new components, $N$ is the number of new components, and $O_{old}$ is the opacity of the old component [S2].
5.  **Covariance Preservation:** Minimize the difference between integrated covariances:
    $$ \text{minimize } \left\| \int_{-\infty}^{\infty} C_{new}(x) - \int_{-\infty}^{\infty} C_{old}(x) \right\| dx $$
    This ensures the distribution before and after relocation is unchanged [S2].
6.  **Cache Optimization:** Every 1,000 iterations, reorder splats along a Morton (Z-order) curve to improve GPU cache locality [S1, unofficial].
7.  **Post-Fit Culling:** After fitting completes, remove splats accounting for the lowest fraction of cumulative amplitude [S1, unofficial]. Default is 5% removal (95% retention), but benchmarks use 0.1% removal (99.9% retention) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fixed_pool_relocation(splats, residual_volume, iteration, max_relocations=64):
    if iteration % 50 != 0:
        return splats

    # Step 1: Identify bottom 1% low-residual splats
    scores = [compute_importance(s) for s in splats]
    sorted_indices = sorted(range(len(scores)), key=lambda i: scores[i])
    num_candidates = max(1, len(splats) // 100)
    candidates = [splats[i] for i in sorted_indices[:num_candidates]]

    # Filter by cooldown and limit to max_relocations
    valid_candidates = [s for s in candidates if not s.cooldown_active]
    if len(valid_candidates) > max_relocations:
        valid_candidates = valid_candidates[:max_relocations]

    # Step 2: Find residual peaks via non-maximum suppression
    peaks = find_residual_peaks(residual_volume)

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
        splat.m = [0.0, 0.0, 0.0]
        splat.v = [0.0, 0.0, 0.0]

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

def find_residual_peaks(volume):
    # Find local maxima in positive residual volume
    peaks = []
    for x in range(volume.width):
        for y in range(volume.height):
            for z in range(volume.depth):
                if is_local_max(x, y, z, volume.width, volume.height, volume.depth):
                    peaks.append(volume.get_position(x, y, z))
    return peaks

def is_local_max(x, y, z, w, h, d):
    # Check if point is a local maximum in its 3D neighborhood
    val = volume.get(x, y, z)
    if val <= 0:
        return False
    for nx, ny, nz in get_3d_neighbors(x, y, z, w, h, d):
        if volume.get(nx, ny, nz) > val:
            return False
    return True

def get_3d_neighbors(x, y, z, w, h, d):
    # Return valid 3D grid neighbors
    neighbors = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                nx, ny, nz = x + dx, y + dy, z + dz
                if 0 <= nx < w and 0 <= ny < h and 0 <= nz < d:
                    neighbors.append((nx, ny, nz))
    return neighbors

def reorder_morton(splats):
    # Reorder splats along a Morton (Z-order) curve
    return sorted(splats, key=lambda s: compute_morton_key(s.mu))

def compute_morton_key(point):
    # Encode 3D coordinates into a Morton code
    x, y, z = point
    return expand_bits(x) | (expand_bits(y) << 1) | (expand_bits(z) << 2)

def expand_bits(v):
    v = (v | (v << 16)) & 0x00030000
    v = (v | (v << 8))  & 0x000300FF
    v = (v | (v << 4))  & 0x00C30C30
    v = (v | (v << 2))  & 0x09249249
    v = (v | (v << 1))  & 0x01249249
    return v
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Improves reconstruction quality in under-sampled regions by targeting high-error areas [S1, unofficial].
*   **Strengths:** Maintains fixed budget, corrects positional errors, improves cache locality [S1, unofficial].
*   **Limitations:** Requires residual volume computation, may relocate splats to suboptimal positions if residual peaks are noisy [S1, unofficial].

## Background
Originated in Luxar, a Gaussian splatting framework for microscopy and scalable interactive web visualization [S1, unofficial]. The technique is also referred to as "Component Recycling" in the context of 3D Student Splatting [S2]. It addresses the need for dynamic adjustment of Gaussian distributions to maintain coverage and accuracy during optimization [S1, unofficial][S2].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] 3D Student Splatting and Scooping, https://doi.org/10.1109/cvpr52734.2025.01960, fetched 2026-10-29
