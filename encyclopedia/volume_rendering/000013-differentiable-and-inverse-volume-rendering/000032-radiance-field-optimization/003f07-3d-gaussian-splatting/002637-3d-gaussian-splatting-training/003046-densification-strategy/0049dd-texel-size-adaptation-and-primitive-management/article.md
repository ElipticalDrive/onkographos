# Texel Size Adaptation and Primitive Management

Also called: Adaptive Texture Allocation, Primitive Management via Quantile Error

## Overview
Texel Size Adaptation and Primitive Management is a content-aware optimization routine that dynamically adjusts the resolution of per-primitive texture maps and the number of 3D Gaussian primitives during training [S1]. It solves the problem of balancing model capacity against appearance quality by allocating texels and primitives only where necessary, preventing overfitting and memory exhaustion [S1]. Unlike standard densification strategies that rely primarily on gradient norms or screen-space area, this method uses a quantile-based error threshold to drive both primitive splitting and texture upscaling/downscaling, allowing the model to automatically adjust the ratio of primitives to texels based on scene complexity [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current training iteration count, set of active 3D Gaussian primitives, per-primitive error values $E_i$, current texture resolution, current texture size per primitive ($t_{2pr}$), and hyperparameters $\tau_{tr}$, $\tau_{ds}$, $\lambda_{texture}$ [S1].
*   **Outputs:** Updated set of primitives (with potential splits), updated texture map resolutions (upscaled or downscaled) [S1].

**Data Structures:**
*   **Primitive Set:** A list of 3D Gaussians. Each primitive maintains its geometric parameters (mean, covariance, opacity) and an associated texture grid [S1].
*   **Texture Grid:** A 2D or 3D array of values associated with each primitive, representing high-frequency detail [S1]. The size of this grid is defined by $t_{2pr}$ [S1].
*   **Error Buffer:** A collection of scalar error values $E_i$ computed for each primitive, representing the reconstruction error attributable to that primitive [S1].

**Parameters:**
*   $\tau_{tr}$ (Texture Resolution Threshold): Controls the aggressiveness of primitive splitting. A lower value leads to more primitives [S1]. Starts at 64 and is progressively reduced to 32 within the first 7,000 iterations [S1].
*   $\tau_{ds}$ (Downscale Threshold): Controls the ease of texture downscaling. A higher value makes downscaling easier, leading to simpler appearance and fewer texels [S1].
*   $\lambda_{texture}$: Controls the variance of the texture maps. Higher values lead to smoother results; lower values result in fewer texels per primitive [S1].
*   $t_{2pr}$ (Texture Size Per Primitive): The number of texels allocated to a primitive. Initialized at iteration 500 to the closest power of two such that the smallest axis is 8 texels wide [S1].
*   Quantile: The top 90% of errors are considered for splitting and upscaling [S1].

**Steps:**
1.  **Initialization:** At iteration 500, initialize texture grids with zero values. Set $t_{2pr}$ for each primitive to the closest power of two such that the smallest axis is 8 texels wide [S1].
2.  **Periodic Execution:** Every 250 iterations, up to iteration 25,000, execute the following routine [S1]:
    1.  Compute the error $E_i$ for each primitive $i$ [S1].
    2.  Determine the error threshold corresponding to the top 90% of all $E_i$ values [S1].
    3.  For each primitive $i$:
        *   **Primitive Splitting:** If the current texture resolution is greater than $\tau_{tr}$ AND $E_i$ is in the top 90% of errors, split the primitive along its overflowing axes [S1].
        *   **Texture Upscaling:** If $t_{2pr} > 1$ AND $E_i$ is in the top 90% of errors, decrease the texel size (upscale the texture map) [S1].
        *   **Texture Downscaling:** Else if the downscale metric $E_d < \tau_{ds}$, increase the texel size (downscale the texture map) [S1]. Note: The condition "Else if" implies that downscaling is considered only if the upscaling condition was not met [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texel_size_adaptation_and_primitive_management(iteration, primitives, tau_tr, tau_ds, lambda_texture):
    # Initialization phase
    if iteration == 500:
        for primitive in primitives:
            primitive.texture_grid = zeros_like(primitive.texture_grid)
            # Set t2pr to closest power of two such that smallest axis is 8 texels
            primitive.t2pr = closest_power_of_two(primitive.smallest_axis / 8) * 8
            
    # Periodic execution phase
    if iteration % 250 == 0 and iteration <= 25000:
        # Compute errors for all primitives
        errors = []
        for primitive in primitives:
            errors.append(compute_error(primitive))
            
        # Determine threshold for top 90%
        threshold_90th_percentile = np.percentile(errors, 90)
        
        for i, primitive in enumerate(primitives):
            Ei = errors[i]
            
            # Primitive Splitting
            # Note: "texture resolution > tau_tr" refers to the global or per-primitive resolution metric
            # defined in the source context. Assuming a global or per-primitive resolution variable exists.
            if primitive.texture_resolution > tau_tr and Ei > threshold_90th_percentile:
                split_primitive_along_overflowing_axes(primitive)
                
            # Texture Upscaling
            if primitive.t2pr > 1 and Ei > threshold_90th_percentile:
                # Decrease texel size (Upscale)
                # Implementation detail: Use interpolation with scale factor 2
                primitive.texture_grid = interpolate(primitive.texture_grid, scale_factor=2)
                primitive.t2pr *= 2 # Assuming t2pr scales with resolution
                
            # Texture Downscaling
            # Note: The source uses "else if", implying this is mutually exclusive with upscaling in the same step
            elif primitive.downscale_metric < tau_ds:
                # Increase texel size (Downscale)
                # Implementation detail: Use interpolation with scale factor 0.5
                primitive.texture_grid = interpolate(primitive.texture_grid, scale_factor=0.5)
                primitive.t2pr /= 2 # Assuming t2pr scales with resolution
                
    # Hyperparameter scheduling
    if iteration <= 7000:
        # Progressively reduce tau_tr from 64 to 32
        tau_tr = 64 - (32 * (iteration / 7000))
    else:
        tau_tr = 32
```

**Edge Cases and Pitfalls:**
*   **Hyperparameter Interdependence:** Changing one hyperparameter can have secondary, indirect effects. For example, encouraging more primitives via a lower $\tau_{tr}$ can lead to fewer texels per primitive, as each primitive represents a more localized and simpler part of the scene [S1].
*   **Memory Management:** Failure to manage the growth of texels and primitives can lead to out-of-memory errors. The method is designed to prevent early overfitting and memory exhaustion by limiting the number of texels [S1].
*   **Initialization:** Texture grids must be initialized to zero and allowed to optimize after 500 iterations to avoid instability [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   The routine runs every 250 iterations, adding minimal overhead to the training loop [S1].
*   Memory usage is directly proportional to the number of primitives and the texture size per primitive ($t_{2pr}$) [S1].
*   Performance metrics vary by scene. On the 'drjohnson' scene (DeepBlending dataset), the default configuration uses 293K points, 20.8M texels, and 79.7M parameters, achieving 77 FPS and 318MB memory [S1]. On the 'bonsai' scene (Tanks&Temples dataset), it uses 291K points, 9.0M texels, and 44.1M parameters, achieving 35 FPS and 176MB memory [S1].

**Strengths and Limitations:**
*   **Strengths:** Automatically adjusts the allocation of model capacity (primitives vs. texels) based on scene content, leading to robust performance across different ratios of primitive and texel budgets [S1]. Prevents overfitting and memory issues [S1].
*   **Limitations:** The method is specific to the training phase and requires careful tuning of $\tau_{tr}$ and $\tau_{ds}$ to balance detail and efficiency [S1]. The "else if" logic for downscaling may prevent simultaneous optimization of splitting and downscaling in rare cases [S1].

## Background
This method was introduced by Panagiotis Papantonakis, Georgios Kopanas, Frédo Durand, and George Drettakis in their work on "Content-Aware Texturing for Gaussian Splatting" [S1]. It builds upon the 3D Gaussian Splatting framework by adding a content-aware texturing layer that adapts to the visual complexity of the scene [S1]. The approach is part of a broader effort to improve the efficiency and quality of Gaussian Splatting by decoupling geometric density from textural detail [S1].

## Sources
[S1] Content-Aware Texturing for Gaussian Splatting, https://doi.org/10.2312/sr.20251190, fetched 2026-10-26
