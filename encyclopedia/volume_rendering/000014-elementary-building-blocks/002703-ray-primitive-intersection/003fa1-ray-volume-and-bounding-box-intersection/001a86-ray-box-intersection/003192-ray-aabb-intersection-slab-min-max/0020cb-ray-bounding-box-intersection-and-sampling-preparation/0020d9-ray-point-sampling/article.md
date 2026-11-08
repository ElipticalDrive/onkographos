# Ray Point Sampling


## Overview
Ray Point Sampling generates query points along camera rays for volume rendering by discretizing the ray segment between near and far clipping planes using a fixed step distance. It differs from its parent method, [Ray-Bounding Box Intersection and Sampling Preparation](2364-ray-bounding-box-intersection-and-sampling-preparation.md), by performing the actual coordinate generation and filtering of points relative to the scene's bounding box. It serves as the bridge between ray geometry and voxel grid lookups.

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $(H, W, 3)$ or $(N, 3)$, contiguous.
- `rays_d`: Ray directions, shape $(H, W, 3)$ or $(N, 3)$, contiguous.
- `xyz_min`: Bounding box minimum corner, shape $(3,)$.
- `xyz_max`: Bounding box maximum corner, shape $(3,)$.
- `near`: Near clipping distance scalar.
- `far`: Far clipping distance scalar. Default: $10^9$ [S1, unofficial].
- `stepsize`: Base step size scalar.
- `voxel_size`: Voxel grid physical size scalar.

**Outputs:**
- `ray_pts`: Sampled point coordinates, shape $(N_{total}, 3)$.
- `ray_id`: Index of the ray each point belongs to, shape $(N_{total},)$.
- `step_id`: Index of the step along the ray, shape $(N_{total},)$.

**Data Structures:**
The implementation relies on a CUDA kernel `sample_pts_on_rays` which returns intermediate tensors:
- `ray_pts`: All generated points.
- `mask_outbbox`: Boolean mask indicating points outside the bounding box.
- `ray_id`: Ray indices for all generated points.
- `step_id`: Step indices for all generated points.
- `N_steps`: Number of steps per ray.
- `t_min`, `t_max`: Intersection distances of the ray with the bounding box.

**Steps:**
1. Ensure `rays_o` and `rays_d` are contiguous in memory [S1, unofficial].
2. Calculate the physical step distance:
   $$ \text{stepdist} = \text{stepsize} \times \text{voxel\_size} $$
3. Call the CUDA kernel `sample_pts_on_rays` with inputs `rays_o`, `rays_d`, `xyz_min`, `xyz_max`, `near`, `far`, and `stepdist`. This generates points along each ray from `near` to `far` with spacing `stepdist` and computes bounding box intersection bounds [S1, unofficial].
4. Compute the mask for points inside the bounding box:
   $$ \text{mask\_inbbox} = \neg \text{mask\_outbbox} $$
5. Filter the output tensors using `mask_inbbox`:
   - `ray_pts` = `ray_pts[mask_inbbox]`
   - `ray_id` = `ray_id[mask_inbbox]`
   - `step_id` = `step_id[mask_inbbox]`
6. Return `ray_pts`, `ray_id`, `step_id` [S1, unofficial].

**Pitfalls:**
- The ray directions and origins must be contiguous to ensure correct CUDA kernel execution [S1, unofficial].
- If the ray origin is inside the bounding box, the near bound adjustment logic in variants must handle this to avoid incorrect clipping [S2].

## Subtopics

### Fine-stage Points Sampling
This variant optimizes sampling for the fine-stage of volume rendering by skipping free space. It filters out rays that do not intersect known free space (determined by a coarse density grid) and adjusts the `t(near)` and `t(far)` bounds to the exact endpoints of the ray-box intersection. A critical pitfall is that `t(near)` must not be adjusted if the ray origin $x_0$ is already inside the bounding box [S2].

## Operating characteristics
**Time Complexity:** Linear with respect to the number of rays and the number of steps per ray. The filtering step is also linear.
**Memory Complexity:** Proportional to the number of valid sample points inside the bounding box.
**Strengths:** Efficiently restricts sampling to the relevant scene volume, reducing unnecessary voxel lookups in empty space.
**Limitations:** Requires pre-computed bounding box coordinates (`xyz_min`, `xyz_max`). Performance depends heavily on the ratio of occupied to empty volume.

## Background
Originating from implementations of Direct Voxel Grid Optimization (DVGO) and its references, this method is used in neural radiance field reconstruction pipelines to efficiently sample voxel grids [S1, unofficial][S2].

## Sources
[S1] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S2] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
