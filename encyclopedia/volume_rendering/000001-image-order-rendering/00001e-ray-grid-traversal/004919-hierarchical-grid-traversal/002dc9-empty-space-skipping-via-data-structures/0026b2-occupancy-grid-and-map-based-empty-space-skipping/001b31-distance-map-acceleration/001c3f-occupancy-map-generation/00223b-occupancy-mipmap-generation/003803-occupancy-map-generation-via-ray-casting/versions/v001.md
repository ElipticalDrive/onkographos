# Occupancy Map Generation via Ray Casting

Also called: DVGO Voxel Count Views

## Overview
This method generates an occupancy map by counting the number of rays passing through each voxel in a 3D grid, identifying regions viewed by training cameras to initialize or refine the geometry mask [S1, unofficial]. It differs from hierarchical approaches like [Occupancy Mipmap Generation](2700-occupancy-mipmap-generation.md) by using explicit ray casting rather than JAX convolutions, and from [Occupancy Grid Update](2916-occupancy-grid-update.md) by focusing on initial generation via ray traversal statistics rather than incremental updates.

## How it works
**Inputs:**
- `rays`: Tensor of ray origins and directions.
- `world_size`: Dimensions of the 3D grid.
- `stepsize`: Distance between sample points along a ray.
- `near`, `far`: Clamping distances for ray segment intersection.
- `downrate`: Downsampling factor for rays if `irregular_shape` is false.
- `irregular_shape`: Boolean flag determining ray batching strategy.

**Outputs:**
- `count`: A zero-initialized tensor with the same shape as the density grid, accumulating ray hits.

**Data Structures:**
- `count`: 3D tensor, shape `(D, H, W)`, dtype float32.
- `ones`: A `DenseGrid` object representing a 3D grid of ones, used for gradient accumulation.

**Steps:**
1. Initialize `count` as a zero tensor with shape `world_size`.
2. Determine ray batching:
   - If `irregular_shape` is true, split rays into chunks of 10,000.
   - Otherwise, downsample rays by `downrate` along both spatial dimensions, flatten, and split into chunks of 10,000.
3. For each batch of rays:
   a. Create a `DenseGrid` named `ones` with size `world_size` and appropriate world bounds.
   b. Compute ray direction normalization vector `vec`. Replace zero components with $10^{-6}$ to avoid division by zero.
   c. Calculate intersection parameters `rate_a` and `rate_b` for the bounding box corners relative to the ray origin.
   d. Determine near and far intersection distances:
      $$ t_{min} = \max(\min(rate\_a), \min(rate\_b)) $$
      $$ t_{max} = \min(\max(rate\_a), \max(rate\_b)) $$
      Clamp $t_{min}$ and $t_{max}$ to the interval $[near, far]$.
   e. Generate sample positions `interpx` along the ray segment between $t_{min}$ and $t_{max}$ using precomputed `rng` and `stepsize`. The number of samples $N_{samples}$ is calculated as:
      $$ N_{samples} = \left\lfloor \frac{\| \text{world\_size} + 1 \|}{\text{stepsize}} \right\rfloor + 1 $$
   f. Compute 3D coordinates `rays_pts` for all sample points.
   g. Query the `ones` grid at `rays_pts` and call `backward()` to accumulate gradients, effectively marking the voxels traversed by the ray.
   h. After processing all rays, increment the `count` grid where the gradient of the `ones` grid is greater than 1.

**Parameters:**
- `far`: Set to $10^9$ to prevent rays from stopping early due to small bounding box limits [S1, unofficial].
- `N_samples`: Calculated as above.

**Edge Cases:**
- Division by zero in ray direction normalization is handled by replacing zeros with $10^{-6}$ [S1, unofficial].
- Small bounding box limits causing premature ray termination are mitigated by setting `far` to $10^9$ [S1, unofficial].

## Operating characteristics
Time complexity is proportional to the number of rays and samples per ray. Memory complexity is dominated by the `count` grid and the `ones` grid. Performance depends on the number of training cameras and the resolution of the 3D grid. Strengths include explicit visibility information. Limitations include the computational cost of ray casting for large scenes.

## Background
Originated in the context of Unbounded NeRF implementations, specifically in the `sjtuytc/UnboundedNeRFPytorch` repository [S1, unofficial]. It is used to initialize or refine geometry masks based on camera visibility.

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
