# EmissivityRenderer

Also called: Differentiable renderer for multi-line neural tomography

## Overview
EmissivityRenderer is a differentiable volume renderer designed for multi-line neural tomography, specifically recovering 3-D coronal electron density ($n_e$) and temperature ($T_e$) from multiview, multiline spectral-intensity images [S1, unofficial]. It performs ray marching and Line-of-Sight (LOS) integration of emissivity coefficients derived from neural network models. It is distinguished from its siblings by its specific application to solar coronal physics, utilizing a density-temperature benchmark and CHIANTI atomic data for emissivity calculation, whereas [Differential Rendering for CT Reconstruction](6462-differential-rendering-for-ct-reconstruction.md) focuses on general differential CT and [DiffVox](10071-diffvox.md) on voxel-based differentiable rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins and directions (defined by the parent [Differentiable CT Execution Model](3066-differentiable-ct-execution-model.md)), occupancy grid parameters, neural network weights for density and temperature prediction, and physical constants/ion data for emissivity lookup.
*   **Outputs:** Differentiable LOS-integrated intensity images for multiple spectral lines, gradients with respect to network weights and input parameters.

**Data Structures:**
*   **Occupancy Grid:** A hash grid of resolution `occ_res` (default 128) storing occupancy probabilities. Updated via atomic operations during training [S1, unofficial].
*   **DensityMLP:** A neural network mapping spatial coordinates to electron density $n_e$.
*   **Temperature Model:** A neural network or function mapping spatial coordinates to temperature $T_e$.
*   **Emissivity LUT:** A precomputed lookup table for the emissivity coefficient $c_{coef}$ as a function of $n_e$, $T_e$, and radial distance $r$, derived from CHIANTI atomic data [S1, unofficial].

**Parameters:**
*   `occ_warmup`: Number of warmup steps for occupancy grid training. Default: 256 [S1, unofficial].
*   `occ_res`: Resolution of the occupancy grid. Default: 128 [S1, unofficial].
*   `step_size`: Step size for ray marching. Default: 1/256 [S1, unofficial].
*   `occ_decay`: Decay rate for occupancy grid updates. Range: $(0, 1)$. Default: 0.01 [S1, unofficial].
*   `occ_alpha`: Alpha parameter for occupancy grid updates. Range: $[1, 2]$. Default: 1.5 [S1, unofficial].
*   `sigma_scale`: Scale factor for $\sigma$ proportional to $n_e^\alpha$ for importance sampling. Initialized as a learnable parameter [S1, unofficial].
*   `occ_eval_fn_ne_width`: Width parameter for the occupancy evaluation function related to electron density. Default: 0.25 [S1, unofficial].
*   `occ_eval_fn_ne_mid`: Midpoint parameter for the occupancy evaluation function related to electron density. Default: 7.25 [S1, unofficial].
*   `min_samples_per_ray`: Minimum number of samples per ray. Default: 64 [S1, unofficial].
*   `fallback_when_samples`: Threshold for triggering fallback when sample count is low. Default: 8 [S1, unofficial].

**Algorithm Steps:**
For each ray, the renderer executes the following sequence [S1, unofficial]:
1.  **Sampling:** Sample points $x_i$ along the ray within the axis-aligned bounding box ($x_{AABB}$). The number of samples is determined by `step_size` and `min_samples_per_ray`. Importance sampling may be used based on $\sigma \propto n_e^\alpha$ scaled by `sigma_scale` [S1, unofficial].
2.  **Temperature Estimation:** Compute the temperature $T(x_i)$ at each sample point using the temperature model.
3.  **Radial Distance:** Compute the radial distance $r_{xyz}$ from the center of the volume (or sun center) for each sample point $x_i$.
4.  **Density Estimation:** Compute the electron density $n_e$ at each sample point using the `DensityMLP` model.
5.  **Emissivity Coefficient:** Compute the emissivity coefficient $c_{coef}(n_e, T, r)$ using the precomputed LUT or analytical function based on CHIANTI data for the specific ion species and wavelengths [S1, unofficial].
6.  **LOS Integration:** Perform the Line-of-Sight integral of $c_{coef}$ along the ray to obtain the final intensity. This is typically done using a weighted sum: $I = \sum_i c_{coef}(i) \cdot \Delta s_i$, where $\Delta s_i$ is the step size. Gradients are backpropagated through this sum.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, density_mlp, temp_model, occupancy_grid, params):
    # 1. Sample points in AABB
    # Determine sample points along the ray within the bounding box
    # Use importance sampling if sigma_scale is active
    num_samples = max(params['min_samples_per_ray'], calculate_num_samples(ray_direction, params['step_size']))
    t_values = sample_t_values(ray_origin, ray_direction, num_samples, params)
    points = ray_origin + t_values[:, None] * ray_direction

    # 2. Compute Temperature
    temperatures = temp_model(points)

    # 3. Compute Radial Distance
    # Assuming center is at origin for simplicity, or adjust based on scene
    radial_distances = np.linalg.norm(points, axis=1)

    # 4. Compute Electron Density
    # Check occupancy if needed, but CoroNeRF uses hash grid for training updates
    densities = density_mlp(points)

    # 5. Compute Emissivity Coefficient
    # c_coef depends on n_e, T, and r. 
    # In practice, this might use a LUT or a physics-based formula.
    # Here we assume a function get_ccoef that handles the physics/LUT lookup.
    emissivities = get_ccoef(densities, temperatures, radial_distances, params)

    # 6. LOS Integral
    # Simple summation with step size
    step_size = params['step_size']
    intensity = np.sum(emissivities * step_size)
    
    return intensity
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays. The neural network evaluations dominate the per-sample cost.
*   **Memory Complexity:** Dominated by the occupancy grid and the neural network weights. The occupancy grid size is $2^{occ\_res \times 3}$ entries in the worst case, but hash grids are sparse.
*   **Performance:** Performance depends on the resolution of the occupancy grid and the complexity of the neural networks. GPU acceleration is used for hash grid updates and neural network inference.
*   **Strengths:** Differentiable end-to-end, allowing for gradient-based optimization of the 3D density and temperature fields from 2D images. Specific to solar coronal physics, leveraging accurate atomic data.
*   **Limitations:** Not bitwise-deterministic due to atomic scatter in hash grid updates and LOS accumulation on the GPU [S1, unofficial]. Requires careful tuning of occupancy grid parameters.

## Background
EmissivityRenderer is part of the CoroNeRF project, which applies neural radiance fields (NeRF) to solar coronal tomography [S1, unofficial]. It was developed to address the challenge of reconstructing 3D coronal structures from limited multiview observations. The method builds on the [Differentiable CT Execution Model](3066-differentiable-ct-execution-model.md) framework [S1, unofficial].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
