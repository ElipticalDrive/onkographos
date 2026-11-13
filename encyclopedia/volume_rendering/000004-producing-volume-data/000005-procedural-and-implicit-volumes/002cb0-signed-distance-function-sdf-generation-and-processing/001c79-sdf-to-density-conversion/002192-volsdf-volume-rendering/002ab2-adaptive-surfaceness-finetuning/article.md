# Adaptive Surfaceness Finetuning

Also called: DtoOModel

## Overview
Adaptive Surfaceness Finetuning is a method for refining the density representation in Signed Distance Function (SDF) based volume rendering by making the sharpness parameter $\beta$ spatially adaptive. It solves the problem where a uniform $\beta$ degrades rendering quality near fine-grained or transparent structures by allowing $\beta(\mathbf{x})$ to vary across the scene [S2]. Unlike siblings that optimize $\beta$ directly or use fixed functions, this method uses the Eikonal loss as an indicator to increase surfaceness only where the SDF constraint is violated, preventing the model from collapsing into arbitrary density predictions [S2]. It builds on [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md).

## How it works
**Inputs and Outputs:**
- Input: A trained SDF field $f(\mathbf{x})$, a coarse occupancy grid, and training rays.
- Output: An adaptive $\beta$ grid (surfaceness grid) and refined density estimates $\sigma(\mathbf{x}) = \alpha \beta(\mathbf{x}) \exp(-\beta(\mathbf{x}) |f(\mathbf{x})|)$ [S1, unofficial][S2].

**Data Structures:**
1.  **Coarse Occupancy Grid:** Binary grid initialized with AABB scale 1.0 and resolution 32. Created from the scene box [S1, unofficial].
2.  **Fine Occupancy Grid:** Generated during finetuning. Voxel size is $2.0 / 32$. Contains fine offset cube coordinates [S1, unofficial].
3.  **Surfaceness Grid ($\beta$):** A $512^3$ voxel grid storing $\beta_v$ for each voxel $v$ [S2].
4.  **Sample Triplets:** Accumulated data $(\mathbf{x}, \eta, w)$, where $\mathbf{x}$ is the sample position, $\eta$ is the SDF value, and $w$ is the weight [S2].

**Numbered Steps:**
1.  **Initialization:** Initialize the coarse occupancy grid with AABB scale 1.0 and resolution 32. Create the coarse binary grid from the scene box [S1, unofficial].
2.  **Sampler Setup:** Configure samplers: `PDFSampler` for importance sampling, `UniformSampler` for uniform sampling, `NeuSSampler` for NeuS-style sampling, and `ErrorBoundedSampler` for VolSDF sampling [S1, unofficial].
3.  **Accumulation:** Collect per-sample triplets $(\mathbf{x}, \eta, w)$ rendered during the finetuning process. Accumulate these triplets over multiple training iterations (specifically 5,000 steps) [S2].
4.  **Grid Generation (Conditional):** During training, if `step_counter > 5000` and `step_counter % 5000 == 1`, generate a fine occupancy grid. Calculate voxel size as $2.0 / 32$. Generate fine offset cube coordinates [S1, unofficial].
5.  **Partitioning:** Partition the accumulated triplets across the voxels of the surfaceness grid. Let $\Lambda_v$ be the subset of triplets associated with voxel $v$ [S2].
6.  **SDF Evaluation:** Evaluate the SDF at fine voxel positions using `occupancy_field.forward_geonetwork`. Reshape SDF results to match the coarse voxel structure [S1, unofficial].
7.  **Ray Marching (Fine Grid):** If the fine grid exists, perform ray marching using the fine binary grid with a smaller threshold ($10^{-3}$). Update ray near/far bounds by subtracting/adding 0.03 to the hit positions [S1, unofficial].
8.  **Sampling (VolSDF Mode):** If the method is 'volsdf', sample points using `ErrorBoundedSampler` with `density_fn` set to `occupancy_field.laplace_density` and `sdf_fn` set to `occupancy_field.get_sdf` [S1, unofficial].
9.  **Merging:** Merge samples from the uniform sampler (`voxel_samples`) and the importance sampler (`occupancy_samples`) using `UniSurfSampler.merge_ray_samples_in_eculidean` [S1, unofficial].
10. **Weight Computation:** For VolSDF method, query field outputs from `occupancy_field`. Compute weights and transmittance using `occupancy_samples.get_weights_and_transmitance` from the DENSITY field head [S1, unofficial].
11. **Surfaceness Update:** For each voxel $v$, calculate the weighted average Eikonal loss. If the loss is below threshold $\bar{\gamma}$, increase $\beta_v$ by a fixed increment [S2].

**Formulas:**
The density is computed using the Laplace distribution approximation [S1, unofficial]:
$$ \sigma(\mathbf{x}) = \alpha \beta(\mathbf{x}) \exp(-\beta(\mathbf{x}) |f(\mathbf{x})|) $$

The condition for increasing $\beta_v$ is [S2]:
$$ \frac{\sum_{(\mathbf{x}, \boldsymbol{\eta}, w) \in \Lambda_{v}} w \eta (\|\nabla f(\mathbf{x})\| - 1)^2}{\sum_{(\mathbf{x}, \boldsymbol{\eta}, w) \in \Lambda_{v}} w} < \bar{\gamma} $$
Where:
- $\Lambda_v$: Subset of triplets in voxel $v$.
- $w$: Weight of the sample.
- $\eta$: SDF value (used as a multiplier in the numerator).
- $\|\nabla f(\mathbf{x})\|$: Norm of the gradient of the SDF.
- $\bar{\gamma}$: Predefined threshold for increasing surfaceness.

**Parameters:**
- $\bar{\gamma} = 0.25$: Threshold for increasing surfaceness [S2].
- Increment for $\beta_v = 100$: Fixed value added to $\beta_v$ when the condition is met [S2].
- `smooth_loss_multi`: 0.005. Smoothness loss multiplier applied to surface points in the UniSurf context [S1, unofficial].
- `s3im_loss_mult`: 0.0. Multiplier for the S3IM perceptual loss [S1, unofficial].
- `s3im_kernel_size`: 4. Kernel size for the S3IM loss calculation [S1, unofficial].
- `s3im_stride`: 4. Stride for the S3IM loss calculation [S1, unofficial].
- `s3im_repeat_time`: 10. Number of times the S3IM patch is repeated [S1, unofficial].
- `s3im_patch_height`: 32. Virtual patch height for S3IM [S1, unofficial].
- `anneal_end`: 20000. The training step at which the cosine annealing for NeuS is completed [S1, unofficial].

**Pitfalls:**
- Directly optimizing $\beta(\mathbf{x})$ via gradient descent overly relaxes the SDF correctness constraint, causing $f(\mathbf{x})$ to predict arbitrary density values like original NeRF. The Eikonal loss is used as an indicator to only increase surfaceness where the model fails to reconstruct via SDF [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_surfaceness_finetuning(occupancy_field, training_rays, step_counter):
    # Step 1: Initialize coarse grid
    coarse_grid = create_binary_grid(aabb_scale=1.0, resolution=32)
    
    # Step 2: Setup samplers
    pdf_sampler = PDFSampler()
    uniform_sampler = UniformSampler()
    neus_sampler = NeuSSampler()
    error_bounded_sampler = ErrorBoundedSampler()
    
    # Step 3: Accumulate triplets over 5000 steps
    # Note: This accumulation happens over multiple iterations
    accumulated_triplets = []
    for step in range(5000):
        # Render rays and collect (x, eta, w)
        samples = render_rays(training_rays, occupancy_field, error_bounded_sampler)
        for s in samples:
            accumulated_triplets.append((s.x, s.eta, s.w))
            
    # Step 4: Generate fine grid if condition met
    fine_grid = None
    if step_counter > 5000 and step_counter % 5000 == 1:
        voxel_size = 2.0 / 32
        fine_grid = create_fine_grid(voxel_size=voxel_size)
        fine_offset_coords = generate_fine_offset_cube_coordinates()
        
    # Step 5: Partition triplets into voxels
    surfaceness_grid = initialize_beta_grid(size=512**3)
    voxel_subsets = partition_triplets(accumulated_triplets, surfaceness_grid)
    
    # Step 6: Evaluate SDF at fine positions
    if fine_grid is not None:
        sdf_values = occupancy_field.forward_geonetwork(fine_offset_coords)
        sdf_values = reshape_to_coarse_structure(sdf_values)
        
    # Step 7: Ray marching with fine grid
    if fine_grid is not None:
        ray_hits = ray_march(fine_grid, threshold=1e-3)
        update_ray_bounds(ray_hits, offset=0.03)
        
    # Step 8: Sample points (VolSDF mode)
    if method == 'volsdf':
        volsdf_samples = error_bounded_sampler.sample(
            density_fn=occupancy_field.laplace_density,
            sdf_fn=occupancy_field.get_sdf
        )
        
    # Step 9: Merge samples
    merged_samples = merge_ray_samples(
        uniform_sampler.samples,
        pdf_sampler.samples,
        merge_func=UniSurfSampler.merge_ray_samples_in_eculidean
    )
    
    # Step 10: Compute weights and transmittance
    weights, transmittance = compute_weights_and_transmittance(
        merged_samples,
        head='DENSITY'
    )
    
    # Step 11: Update surfaceness
    gamma_threshold = 0.25
    beta_increment = 100
    
    for v in surfaceness_grid.voxels:
        lambda_v = voxel_subsets[v]
        if not lambda_v:
            continue
            
        numerator = sum(w * eta * (norm_gradient(f(x)) - 1)**2 for x, eta, w in lambda_v)
        denominator = sum(w for x, eta, w in lambda_v)
        
        if denominator > 0:
            avg_eikonal = numerator / denominator
            if avg_eikonal < gamma_threshold:
                surfaceness_grid[v] += beta_increment
                
    return surfaceness_grid
```

## Subtopics

### [NeuS Weight Function](3238-neus-weight-function.md)
Uses a different weight function for density estimation. Choose when using NeuS-style sampling is preferred over VolSDF.

### [U-density Function](3695-u-density-function.md)
Uses a different density formulation. Choose when a specific U-density mapping is required.

## Operating characteristics
- **Time Complexity:** Depends on the number of voxels ($512^3$) and the number of accumulated triplets. Partitioning and Eikonal loss calculation are performed per voxel per update interval (5000 steps) [S2].
- **Memory Complexity:** Requires storage for the $512^3$ $\beta$ grid and the accumulated triplets. The coarse grid is $32^3$ and the fine grid is derived from it [S1, unofficial][S2].
- **Measured Performance:** Not given in the sources.
- **Scaling:** The method scales with the resolution of the surfaceness grid ($512^3$) and the number of training iterations [S2].
- **Strengths:** Prevents degradation of quality near fine-grained structures by adapting $\beta$ spatially. Avoids arbitrary density predictions by using Eikonal loss as a guide [S2].
- **Limitations:** Requires accumulation of triplets over multiple iterations, adding complexity to the training loop. The fixed increment and threshold may not be optimal for all scenes [S2].
- **When to use:** When rendering scenes with fine-grained or transparent structures where a uniform $\beta$ is insufficient [S2].
- **When not to use:** When computational resources are limited and the overhead of maintaining the surfaceness grid is prohibitive.

## Background
Adaptive Surfaceness Finetuning was introduced in the HybridNeRF paper (2024) [S2]. It addresses limitations in previous SDF-based methods by allowing spatial adaptation of the surfaceness parameter. The method is implemented in the SDFStudio repository [S1, unofficial].

## Sources
[S1] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S2] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
