# Texel Size Adaptation and Primitive Management

Also called: Adaptive Texture Allocation, Primitive Management via Quantile Error

## Overview
Texel Size Adaptation and Primitive Management is a training-time optimization routine that dynamically adjusts the resolution of per-primitive texture maps and the number of 3D Gaussian primitives to balance model capacity and appearance quality [S1]. It differs from sibling methods like [Adaptive Density Control](9188-adaptive-density-control.md) or [GroupFlow](6351-groupflow.md) by explicitly coupling the geometric density of primitives with the textural detail allocated to them, using quantile-based error thresholds to trigger splitting, upscaling, or downscaling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of 3D Gaussian primitives $\mathcal{G}$, where each primitive $k$ has an associated texture grid (texel map) and a computed error metric $E_i$ [S1].
*   **Output:** Updated primitive set $\mathcal{G}'$ with modified counts, scales, and texture resolutions [S1].

**Data Structures:**
*   **Primitive:** Contains mean position, covariance, opacity, spherical harmonics, and a texture grid [Parent].
*   **Texture Grid:** A 2D array of values (channels) representing local appearance details [S1].
*   **Resolution Definition:** The texture grid resolution is defined by $t2pr$ (texels per primitive axis) [S1].
*   **Error Metric $E_i$:** A scalar value representing the reconstruction error for primitive $i$, used to rank primitives for quantile-based decisions [S1].

**Parameters:**
*   $\tau_{tr}$: Texture resolution threshold. Controls splitting aggressiveness. Starts at 64 and is progressively reduced to 32 within 7000 iterations [S1].
*   $\tau_{ds}$: Downscale threshold. Controls downscaling ease. Higher values make downscaling easier, leading to simpler appearance and fewer texels [S1].
*   $\lambda_{texture}$: Controls texture variance. Higher values yield smoother results; lower values reduce texel count per primitive [S1].
*   $t2pr$: Texture size per primitive (texels per axis). Initialized at iteration 500 to the closest power of two such that the smallest axis is 8 texels wide [S1].
*   Quantile: Top 90% of errors are considered for splitting and upscaling [S1].
*   Interval: Routine runs every 250 iterations until 25,000 iterations [S1].

**Steps:**
1.  **Initialization:** At iteration 500, initialize texture grids to 0 and begin optimization [S1]. Set $t2pr$ for each primitive to the closest power of two such that the smallest axis is 8 texels wide [S1].
2.  **Periodic Check:** If `iter mod 250 == 0` and `iter <= 25000`:
    a.  Compute error $E_i$ for all primitives [S1].
    b.  Identify the set of primitives in the top 90% of errors [S1].
    c.  For each primitive $k$:
        i.  **Splitting:** If `texture resolution > τtr` AND $E_k$ is in the top 90%:
            *   Split the primitive along overflowing axes, following the parent [Densification Strategy](6234-densification-strategy.md) principles for replacing a parent with children [Parent].
        ii. **Upscaling:** If `t2pr > 1` AND $E_k$ is in the top 90%:
            *   Decrease texel size (Upscale) using a scale factor of 2 [S1].
        iii. **Downscaling:** Else if `E_d < τds`:
            *   Increase texel size (Downscale) using a scale factor of 2 [S1].
            *   *Note:* The source uses `E_d` in the condition but does not explicitly define it beyond implying it relates to the error or texture state [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texel_size_adaptation_and_primitive_management(iteration, primitives, tau_tr, tau_ds):
    """
    Adaptively manage primitive count and texture resolution.
    """
    # Routine runs every 250 iterations until 25,000
    if iteration % 250 != 0 or iteration > 25000:
        return primitives

    # Compute errors for all primitives
    errors = [p.error for p in primitives]
    if not errors:
        return primitives

    # Calculate the 90th percentile threshold for top 90% selection
    sorted_errors = sorted(errors)
    threshold_index = min(int(len(sorted_errors) * 0.9), len(sorted_errors) - 1)
    error_threshold = sorted_errors[threshold_index]

    updated_primitives = []
    for p in primitives:
        is_high_error = p.error >= error_threshold

        # Splitting Condition
        if p.texture_resolution > tau_tr and is_high_error:
            # Split primitive along overflowing axes
            # [G] Exact splitting geometry follows parent [Densification Strategy](6234-densification-strategy.md) principles
            # Typically replaces parent with children sampled from N(mu, Sigma) with reduced scale
            children = split_primitive(p)
            updated_primitives.extend(children)
            continue

        # Upscaling Condition
        if p.t2pr > 1 and is_high_error:
            p.upscale_texture(scale_factor=2)
            p.t2pr = p.t2pr * 2
        # Downscaling Condition (else if attached to upscaling block)
        elif p.E_d < tau_ds:
            p.downscale_texture(scale_factor=2)

    return updated_primitives

def split_primitive(primitive):
    # [G] Standard densification: replace parent with children sampled from N(mu, Sigma)
    # Scale reduction: s_child = s_parent * (5/4) / sqrt(N_children)
    return [primitive.clone() for _ in range(2)]
```

**Edge Cases and Pitfalls:**
*   **Hyperparameter Interdependence:** Changing $\tau_{tr}$ to encourage more primitives can indirectly lead to fewer texels per primitive, as each primitive represents a more localized, simpler part of the scene [S1].
*   **Memory Management:** Pruned or split primitives require buffer compaction to maintain contiguous memory access for rasterization [Parent].
*   **Initialization:** Texture grids start at 0 and begin optimizing after 500 iterations [S1].

## Operating characteristics
*   **Time Complexity:** The routine runs every 250 iterations [S1]. The cost is proportional to the number of primitives $N$ for error computation and thresholding [G]. Splitting and texture interpolation are constant time per primitive but increase $N$ over time [G].
*   **Memory Complexity:** Memory usage scales with the total number of texels [G]. The source reports 20.8M texels for 'drjohnson' and 9.0M for 'bonsai' [S1].
*   **Performance:** On the 'drjohnson' scene, the method achieves 77 FPS and 318MB memory with 293K points and 79.7M parameters [S1]. On 'bonsai', it achieves 35 FPS and 176MB memory with 291K points and 44.1M parameters [S1].
*   **Strengths:** Automatically adjusts model capacity based on scene complexity. Robust to different ratios of primitive and texel budgets [S1].
*   **Limitations:** Requires careful tuning of $\tau_{tr}$ and $\tau_{ds}$. The indirect effects of hyperparameters can make prediction of final model size difficult [S1].

## Background
The method was introduced by Panagiotis Papantonakis, Georgios Kopanas, Frédo Durand, and George Drettakis in "Content-Aware Texturing for Gaussian Splatting" [S1]. It addresses the need for efficient representation of high-frequency details in Gaussian Splatting by decoupling geometric density from textural resolution and managing both adaptively [S1].

## Sources
[S1] Content-Aware Texturing for Gaussian Splatting, https://doi.org/10.2312/sr.20251190, fetched 2026-10-26
