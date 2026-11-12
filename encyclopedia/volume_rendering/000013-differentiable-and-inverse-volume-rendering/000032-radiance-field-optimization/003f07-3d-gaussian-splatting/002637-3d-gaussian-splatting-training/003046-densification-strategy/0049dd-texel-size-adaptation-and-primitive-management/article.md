# Texel Size Adaptation and Primitive Management

Also called: Adaptive Texture Allocation, Primitive Management via Quantile Error

## Overview
This method manages the capacity of 3D Gaussian primitives and their associated texture maps by adapting texel resolution and splitting primitives based on quantile error thresholds [S1]. It solves the problem of balancing model complexity against appearance quality by ensuring that high-error regions receive more geometric detail (via splitting) and higher texture resolution (via upscaling), while low-error regions are simplified [S1]. It differs from sibling methods like [Score-based Densification](4705-score-based-densification.md) by explicitly linking the number of texels per primitive to the geometric density, rather than treating texture resolution as a static or globally uniform parameter.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current set of 3D Gaussian primitives, their associated texture grids, current iteration count, and hyperparameters ($\tau_{tr}$, $\tau_{ds}$, $\lambda_{texture}$, $t_{2pr}$).
*   **Outputs:** Updated set of primitives (potentially split), updated texture grid resolutions (potentially upscaled or downscaled).

**Data Structures:**
*   **Primitive:** Contains 3D mean, covariance, opacity, and a reference to its texture grid.
*   **Texture Grid:** A 2D array of values (e.g., RGB) associated with each primitive. The dimensions are defined by $t_{2pr}$ (texture size per primitive).
*   **Error Metric ($E_i$):** A scalar value representing the reconstruction error for primitive $i$. The specific formula for $E_i$ is not detailed in the sources, but it is computed per primitive [S1].

**Parameters:**
*   $\tau_{tr}$: Texture resolution threshold. Controls splitting aggressiveness. Starts at 64, reduced to 32 by iteration 7000 [S1].
*   $\tau_{ds}$: Downscale threshold. Controls downscaling aggressiveness. Higher values make downscaling easier [S1].
*   $\lambda_{texture}$: Controls texture variance. Higher values yield smoother results; lower values result in fewer texels per primitive [S1].
*   $t_{2pr}$: Texture size per primitive (in texels). Initialized at iteration 500 to the closest power of two such that the smallest axis is 8 texels wide [S1].
*   Quantile: Top 90% of errors are considered for splitting/upscaling [S1].

**Steps:**
1.  **Initialization:** At iteration 500, initialize texture grids to 0. Set $t_{2pr}$ for each primitive to the closest power of two such that the smallest axis is 8 texels wide [S1].
2.  **Periodic Execution:** Every 250 iterations, until iteration 25,000, perform the following for all primitives [S1]:
    a. Compute error $E_i$ for each primitive $i$ [S1].
    b. Determine if $E_i$ is in the top 90% of all errors [S1].
    c. **Split Condition:** If current texture resolution $> \tau_{tr}$ AND $E_i$ is in the top 90%, split the primitive along overflowing axes [S1].
    d. **Upscale Condition:** If $t_{2pr} > 1$ AND $E_i$ is in the top 90%, decrease texel size (Upscale) [S1].
    e. **Downscale Condition:** Else if $E_d < \tau_{ds}$, increase texel size (Downscale) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texel_size_adaptation_and_primitive_management(iteration, primitives, tau_tr, tau_ds, t2pr_init_value):
    """
    Manages primitive splitting and texture resolution adaptation.
    
    Args:
        iteration: Current training iteration.
        primitives: List of primitive objects, each with attributes:
            - error: Computed error E_i.
            - texture_resolution: Current texture resolution.
            - t2pr: Current texture size per primitive.
            - split(): Method to split the primitive along overflowing axes.
            - upscale(): Method to decrease texel size (increase resolution).
            - downscale(): Method to increase texel size (decrease resolution).
        tau_tr: Texture resolution threshold for splitting.
        tau_ds: Downscale threshold.
        t2pr_init_value: Initial t2pr value (used for initialization logic if needed, 
                         though initialization happens separately at iter 500).
    
    Returns:
        Updated list of primitives.
    """
    # Initialization phase (once at iteration 500)
    if iteration == 500:
        for prim in primitives:
            # Set t2pr to closest power of two such that smallest axis is 8 texels wide
            # Implementation detail: This requires knowing the primitive's bounding box or 
            # projected size to determine the "smallest axis". 
            # The source states: "in practice using the closest power of two value" [S1].
            # We assume a helper function exists to compute this based on primitive geometry.
            prim.t2pr = compute_closest_power_of_two_for_8_texels(prim)
            # Initialize texture grid to 0
            prim.texture_grid = initialize_zeros(prim.t2pr)
        return primitives

    # Periodic execution phase (every 250 iterations until 25,000)
    if iteration % 250 == 0 and iteration <= 25000:
        # Compute errors for all primitives
        # Note: The exact formula for E_i is not provided in the source [S1].
        # This is a placeholder for the error computation logic.
        for prim in primitives:
            prim.error = compute_primitive_error(prim)
        
        # Determine the 90th percentile of errors to identify the top 90%
        all_errors = [prim.error for prim in primitives]
        error_threshold_90pct = get_percentile(all_errors, 90)
        
        for prim in primitives:
            is_high_error = prim.error >= error_threshold_90pct
            
            # Split Condition
            # "if texture resolution > tau_tr and Ei in top 90% then Split primitive along overflowing axes" [S1]
            if prim.texture_resolution > tau_tr and is_high_error:
                prim.split()
                # Note: Splitting may create new primitives. The source does not specify 
                # if the loop continues to process new primitives in the same iteration.
                # Assuming standard behavior: new primitives are added to the set for future iterations.
            
            # Upscale Condition
            # "if t2pr > 1 and Ei in top 90% then Decrease texel size (Upscale)" [S1]
            if prim.t2pr > 1 and is_high_error:
                prim.upscale()
            
            # Downscale Condition
            # "else if Ed < tau_ds then Increase texel size (Downscale)" [S1]
            # Note: The source uses "Ed" here. It is assumed to be the error E_i for the current primitive.
            # The "else if" implies this check is only performed if the upscale condition was NOT met?
            # Or is it independent? The indentation in the source suggests it's an else-if to the upscale block.
            # However, logically, downscaling should be independent of upscaling.
            # Given the source structure:
            # if ... split ...
            # if ... upscale ...
            # else if ... downscale ...
            # This implies downscaling only happens if upscale did NOT happen.
            # This is a potential pitfall: a high-error primitive might be split but not upscaled if t2pr=1,
            # and then not downscaled. A low-error primitive would not be split or upscaled, and might be downscaled.
            elif prim.error < tau_ds:
                prim.downscale()
    
    return primitives

def compute_closest_power_of_two_for_8_texels(prim):
    """
    Computes the closest power of two for t2pr such that the smallest axis is 8 texels wide.
    This requires knowledge of the primitive's spatial extent.
    Since the source does not provide the exact geometric calculation, this is a placeholder [G].
    In practice, this likely involves projecting the primitive's covariance to screen space 
    and determining the pixel dimensions, then scaling to ensure the smallest dimension is 8 texels.
    """
    # Placeholder for geometric calculation
    pass

def compute_primitive_error(prim):
    """
    Computes the reconstruction error E_i for a primitive.
    The exact formula is not provided in the source [S1].
    """
    pass

def get_percentile(errors, percentile):
    """
    Computes the value below which a given percentage of observations in a group of observations falls.
    """
    sorted_errors = sorted(errors)
    index = int(len(sorted_errors) * (percentile / 100.0))
    return sorted_errors[index]
```

**Edge Cases and Pitfalls:**
*   **Hyperparameter Interdependence:** Changing $\tau_{tr}$ to encourage more primitives can lead to fewer texels per primitive because each primitive represents a more localized, simpler part of the scene [S1].
*   **Quantile Sensitivity:** A lower quantile makes changes more aggressive (more primitives upscaled/split), while a higher quantile has the opposite effect [S1].
*   **Initialization:** Texture grids start at 0 and begin optimizing after 500 iterations [S1].

## Operating characteristics
*   **Time Complexity:** The routine runs every 250 iterations. The cost is proportional to the number of primitives, as each primitive's error is computed and potentially split/upscaled/downscaled [S1].
*   **Memory Complexity:** Memory usage is determined by the number of primitives and the total number of texels. For example, on the 'drjohnson' scene, the method results in 293K points and 20.8M texels, using 318MB memory [S1]. On 'bonsai', it uses 291K points and 9.0M texels, using 176MB memory [S1].
*   **Performance:** The method achieves real-time rendering speeds (e.g., 77 FPS on 'drjohnson', 35 FPS on 'bonsai') while maintaining high quality (SSIM > 0.9 on many scenes) [S1].
*   **Strengths:** Automatically adjusts model capacity, balancing geometric and textural detail. Robust to different ratios of primitive and texel budgets [S1].
*   **Limitations:** Requires careful tuning of hyperparameters ($\tau_{tr}$, $\tau_{ds}$, $\lambda_{texture}$) to achieve desired results. The indirect effects of hyperparameters can be complex [S1].

## Background
This method was introduced by Panagiotis Papantonakis, Georgios Kopanas, Frédo Durand, and George Drettakis in their work on "Content-Aware Texturing for Gaussian Splatting" [S1]. It is part of a broader effort to improve the efficiency and quality of Gaussian Splatting by managing the texture resources associated with each primitive. The method builds on the principles of adaptive density control, where the model capacity is adjusted based on reconstruction error [Parent].

## Sources
[S1] Content-Aware Texturing for Gaussian Splatting, https://doi.org/10.2312/sr.20251190, fetched 2026-10-26
