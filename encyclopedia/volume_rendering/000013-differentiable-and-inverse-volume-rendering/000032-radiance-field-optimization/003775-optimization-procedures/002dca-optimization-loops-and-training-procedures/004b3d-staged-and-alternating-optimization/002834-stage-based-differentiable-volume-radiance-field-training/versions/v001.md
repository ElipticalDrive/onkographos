# Stage-based Differentiable Volume/Radiance Field Training

Also called: Mitsuba radiance field optimization, TLIR training loop

## Overview
A stage-based optimization loop for differentiable volume or radiance field reconstruction that alternates between image-based and ray-based sampling modes, applies stochastic preconditioning with exponential alpha decay, and optionally upsamples parameters between stages [S1, unofficial]. It distinguishes itself from standard NeRF training loops by explicitly managing memory via ray batching and progressively refining the field resolution through staged upsampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training dataset (images, masks, depths, normals), camera parameters, scene geometry, integrator configuration, and hyperparameters (learning rate, stages, batch sizes).
*   **Outputs:** Optimized radiance field parameters, rendered test images, and loss metrics.

**Data Structures:**
*   `spn_state`: A dictionary managing Stochastic Preconditioning (SPN).
    *   `alpha` (float): Current preconditioning scale factor.
    *   `step` (int): Iteration counter for SPN decay.
    *   `gamma` (float): Exponential decay factor for alpha.
*   `ray_batch`: A pre-extracted batch of rays (used in ray-based mode).
*   `losses`: A list accumulating scalar loss values per iteration.

**Parameters:**
*   `use_ray_batching` (bool): Selects between image-based and ray-based sampling.
*   `rays_per_batch` (int): Number of rays sampled per iteration in ray-based mode.
*   `spp` (int): Samples per pixel for test image rendering.
*   `num_iterations_per_stage` (int): Iterations per training stage.
*   `num_stages` (int): Total number of training stages.
*   `enable_upsampling` (bool): Whether to upsample parameters between stages.
*   `learning_rate` (float): Base learning rate.
*   `stochastic_preconditioning_starting_alpha` (float): Initial alpha for SPN.
*   `stochastic_preconditioning_iterations` (int): Iterations for alpha decay.
*   `stochastic_preconditioning_update_interval` (int): Frequency of SPN state updates.

**Algorithm Steps:**

1.  **Initialization:**
    *   Set integrator loss type.
    *   Initialize RNG with seed `stage_idx * 1000`.
    *   Initialize `spn_state`:
        *   If `stochastic_preconditioning_iterations` == 0, set `alpha = 0.0` and `gamma = 0.0` to prevent division by zero [S1, unofficial].
        *   Otherwise, set `alpha = stochastic_preconditioning_starting_alpha`.
        *   Calculate `gamma = 1e-4 ** (1.0 / stochastic_preconditioning_iterations)` [S1, unofficial].
        *   Set `step = 0`.

2.  **Ray Sampling:**
    *   If `use_ray_batching` is true:
        *   Shuffle the pre-extracted `ray_batch` using the current RNG.
        *   Select `rays_per_batch` rays from the shuffled batch.
        *   Convert rays to Mitsuba tensors.
    *   If `use_ray_batching` is false:
        *   Select a random sensor index.
        *   Sample all rays from that image using `sample_rays_from_image`.
        *   Retrieve rays, target colors, and masks.

3.  **Rendering and Loss:**
    *   Create an independent sampler seeded with the iteration index and ray count.
    *   Render rays using the integrator:
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
    *   Append `loss` to `losses`.

4.  **Optimization Step:**
    *   Step the optimizer using the computed gradients.
    *   Perform post-step updates on the integrator.
    *   Update `spn_state['step']`.
    *   If `spn_state['step'] % stochastic_preconditioning_update_interval == 0`:
        *   Update `spn_state['alpha']` by multiplying with `gamma`.

5.  **Stage Completion:**
    *   After `num_iterations_per_stage` iterations:
        *   Render `num_test_images` test images using current scene state.
        *   Compute PSNR against reference images.
        *   If `enable_upsampling` is true and not the final stage:
            *   Upsample integrator parameters.
            *   Update scene state.
        *   Proceed to next stage (increment `stage_idx`, re-seed RNG).

**Pitfalls:**
*   **Division by Zero:** If `stochastic_preconditioning_iterations` is 0, `gamma` and `alpha` must be explicitly set to 0.0 [S1, unofficial].
*   **Memory Overhead:** Ray-based mode requires pre-extracting and storing a large ray batch, which may exceed memory for high-resolution datasets if `rays_per_batch` is not managed carefully.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and iterations per stage. Upsampling adds a one-time cost per stage.
*   **Memory Complexity:**
    *   Image-based mode: O(1) extra memory per iteration (rays generated on the fly).
    *   Ray-based mode: O(N) where N is the total number of rays in the pre-extracted batch [S1, unofficial].
*   **Performance:** Ray-based mode reduces memory overhead and enables better data shuffling for large datasets compared to image-based mode [S1, unofficial].
*   **Strengths:** Progressive upsampling improves convergence and detail; SPN stabilizes training; flexible sampling modes.
*   **Limitations:** Requires careful tuning of SPN parameters; upsampling may introduce artifacts if not smoothed.

## Background
Originated in the TLIR (Three-dimensional Light Field Inverse Rendering) project, implemented in Mitsuba 3 [S1, unofficial]. It addresses the need for stable training of high-resolution radiance fields by combining stochastic preconditioning with progressive resolution refinement.

## Sources
[S1] JackZhouSz/tlir, https://github.com/JackZhouSz/tlir, fetched 2026-10-27 (unofficial)
