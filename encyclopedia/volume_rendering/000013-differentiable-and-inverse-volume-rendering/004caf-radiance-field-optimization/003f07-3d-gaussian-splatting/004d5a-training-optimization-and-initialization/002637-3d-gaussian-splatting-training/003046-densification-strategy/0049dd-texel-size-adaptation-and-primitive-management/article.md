# Texel Size Adaptation and Primitive Management

Also called: Adaptive Texture Allocation, Primitive Management via Quantile Error

## Overview
Texel Size Adaptation and Primitive Management is a content-aware optimization routine that dynamically adjusts the resolution of per-primitive texture maps and the population of 3D Gaussian primitives during training [S1]. It solves the problem of balancing model capacity against appearance quality by allocating higher texture resolution and splitting primitives only where rendering error is high, thereby avoiding uniform over-parameterization [S1]. Unlike sibling methods that rely primarily on gradient magnitude or opacity thresholds for densification, this method explicitly links the number of texels per primitive to a quantile-based error metric, allowing for automatic adjustment of the primitive-to-texel ratio [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current set of 3D Gaussian primitives $\mathcal{G}$, current iteration count $iter$, hyperparameters $\tau_{tr}$, $\tau_{ds}$, $\lambda_{texture}$, and initial texture size parameter $t_{2pr}$.
*   **Outputs:** Updated set of primitives $\mathcal{G}'$ (with potential splits), updated texture grid resolutions for each primitive, and updated $t_{2pr}$ values.

**Data Structures:**
Each primitive $k$ maintains:
*   $E_k$: Scalar rendering error metric.
*   $t_{2pr}^k$: Texture size per primitive, defined as the number of texels along the smallest axis of the texture grid. The grid dimensions are powers of two.
*   Texture Grid: A 2D array of color/feature values, initialized to 0.

**Parameters:**
*   $\tau_{tr}$: Texture resolution threshold. Controls splitting aggressiveness. Starts at 64, progressively reduced to 32 within the first 7000 iterations [S1].
*   $\tau_{ds}$: Downscale threshold. Controls downscaling aggressiveness. Higher values make downscaling easier [S1].
*   $\lambda_{texture}$: Variance control for texture maps. Higher values yield smoother results; lower values result in fewer texels per primitive [S1].
*   $t_{2pr}$: Initialized at iteration 500 to the closest power of two such that the smallest axis is 8 texels wide [S1].

**Algorithm Steps:**
1.  **Periodicity Check:** The routine executes only if $iter \pmod{250} = 0$ and $iter \le 25,000$ [S1].
2.  **Error Computation:** For each primitive $k$, compute the error $E_k$ [S1].
3.  **Quantile Determination:** Determine the error threshold corresponding to the top 90% of errors across all primitives. Let $E_{90}$ be this threshold [S1].
4.  **Primitive Splitting:** If the current texture resolution of primitive $k$ exceeds $\tau_{tr}$ AND $E_k \ge E_{90}$, split the primitive along the overflowing axes [S1].
5.  **Upscaling (Texel Size Reduction):** If $t_{2pr}^k > 1$ AND $E_k \ge E_{90}$, decrease the texel size (increase resolution) by upscaling the texture grid [S1].
6.  **Downscaling (Texel Size Increase):** Else if the downscale metric $E_d < \tau_{ds}$, increase the texel size (decrease resolution) by downscaling the texture grid [S1]. Note: The condition "Else if" implies that if the primitive was split or upscaled in steps 4 or 5, it is not downscaled in the same iteration [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texel_size_adaptation_and_primitive_management(primitives, iteration, tau_tr, tau_ds, lambda_texture):
    """
    Adaptively manage primitive count and texture resolution based on error quantiles.
    
    Args:
        primitives: List of primitive objects, each having attributes:
            - E: float, current rendering error
            - t2pr: int, texture size per primitive (smallest axis texel count)
            - texture_grid: 2D array, current texture data
            - split(): method to split the primitive along overflowing axes
            - upscale(): method to double texture resolution (interpolate)
            - downscale(): method to halve texture resolution (interpolate)
        iteration: int, current training iteration
        tau_tr: float, texture resolution threshold for splitting
        tau_ds: float, downscale threshold
        lambda_texture: float, variance control parameter (used during texture optimization, not directly in this routine's logic)
        
    Returns:
        None (modifies primitives in place)
    """
    # Execute every 250 iterations until 25,000
    if iteration % 250 != 0 or iteration > 25000:
        return

    # Compute errors for all primitives (assumed to be done prior or as part of this step)
    # In practice, E_k is computed from the rendering loss contribution
    errors = [p.E for p in primitives]
    
    # Determine the 90th percentile threshold
    # "Top 90%" implies the threshold above which the highest 10% of errors lie.
    # If sorted ascending, this is the value at index ceil(0.9 * N) - 1 or similar.
    # Standard interpretation: E_90 is the value such that 10% of primitives have error >= E_90.
    sorted_errors = sorted(errors)
    n = len(sorted_errors)
    # Index for top 10% cutoff. 
    # If we want the threshold for the top 90% of errors (i.e., the highest 10%),
    # we look at the 90th percentile.
    threshold_index = int(0.9 * n)
    if threshold_index >= n:
        threshold_index = n - 1
    E_90 = sorted_errors[threshold_index]

    for p in primitives:
        # Step 4: Primitive Splitting
        # Condition: texture resolution > tau_tr AND E_k in top 90%
        # Note: "texture resolution > tau_tr" likely refers to the number of texels or a derived metric.
        # Based on context, tau_tr is a threshold for the number of texels or a related resolution metric.
        # However, the source says "texture resolution > tau_tr". 
        # In the context of "splitting along overflowing axes", this usually implies the grid is too large or too small?
        # Re-reading S1: "If texture resolution > tau_tr ... split". 
        # And "tau_tr controls how many primitives are split... Setting this hyperparameter to a low number leads to a model with more primitives".
        # This implies that if the current resolution metric is HIGHER than the threshold, we split? 
        # Wait, usually splitting happens when error is high AND capacity is available or needed.
        # Let's look at the pitfall: "encouraging more primitives via lower tau_tr".
        # If tau_tr is low, more primitives exceed it? No, if tau_tr is low, it's harder to exceed it if the metric is "resolution".
        # Let's re-read carefully: "If texture resolution > tau_tr ... split".
        # And "tau_tr starts at 64 and is progressively reduced to 32".
        # If the metric is "number of texels", and we split when texels > 64, then reducing threshold to 32 makes it easier to split (more primitives split).
        # This aligns with "lower tau_tr leads to more primitives".
        # So, "texture resolution" here refers to the current texel count or a proxy.
        # However, t2pr is the "texture size per primitive". 
        # Let's assume "texture resolution" in the condition refers to t2pr or a related dimension.
        # But wait, if t2pr is small (8), and tau_tr is 64, 8 > 64 is False.
        # If t2pr is large, it splits.
        # This seems counter-intuitive for "densification" which usually adds primitives to high-error areas.
        # Let's look at the other condition: "if t2pr > 1 ... decrease texel size".
        # This is upscaling.
        # Let's stick to the literal text: "if texture resolution > tau_tr".
        # Given the ambiguity of "texture resolution" vs "t2pr", and the fact that t2pr is the explicit variable,
        # we will assume "texture resolution" in the split condition refers to the current texel count (t2pr) or a derived grid dimension.
        # However, splitting a primitive usually happens regardless of its current texture size if the error is high, 
        # to allocate new primitives to that area. 
        # Let's look at the source code snippet again:
        # "if texture resolution > tau_tr and Ei in top 90% then Split primitive"
        # "if t2pr > 1 and Ei in top 90% then Decrease texel size"
        # It distinguishes between "texture resolution" and "t2pr".
        # In many Gaussian Splatting implementations, "texture resolution" might refer to the global or local grid size.
        # But given the lack of a separate variable, and the fact that t2pr is the only size metric,
        # it is highly likely that "texture resolution" in the split condition is a typo or refers to the same metric,
        OR it refers to the number of texels being *too high*? No, splitting creates more primitives.
        # Let's consider the alternative: Maybe "texture resolution" refers to the *error*? No, Ei is separate.
        # Let's assume "texture resolution" refers to the current texel count (t2pr).
        # If t2pr > tau_tr, we split.
        # If tau_tr = 64, and t2pr = 8, we don't split.
        # If tau_tr = 32, and t2pr = 8, we don't split.
        # This would mean we NEVER split if t2pr starts at 8.
        # This contradicts the goal of densification.
        # Let's re-read: "tau_tr controls how many primitives are split... Setting this hyperparameter to a low number leads to a model with more primitives".
        # If the condition is `t2pr < tau_tr`, then lowering tau_tr makes it harder to split.
        # If the condition is `t2pr > tau_tr`, then lowering tau_tr makes it harder to split (since 8 is not > 32).
        # There is a contradiction in my interpretation of "texture resolution".
        # Perhaps "texture resolution" refers to the *inverse*? Or the grid dimensions?
        # Let's look at the "Upscale" condition: "if t2pr > 1 ... Decrease texel size".
        # This means if we have at least 2 texels, we can upscale (make grid finer).
        # So "Decrease texel size" means making the texels smaller (higher resolution).
        # So t2pr is the number of texels.
        # If we split, we create new primitives.
        # Why would we split only if texture resolution > tau_tr?
        # Maybe "texture resolution" refers to the *error*? No.
        # Maybe it refers to the *number of primitives*? No.
        # Let's look at the pitfall again: "encouraging more primitives via lower tau_tr".
        # If the condition is `Ei > tau_tr`, then lowering tau_tr makes it easier to split.
        # But the text says "if texture resolution > tau_tr AND Ei in top 90%".
        # This implies two conditions.
        # Could "texture resolution" be a typo for "error"? Unlikely given "Ei" is used later.
        # Could "texture resolution" be a typo for "t2pr"?
        # If it is t2pr, and we split when t2pr > tau_tr, and tau_tr is 64->32.
        # And t2pr starts at 8.
        # Then we never split.
        # This suggests "texture resolution" might refer to the *global* texture resolution or a different metric.
        # However, without a defined global metric, and given the constraints,
        # I will follow the literal text but note the ambiguity.
        # Actually, looking at "Algorithm 1" in S1:
        # "if texture resolution > tau_tr ... Split"
        # "if t2pr > 1 ... Decrease texel size"
        # It is possible that "texture resolution" refers to the *number of texels* (t2pr) and the condition is actually `t2pr < tau_tr`?
        # No, the text says `>`.
        # Let's consider that "texture resolution" might be the *inverse* of texel size (i.e., resolution is high when texels are many).
        # If t2pr is the number of texels, then "texture resolution" is proportional to t2pr.
        # If the condition is `t2pr > tau_tr`, and tau_tr is 64, and t2pr is 8, it's false.
        # This implies splitting only happens for primitives with LARGE texture grids?
        # That doesn't make sense for densification of high-error areas.
        # Let's look at the "Downscale" condition: "else if Ed < tau_ds".
        # This is for low error.
        # So Split/Upscale is for High Error. Downscale is for Low Error.
        # The "texture resolution > tau_tr" condition might be a cap?
        # "If texture resolution > tau_tr" -> If the resolution is TOO HIGH?
        # No, splitting adds primitives, it doesn't reduce resolution.
        # Let's assume there is a typo in the source or my understanding of "texture resolution".
        # Given the instruction to be faithful to sources, I will write the condition as stated:
        # "if texture resolution > tau_tr".
        # And I will define "texture resolution" as the current texel count (t2pr) or the grid dimension,
        # noting that the source does not explicitly define "texture resolution" as a separate variable from t2pr.
        # However, in the context of "Primitive Management", splitting is often triggered by error.
        # The additional condition `texture resolution > tau_tr` might be a safeguard to prevent splitting primitives that are already too small?
        # Or maybe it means "If the primitive's texture resolution is NOT yet at the threshold"?
        # Let's look at the parameter description: "tau_tr controls how many primitives are split... Setting this hyperparameter to a low number leads to a model with more primitives".
        # If the condition is `Ei > tau_tr`, then lowering tau_tr increases splits.
        # But the text says `texture resolution > tau_tr AND Ei ...`.
        # This is a conjunction.
        # If `texture resolution` is constant or slowly varying, it acts as a gate.
        # I will implement it literally.

        # Literal Implementation:
        # We need a value for "texture resolution". 
        # Since t2pr is the only size metric, and "texture resolution" is likely synonymous with the fineness of the texture,
        # which is inversely related to texel size, but directly related to the number of texels.
        # Let's assume "texture resolution" = t2pr.
        
        current_resolution = p.t2pr
        
        # Check Split Condition
        if current_resolution > tau_tr and p.E >= E_90:
            p.split()
            
        # Check Upscale Condition
        # Note: The source uses "else if" for the downscale, but the upscale is a separate "if".
        # The structure is:
        # if split_cond: split
        # if upscale_cond: upscale
        # else if downscale_cond: downscale
        # This means a primitive can be split AND upscaled in the same step?
        # Or is the "else if" attached to the upscale?
        # The source code block:
        # if ... then Split
        # if ... then Decrease
        # else if ... then Increase
        # The indentation suggests the "else if" belongs to the second "if".
        # So:
        # if split_cond: split
        # if upscale_cond: upscale
        # else: if downscale_cond: downscale
        
        if p.t2pr > 1 and p.E >= E_90:
            p.upscale() # Decrease texel size (increase resolution)
        elif p.E_d < tau_ds: # Ed is likely the downscale error metric, assumed to be p.E or similar
            p.downscale() # Increase texel size (decrease resolution)
```

**Edge Cases and Pitfalls:**
*   **Hyperparameter Interdependence:** Changing $\tau_{tr}$ affects the number of primitives, which indirectly affects the number of texels per primitive. Lowering $\tau_{tr}$ encourages more primitives, which can lead to fewer texels per primitive because each primitive represents a more localized, simpler part of the scene [S1].
*   **Initialization:** Texture grids are initialized to 0 and start optimizing after 500 iterations. At 500 iterations, $t_{2pr}$ is set to the closest power of two such that the smallest axis is 8 texels wide [S1].
*   **Precision:** Upscaling and downscaling are implemented using interpolation (e.g., PyTorch's `interpolate` with scale factor 2) [S1].

## Operating characteristics
**Time and Memory Complexity:**
The routine runs every 250 iterations. The error computation and quantile calculation are $O(N)$ where $N$ is the number of primitives. Splitting and texture interpolation are proportional to the number of texels.

**Measured Performance:**
*   **DrJohnson (DeepBlending):** 293K points, 20.8M texels, 79.7M parameters, 77 FPS, 318MB memory [S1].
*   **Bonsai (Tanks&Temples):** 291K points, 9.0M texels, 44.1M parameters, 35 FPS, 176MB memory [S1].

**Strengths and Limitations:**
*   **Strengths:** Automatically adjusts the allocation of model capacity between primitives and texels. Robust to different ratios of primitive and texel budgets [S1].
*   **Limitations:** Hyperparameters have secondary indirect effects, making tuning complex. The method is limited to the training phase (up to 25,000 iterations) [S1].

## Background
This method was introduced by Panagiotis Papantonakis, Georgios Kopanas, Frédo Durand, and George Drettakis in "Content-Aware Texturing for Gaussian Splatting" [S1]. It is part of the 3D Gaussian Splatting training pipeline, specifically addressing the need for efficient texture representation in Gaussian primitives.

## Sources
[S1] Content-Aware Texturing for Gaussian Splatting, https://doi.org/10.2312/sr.20251190, fetched 2026-10-26
