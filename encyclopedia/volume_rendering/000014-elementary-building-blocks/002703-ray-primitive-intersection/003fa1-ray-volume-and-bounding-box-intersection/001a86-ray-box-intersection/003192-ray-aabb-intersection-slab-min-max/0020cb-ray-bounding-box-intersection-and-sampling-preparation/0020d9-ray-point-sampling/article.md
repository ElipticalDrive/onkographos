# Ray Point Sampling

Also called: Points Sampling

## Overview
Ray Point Sampling generates query points along camera rays for volume rendering by combining uniform and importance sampling strategies within a defined bounding box [S1][S2, unofficial]. It differs from its sibling [Clipping Distance Computation via Volume Corners](2622-clipping-distance-computation-via-volume-corners.md) by explicitly managing the density and distribution of sample points (uniform vs. importance) rather than just determining geometric boundaries, and from [CUDA Extension Interface for Ray Sampling and Rendering](8135-cuda-extension-interface-for-ray-sampling-and-rendering.md) by focusing on the statistical sampling logic rather than the low-level kernel interface [S1][S2, unofficial].

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $(N, 3)$, contiguous [S2, unofficial].
- `rays_d`: Ray directions, shape $(N, 3)$, contiguous [S2, unofficial].
- `xyz_min`: Bounding box minimum corner, shape $(3)$ [S2, unofficial].
- `xyz_max`: Bounding box maximum corner, shape $(3)$ [S2, unofficial].
- `near`: Near clipping distance scalar [S2, unofficial].
- `far`: Far clipping distance scalar, typically $10^9$ [S2, unofficial].
- `stepsize`: Base step size scalar [S2, unofficial].
- `voxel_size`: Voxel physical size scalar [S2, unofficial].
- `is_unbounded`: Boolean flag indicating if the scene is unbounded (e.g., DTU dataset) [S1].

**Outputs:**
- `ray_pts`: Sample point coordinates, shape $(N_{total}, 3)$ [S2, unofficial].
- `ray_id`: Index of the ray each point belongs to, shape $(N_{total})$ [S2, unofficial].
- `step_id`: Index of the sample along the ray, shape $(N_{total})$ [S2, unofficial].

**Data Structures:**
- `mask_inbbox`: Boolean mask, shape $(N_{total})$, true for points inside the bounding box [S2, unofficial].
- `mask_outbbox`: Boolean mask, shape $(N_{total})$, true for points outside the bounding box [S2, unofficial].

**Parameters:**
- `num_uniform`: Number of uniform points per ray in foreground, typically 64 [S1].
- `num_importance`: Number of importance samples per ray, typically 32 [S1].
- `num_unbounded`: Number of additional points for unbounded scenes, typically 32 [S1].
- `max_batch_points`: Maximum total points in a batch, $512 \times 64 \times 32$ [S1].

**Steps:**
1.  **Preparation**: Ensure `rays_o` and `rays_d` are contiguous in memory [S2, unofficial].
2.  **Step Distance Calculation**: Compute the physical step distance:
    $$ stepdist = stepsize \times voxel\_size $$
    [S2, unofficial]
3.  **Uniform Sampling**: Generate `num_uniform` uniformly distributed points along each ray within the interval $[near, far]$ [S1][S2, unofficial].
4.  **Importance Sampling**: Generate `num_importance` additional points using importance sampling on top of the uniform samples [S1].
5.  **Unbounded Handling**: If `is_unbounded` is true, generate `num_unbounded` additional points in contracted space [S1].
6.  **CUDA Kernel Execution**: Call the CUDA kernel `sample_pts_on_rays` with inputs `rays_o`, `rays_d`, `xyz_min`, `xyz_max`, `near`, `far`, and `stepdist` [S2, unofficial]. This kernel returns:
    - `ray_pts`: All generated points [S2, unofficial].
    - `mask_outbbox`: Boolean mask for points outside the bounding box [S2, unofficial].
    - `ray_id`: Ray indices [S2, unofficial].
    - `step_id`: Step indices [S2, unofficial].
    - `N_steps`: Total number of steps [S2, unofficial].
    - `t_min`, `t_max`: Intersection intervals [S2, unofficial].
7.  **Filtering**: Compute the inside bounding box mask:
    $$ mask\_inbbox = \neg mask\_outbbox $$
    [S2, unofficial]
8.  **Masking**: Apply `mask_inbbox` to `ray_pts`, `ray_id`, and `step_id` to discard points outside the volume [S2, unofficial].
9.  **Return**: Output the filtered `ray_pts`, `ray_id`, and `step_id` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_point_sampling(rays_o, rays_d, xyz_min, xyz_max, near, far, stepsize, voxel_size, is_unbounded):
    # Ensure contiguous memory layout for CUDA kernel
    rays_o = torch.as_tensor(rays_o, dtype=torch.float32, device='cuda').contiguous()
    rays_d = torch.tensor(rays_d, dtype=torch.float32, device='cuda').contiguous()

    # Calculate physical step distance
    stepdist = stepsize * voxel_size

    # Call CUDA kernel for uniform and importance sampling
    # The kernel handles the generation of 64 uniform + 32 importance + 32 unbounded points
    ray_pts, mask_outbbox, ray_id, step_id, N_steps, t_min, t_max = \
        sample_pts_on_rays_cuda(rays_o, rays_d, xyz_min, xyz_max, near, far, stepdist)

    # Filter points inside the bounding box
    mask_inbbox = ~mask_outbbox

    # Apply mask to keep only valid points
    ray_pts = ray_pts[mask_inbbox]
    ray_id = ray_id[mask_inbbox]
    step_id = step_id[mask_inbbox]

    return ray_pts, ray_id, step_id
```

**Edge Cases and Pitfalls:**
- If the ray origin is inside the bounding box, the near bound $t_{near}$ should not be adjusted to the box entry point in fine-stage variants [S3].
- Precision issues may arise if `far` is set too large ($10^9$) relative to the scene scale, potentially affecting floating-point accuracy in intersection calculations [S2, unofficial].

## Subtopics

### Fine-stage Points Sampling
Filters out rays that do not intersect known free space and adjusts the near- and far-bounds ($t_{near}, t_{far}$) to the endpoints of the ray-box intersection [S3]. A key pitfall is that $t_{near}$ must not be adjusted if the ray origin $x_0$ is already inside the bounding box [S3]. This variant improves efficiency by skipping free space queries [S3].

### Edge Sample Retrieval
Retrieves samples lying on an edge defined by two vertices $v_1, v_2$ by filtering samples based on their distance from the ray origin [S4]. It iterates through samples $s$, calculating $dist = Dist(s.position, R.origin)$, and includes $s$ if $Dist(v_1, R.origin) \le dist \le Dist(v_2, R.origin)$ [S4]. This is used for preserving sharp features in sculpting applications [S4].

## Operating characteristics
- **Time Complexity**: Dominated by the CUDA kernel `sample_pts_on_rays` and the filtering step. The number of points scales linearly with the number of rays and the sampling density [S1][S2, unofficial].
- **Memory Complexity**: Proportional to the total number of sample points generated before filtering. The maximum batch size is limited to $512 \times 64 \times 32$ points [S1].
- **Performance**: Importance sampling adds computational overhead compared to pure uniform sampling but improves convergence for complex scenes [S1]. Unbounded scenes require additional contracted space sampling, increasing memory usage [S1].
- **Strengths**: Combines uniform coverage with importance sampling for better visual quality; handles unbounded scenes via contracted space [S1].
- **Limitations**: Requires contiguous ray data; memory usage can be high for large batches [S1][S2, unofficial].

## Background
Originating from volumetric rendering pipelines for neural radiance fields and fuzzy geometry representation [S1, S3], this method was developed to balance rendering quality and training speed. The specific implementation details, including the 64+32 point strategy and CUDA integration, are documented in recent works on volumetric surfaces and direct voxel grid optimization [S1, S3]. The reference implementation is available in the `ref-dvgo` repository [S2, unofficial].

## Sources
[S1] Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes, https://doi.org/10.1109/cvpr52734.2025.01991, fetched 2026-10-26
[S2] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S3] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S4] Detail sculpting using cubical marching squares, https://doi.org/10.1145/1152399.1152402, fetched 2026-11-06
