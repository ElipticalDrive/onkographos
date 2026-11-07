# Stage-based Differentiable Volume/Radiance Field Training

Also called: Mitsuba radiance field optimization, TLIR training loop

## Overview
A stage-based optimization loop for differentiable volume or radiance field reconstruction that alternates between image-based and ray-based sampling modes, applies stochastic preconditioning with exponential alpha decay, and optionally upsamples parameters between stages [S1, unofficial]. It differs from standard NeRF training loops by explicitly managing memory via ray batching and progressive resolution upsampling, while using a specific stochastic preconditioning schedule to stabilize gradient updates [S1, unofficial].

## How it works
**Inputs:**
- Scene parameters (density $\sigma$, color $\mathbf{c}$, etc.) to be optimized.
- Reference images, masks, and pre-extracted ray batches.
- Configuration parameters (see below).

**Outputs:**
- Optimized scene parameters.
- Rendered test images and PSNR metrics.

**Data Structures:**
- `spn_state`: Dictionary containing:
  - `alpha` (float): Current preconditioning scale.
  - `step` (int): Iteration counter.
  - `gamma` (float): Exponential decay factor.

**Parameters:**
- `use_ray_batching` (bool): Selects between image-based and ray-based sampling modes [S1, unofficial].
- `rays_per_batch` (int): Number of rays sampled per iteration in ray-based mode [S1, unofficial].
- `spp` (int): Samples per pixel used for test image rendering [S1, unofficial].
- `num_iterations_per_stage` (int): Number of optimization iterations per training stage [S1, unofficial].
- `num_stages` (int): Total number of training stages for progressive upsampling [S1, unofficial].
- `enable_upsampling` (bool): Whether to upsample parameters between stages [S1, unofficial].
- `learning_rate` (float): Base learning rate for the optimizer [S1, unofficial].
- `stochastic_preconditioning_starting_alpha` (float): Initial alpha value for SPN decay [S1, unofficial].
- `stochastic_preconditioning_iterations` (int): Number of iterations over which alpha decays exponentially [S1, unofficial].
- `stochastic_preconditioning_update_interval` (int): Frequency of SPN state updates [S1, unofficial].

**Steps:**

1. **Initialization:**
   - Set integrator loss type.
   - Configure Stochastic Preconditioning (SPN) with `stochastic_preconditioning_starting_alpha`, `stochastic_preconditioning_iterations`, and `stochastic_preconditioning_update_interval`.
   - Initialize Random Number Generator (RNG) with seed `stage_idx * 1000` [S1, unofficial].
   - Compute `gamma`:
     - If `stochastic_preconditioning_iterations` == 0, set `gamma = 0.0` and `alpha = 0.0` to prevent division by zero [S1, unofficial].
     - Otherwise, calculate:
       $$
       \gamma = 10^{-4 \cdot \frac{1}{\text{stochastic\_preconditioning\_iterations}}}
       $$
       [S1, unofficial]

2. **Sampling Mode Selection:**
   - If `use_ray_batching` is true:
     - Shuffle the pre-extracted ray batch using the current RNG [S1, unofficial].
     - Sample `rays_per_batch` rays from the shuffled batch.
     - Convert rays to Mitsuba tensors [S1, unofficial].
   - If `use_ray_batching` is false:
     - Select a random sensor index.
     - Sample all rays from that image using `sample_rays_from_image` to obtain rays, target colors, and masks [S1, unofficial].

3. **Rendering and Gradient Computation:**
   - Retrieve current `spn_alpha` from the integrator [S1, unofficial].
   - Set AOV (Arbitrary Output Variable) loss masks if available [S1, unofficial].
   - Create an independent sampler seeded with the iteration index and ray count [S1, unofficial].
   - Render rays with automatic gradient computation:
     ```python
     rendered_colors, loss, aovs = integrator.render_rays_with_gradient(
         rays=rays_mi,
         target_colors=target_colors,
         sampler=sampler,
         scene=scene,
         spp=config.spp,
         spn_alpha=spn_alpha
     )
     ```
     [S1, unofficial]

4. **Optimization Step:**
   - Append the scalar `loss` value to the losses list [S1, unofficial].
   - Step the optimizer [S1, unofficial].
   - Perform post-step updates on the integrator [S1, unofficial].

5. **Stage Evaluation:**
   - After `num_iterations_per_stage` iterations:
     - Render `num_test_images` test images using the current scene state [S1, unofficial].
     - Compute PSNR against reference images [S1, unofficial].

6. **Upsampling (Optional):**
   - If `enable_upsampling` is true and not the final stage:
     - Upsample integrator parameters [S1, unofficial].
     - Update the scene state [S1, unofficial].
     - Proceed to the next stage [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `stochastic_preconditioning_iterations` is 0, `gamma` and `alpha` must be explicitly set to 0.0 to avoid invalid decay calculations [S1, unofficial].
- Ray batching reduces memory overhead and enables better data shuffling for large datasets compared to image-based sampling [S1, unofficial].

## Operating characteristics
- **Memory:** Ray-based mode with batching reduces memory overhead by sampling a fixed number of rays per iteration rather than processing full images, enabling training on larger datasets [S1, unofficial].
- **Performance:** Stochastic preconditioning with exponential decay helps stabilize training, particularly in early stages [S1, unofficial].
- **Scaling:** Progressive upsampling allows the model to learn coarse structures first before refining details, improving convergence [S1, unofficial].
- **Strengths:** Flexible sampling modes, memory-efficient ray batching, stable optimization via SPN.
- **Limitations:** Requires careful tuning of SPN parameters and upsampling schedule.

## Background
This method originates from the TLIR (Two-Stage Learning for Inverse Rendering) project, implemented in the Mitsuba 2 renderer [S1, unofficial]. It is designed for differentiable volume and radiance field reconstruction, leveraging Mitsuba's automatic differentiation capabilities [S1, unofficial].

## Sources
[S1] JackZhouSz/tlir, https://github.com/JackZhouSz/tlir, fetched 2026-10-27 (unofficial)
