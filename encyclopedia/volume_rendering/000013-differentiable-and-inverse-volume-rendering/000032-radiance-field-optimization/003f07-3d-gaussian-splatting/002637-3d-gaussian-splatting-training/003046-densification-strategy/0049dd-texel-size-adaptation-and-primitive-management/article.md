# Texel Size Adaptation and Primitive Management

Also called: Adaptive Texture Allocation, Primitive Management via Quantile Error

## Overview
This method manages the capacity of 3D Gaussian splatting models by dynamically adjusting the resolution of per-primitive texture maps (texel size) and the number of Gaussian primitives based on quantile error thresholds [S1]. It runs periodically during training to balance model capacity and appearance quality, ensuring that resources are allocated to regions with high reconstruction error while simplifying low-error regions [S1]. It differs from sibling methods like [Score-based Densification](4705-score-based-densification.md) by explicitly linking texture resolution to primitive splitting, whereas siblings often rely solely on gradient magnitude or score-based heuristics for primitive count without explicit texture grid management.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current training iteration $iter$, set of active 3D Gaussian primitives $\mathcal{G}$, current texture grid resolutions for each primitive, hyperparameters $\tau_{tr}$, $\tau_{ds}$, $\lambda_{texture}$, and $t2pr$.
*   **Outputs:** Updated set of primitives (with potential splits), updated texture grid resolutions (upscaled or downscaled).

**Data Structures:**
*   **Primitive:** Contains position, covariance, opacity, spherical harmonics coefficients, and a 2D texture grid.
*   **Texture Grid:** A 2D array of values (e.g., RGB) associated with each primitive. The dimensions are defined by $t2pr$ (texture size per primitive).
*   **Error Metric ($E_i$):** A scalar error value computed for each primitive $i$, representing the reconstruction error attributable to that primitive [S1].

**Parameters:**
*   $\tau_{tr}$: Texture resolution threshold. Controls the aggressiveness of primitive splitting. Starts at 64 and is progressively reduced to 32 within the first 7000 iterations [S1].
*   $\tau_{ds}$: Downscale threshold. Controls the ease of texture downsampling. Higher values make downscaling easier, leading to simpler appearances and fewer texels [S1].
*   $\lambda_{texture}$: Variance control for texture maps. Higher values lead to smoother results; lower values result in fewer texels per primitive [S1].
*   $t2pr$: Texture size per primitive (in texels). Initialized at iteration 500 to the closest power of two such that the smallest axis is 8 texels wide [S1].
*   Quantile Threshold: Top 90% of errors are considered for splitting and upscaling [S1].

**Steps:**
1.  **Periodic Execution:** The routine executes every 250 iterations, starting from iteration 0 and continuing until iteration 25,000 [S1].
2.  **Initialization (Iteration 500):** If $iter = 500$, initialize texture grids with zero values for all channels. Set $t2pr$ for each primitive to the closest power of two such that the smallest axis is 8 texels wide [S1].
3.  **Error Computation:** For each primitive $i$, compute the error $E_i$ [S1].
4.  **Splitting Decision:** If the current texture resolution exceeds $\tau_{tr}$ AND $E_i$ is in the top 90% of all errors, split the primitive along overflowing axes [S1].
5.  **Upscaling Decision:** If $t2pr > 1$ AND $E_i$ is in the top 90% of all errors, decrease the texel size (upscale the texture grid) [S1].
6.  **Downscaling Decision:** Else if the downscale metric $E_d < \tau_{ds}$, increase the texel size (downscale the texture grid) [S1]. Note: The condition "Else if" implies this step is mutually exclusive with the upscaling step in the same iteration for a given primitive, or applies to primitives not selected for upscaling [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texel_size_adaptation_and_primitive_management(iter, primitives, tau_tr, tau_ds, lambda_texture, t2pr):
    """
    Adaptively manage primitive count and texture resolution based on error quantiles.
    
    Args:
        iter: Current training iteration.
        primitives: List of primitive objects, each with attributes:
            - error: Computed error E_i.
            - texture_resolution: Current texture resolution.
            - t2pr: Current texture size per primitive.
            - split(): Method to split the primitive along overflowing axes.
            - upscale(): Method to decrease texel size (increase resolution).
            - downscale(): Method to increase texel size (decrease resolution).
        tau_tr: Texture resolution threshold for splitting.
        tau_ds: Downscale threshold.
        lambda_texture: Variance control for texture maps.
        t2pr: Texture size per primitive.
    """
    # Execute every 250 iterations until 25,000
    if iter % 250 != 0 or iter > 25000:
        return

    # Initialization at iteration 500
    if iter == 500:
        for prim in primitives:
            prim.texture_grid = zeros_like(prim.texture_grid) # Initialize with 0
            # Set t2pr to closest power of two such that smallest axis is 8 texels
            prim.t2pr = closest_power_of_two(8) 
        return

    # Compute errors for all primitives
    errors = [prim.error for prim in primitives]
    # Determine the threshold for the top 90% of errors
    error_threshold = percentile(errors, 90)

    for prim in primitives:
        # Splitting: if texture resolution > tau_tr and error is in top 90%
        if prim.texture_resolution > tau_tr and prim.error >= error_threshold:
            prim.split()
        
        # Upscaling: if t2pr > 1 and error is in top 90%
        # Note: This is an independent if statement in the source, not an else-if
        if prim.t2pr > 1 and prim.error >= error_threshold:
            prim.upscale()
        # Downscaling: else if downscale metric < tau_ds
        # The source uses "else if", implying this branch is taken if upscaling was not triggered
        elif prim.error < error_threshold: # Assuming "else" refers to not being in top 90% for upscaling
            # The source says "else if Ed < tau_ds". Ed is not explicitly defined as a separate variable
            # from Ei, but context suggests it's the error metric for downscaling decision.
            # We assume Ed is the same as Ei for simplicity, or a related metric.
            # Given the structure "if ... then ... else if ...", it implies:
            # If upscaling condition met -> upscale.
            # Else (if upscaling condition NOT met) AND downscale condition met -> downscale.
            if prim.error < tau_ds: # Using Ei as Ed for downscale check
                prim.downscale()
```

**Edge Cases and Pitfalls:**
*   **Hyperparameter Interdependence:** Changes in one hyperparameter can have secondary, indirect effects. For example, lowering $\tau_{tr}$ encourages more primitives, which can lead to fewer texels per primitive because each primitive represents a more localized, simpler part of the scene [S1].
*   **Memory Constraints:** Early overfitting and memory exhaustion are mitigated by initializing texture grids at iteration 500 and controlling the rate of texture upsampling [S1].
*   **Quantile Sensitivity:** A lower quantile (e.g., top 50%) would make the changes more aggressive, affecting more primitives, while a higher quantile has the opposite effect [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   The routine runs every 250 iterations, reducing the overhead compared to per-step densification [S1].
*   Memory usage is directly tied to the number of primitives and texels. For example, on the 'drjohnson' scene, the method results in 293K points, 20.8M texels, and 79.7M parameters, consuming 318MB of memory [S1]. On the 'bonsai' scene, it uses 291K points, 9.0M texels, and 44.1M parameters, consuming 176MB [S1].

**Performance:**
*   On the 'drjohnson' scene, the method achieves 77 FPS [S1].
*   On the 'bonsai' scene, it achieves 35 FPS [S1].
*   The method is robust to different ratios of primitive and texel budgets, automatically adjusting capacity allocation [S1].

**Strengths and Limitations:**
*   **Strengths:** Automatically balances model capacity and appearance quality. Reduces memory usage by downsampling textures in low-error regions. Handles varying scene complexity by splitting primitives in high-error regions.
*   **Limitations:** Requires careful tuning of hyperparameters ($\tau_{tr}$, $\tau_{ds}$, $\lambda_{texture}$) to avoid suboptimal texture/primitive ratios. The indirect effects of hyperparameters can make prediction of final model size difficult.

## Background
This method was introduced by Panagiotis Papantonakis, Georgios Kopanas, Frédo Durand, and George Drettakis in their work on content-aware texturing for Gaussian Splatting [S1]. It addresses the need for efficient representation of complex scenes by adapting the texture resolution to the local detail required, rather than using a fixed resolution for all primitives. The approach builds on the principles of gradient-based triggering and periodic execution common in densification strategies, but adds explicit texture grid management.

## Sources
[S1] Content-Aware Texturing for Gaussian Splatting, https://doi.org/10.2312/sr.20251190, fetched 2026-10-26
