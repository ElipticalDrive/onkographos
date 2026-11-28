# Iterative Adaptive Sampling Rendering


## Overview
Iterative Adaptive Sampling Rendering is an image-order volume rendering technique that accelerates ray marching by processing rays in dynamic batches and terminating rays that have accumulated sufficient opacity. Unlike fixed-step marching, it adapts the number of samples per ray based on the ratio of alive rays to total rays, ensuring computational resources are concentrated on rays that are still interacting with the volume. It is distinct from [Resolution Adaptation](3063-resolution-adaptation.md) because it maintains a fixed spatial resolution while varying the temporal sampling density along the ray depth.

## How it works
**Inputs:**
- Ray origins and directions.
- A density/rgb function `rgb_sigma_fn` that queries the volume.
- `max_samples`: Maximum total samples per ray.
- `alpha_thre`: Threshold to filter insignificant samples.
- `early_stop_eps`: Epsilon for early stopping based on opacity.
- `cone_angle`: Parameter determining minimum samples per step.

**Outputs:**
- Final RGB image tensor.
- Accumulated depth and opacity tensors.

**Data Structures:**
- `opacity`: Tensor of shape `(num_rays)`, initialized to 0. Stores accumulated opacity per ray.
- `depth`: Tensor of shape `(num_rays)`, initialized to 0. Stores accumulated depth.
- `rgb`: Tensor of shape `(num_rays, 3)`, initialized to 0. Stores accumulated color.
- `ray_mask`: Boolean tensor of shape `(num_rays)`, initialized to `True`. Indicates rays that are not yet fully opaque.
- `RayIntervals`: A structure containing `vals` (edge positions), `is_left`/`is_right` (edge types), and optional `packed_info` or `ray_indices` for flattened batched data [S1, unofficial].

**Algorithm:**

1. **Initialization:**
   - Set `opacity`, `depth`, and `rgb` to zero tensors.
   - Set `ray_mask` to all `True`.
   - Determine `min_samples`: If `cone_angle` is 0, `min_samples = 1`; otherwise, `min_samples = 4` [S1, unofficial].

2. **Iterative Loop:**
   - While `iter_samples < max_samples`:
     - Calculate `n_alive` as the count of `True` values in `ray_mask`.
     - If `n_alive == 0`, break loop.
     - Calculate `n_samples` for this batch:
       $$ n\_samples = \max(\min(\lfloor \frac{num\_rays}{n\_alive} \rfloor, 64), min\_samples) $$
       [S1, unofficial]
     - Call `traverse_grids` to obtain `RayIntervals` for the current batch of rays. This function returns intervals where the ray intersects the volume geometry.
     - Extract `t_starts` and `t_ends` from `intervals.is_left` and `intervals.is_right` respectively.
     - Query `rgbs` and `sigmas` (density) using `rgb_sigma_fn` at the interval endpoints or midpoints.
     - Compute transmittance weights and alphas:
       $$ \alpha_i = 1 - \exp(-\sigma_i \cdot \Delta t_i) $$
       $$ w_i = \alpha_i \cdot \prod_{j < i} (1 - \alpha_j) $$
       where $\Delta t_i$ is the interval length. The prefix transmittance is derived from the current `opacity` state.
     - **Filtering:** If `alpha_thre > 0`, discard samples where `alpha_i < alpha_thre` to reduce noise and computation.
     - **Accumulation:**
       $$ rgb_{new} = rgb_{old} + w_i \cdot rgb_i $$
       $$ opacity_{new} = opacity_{old} + w_i $$
       $$ depth_{new} = depth_{old} + w_i \cdot t_i $$
       Update `opacity` tensor.
     - **Update Mask:**
       - Update `ray_mask` by setting entries to `False` if `opacity >= 1 - early_stop_eps`.
       - Increment `iter_samples` by `n_samples`.

3. **Termination:**
   - Return `rgb`, `depth`, and `opacity`.

**Parameters:**
- `max_samples`: Controls the upper bound of sampling. Not given in the sources.
- `early_stop_eps`: Default $10^{-4}$ [S1, unofficial].
- `alpha_thre`: Threshold for sample filtering. Not given in the sources.

**Edge Cases:**
- If `n_alive` becomes 0 before `max_samples`, the loop terminates early.
- If `cone_angle` is 0, the minimum sampling rate is reduced to 1 to handle degenerate or thin structures.

## Subtopics

### [Uniforms::Iteration](19913-uniforms-iteration.md)
A utility for synchronizing shader uniforms (resolution, step counts) with the iteration count, distinct from the core sampling logic.

## Operating characteristics
- **Time Complexity:** Proportional to the number of alive rays times the batch size. The batch size adapts to keep the workload balanced, typically capping at 64 samples per ray per iteration [S1, unofficial].
- **Memory Complexity:** Linear with respect to the number of rays for the accumulation tensors (`opacity`, `depth`, `rgb`).
- **Strengths:** Efficiently skips empty space by terminating rays early. Adapts to varying density by adjusting batch sizes.
- **Limitations:** Requires multiple passes over the ray data. Performance depends heavily on the efficiency of `traverse_grids`.
- **When to use:** High-density volumes where many rays terminate early.
- **When not to use:** Very sparse volumes where most rays traverse the entire depth, as the overhead of managing alive rays may outweigh benefits.

## Background
This method is implemented in the `taylor-nerf` repository as `render_image_with_occgrid_test` [S1, unofficial]. It represents a practical implementation of iterative adaptive sampling for neural radiance fields and voxel-based volumes, focusing on GPU-efficient batched processing.

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
