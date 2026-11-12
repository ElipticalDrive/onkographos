# Near-Far Linear Ray Generation

Also called: NeRF Near-Far Linear Ray Generation, Near-Far Disparity Linear Ray Generation

## Overview
Near-Far Linear Ray Generation generates a list of 3D sampling points along a camera ray segment defined by near and far clipping distances [S1, S2, S4, unofficial]. It solves the problem of determining where to evaluate the volume density or radiance field along a ray by providing a structured set of depths [S1, unofficial]. It differs from [Near-Middle-Far Ray Generation](6817-near-middle-far-ray-generation.md) by using uniform or disparity-based spacing across the entire near-far interval rather than splitting the ray into distinct regions, and from [Refined Ray Generation via PDF Sampling](6837-refined-ray-generation-via-pdf-sampling.md) by relying on fixed geometric spacing rather than learned probability distributions [S2, S3, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized or unnormalized, depending on implementation).
- Near distance $t_{near}$ and far distance $t_{far}$ along the ray.
- Number of samples $N$ (often denoted `num_points` or `N_samples`) [S1, S3, unofficial].
- Boolean flag `lindisp`: if `True`, samples in disparity space ($1/z$); if `False`, samples in linear depth space [S3, unofficial].
- Float `perturb` (or `jitter`): magnitude of random noise added to break sampling symmetry [S3, S4, unofficial].

**Outputs:**
- Array of 3D coordinates $\mathbf{x}_i$ for $i=0 \dots N-1$ [S1, S3, unofficial].
- Array of depth values $z_i$ (or $t_i$) corresponding to the samples [S2, S3, unofficial].

**Data Structures:**
- `rays_o`: Tensor of shape $(N_{rays}, 3)$ containing ray origins [S3, unofficial].
- `rays_d`: Tensor of shape $(N_{rays}, 3)$ containing ray directions [S3, unofficial].
- `near`: Tensor of shape $(N_{rays}, 1)$ containing near distances [S3, unofficial].
- `far`: Tensor of shape $(N_{rays}, 1)$ containing far distances [S3, unofficial].
- `z_vals`: Tensor of shape $(N_{rays}, N_{samples})$ containing sampled depths [S3, unofficial].
- `xyz_coarse_sampled`: Tensor of shape $(N_{rays}, N_{samples}, 3)$ containing 3D positions [S3, unofficial].

**Steps:**

1. **Extract Ray Parameters:**
   Decompose the input ray tensor into origin $\mathbf{o}$, direction $\mathbf{d}$, near $t_{near}$, and far $t_{far}$ [S3, S4, unofficial].

2. **Generate Normalized Depth Steps:**
   Create a linearly spaced array $z_{steps}$ of length $N$ from 0 to 1 [S3, unofficial]:
   $$ z_{steps} = \text{linspace}(0, 1, N) $$
   [S3, unofficial]

3. **Map to Depth or Disparity Space:**
   Interpolate between $t_{near}$ and $t_{far}$ based on the `lindisp` flag [S3, unofficial]:
   - If `lindisp` is `False` (linear depth):
     $$ z_{vals} = t_{near} \cdot (1 - z_{steps}) + t_{far} \cdot z_{steps} $$
     [S2, S3, S4, unofficial]
   - If `lindisp` is `True` (linear disparity):
     $$ z_{vals} = \frac{1}{\frac{1}{t_{near}} \cdot (1 - z_{steps}) + \frac{1}{t_{far}} \cdot z_{steps}} $$
     [S2, S3, S4, unofficial]
   [S2, unofficial] converts to disparity space using $tvals = 1. / (1. / near * (1 - tvals) + 1. / far * tvals)$.

4. **Expand Dimensions:**
   Expand $z_{vals}$ to match the batch dimension of rays if necessary [S3, unofficial]:
   $$ z_{vals} = \text{expand}(z_{vals}, (N_{rays}, N_{samples})) $$
   [S3, unofficial]

5. **Perturb Samples (Optional):**
   If `perturb > 0`, add uniform random noise to break sampling symmetry [S3, S4, unofficial]:
   - Compute interval midpoints:
     $$ z_{mid} = 0.5 \cdot (z_{vals}[:, :-1] + z_{vals}[:, 1:]) $$
     [S3, unofficial]
   - Define lower and upper bounds for each interval:
     $$ lower = \text{concat}(z_{vals}[:, :1], z_{mid}) $$
     $$ upper = \text{concat}(z_{mid}, z_{vals}[:, -1:]) $$
     [S3, unofficial]
   - Generate random noise and perturb:
     $$ rand = \text{rand}(z_{vals}.shape) $$
     $$ z_{vals} = lower + (upper - lower) \cdot perturb \cdot rand $$
     [S3, unofficial]
   [S4, unofficial] uses a similar perturbation logic scaled by step size.

6. **Compute 3D Coordinates:**
   Calculate the final 3D sample positions by combining ray origin, direction, and sampled depths [S3, S4, unofficial]:
   $$ \mathbf{x}_i = \mathbf{o} + \mathbf{d} \cdot z_{vals}[i] $$
   [S3, unofficial]
   In tensor form:
   $$ \mathbf{xyz} = \mathbf{o}_{unsqueeze} + \mathbf{d}_{unsqueeze} \cdot z_{vals}_{unsqueeze} $$
   [S3, S4, unofficial]

**Parameters:**
- `N_samples`: Integer, number of depth samples per ray. Default value is 64 [S3, unofficial].
- `lindisp`: Boolean, if `True` samples in disparity space, else linear depth [S3, unofficial].
- `perturb`: Float, magnitude of jitter. Default value is 1.0 [S4, unofficial]. Set to 0 for no jitter [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** When computing disparity space ($1/z$), if $t_{near}$ or $t_{far}$ is 0, division by zero occurs. A small epsilon (e.g., $1e-6$) should be added to denominators [S4, unofficial].
- **Bounding Box Clipping:** If a 3D bounding box (AABB) is provided, ray-box intersection must be computed (e.g., using DDA or slab method) to clamp $t_{near}$ and $t_{far}$ to valid ranges before sampling [S4, unofficial].
- **NDC Conversion:** In NDC space, linear depth conversion requires specific formulas involving the NDC near plane [S6, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def near_far_linear_ray_generation(rays_o, rays_d, near, far, N_samples, lindisp=False, perturb=1.0):
    """
    Generate 3D sampling points along camera rays.
    
    Args:
        rays_o: Tensor of shape (N_rays, 3), ray origins.
        rays_d: Tensor of shape (N_rays, 3), ray directions.
        near: Tensor of shape (N_rays, 1), near clipping distances.
        far: Tensor of shape (N_rays, 1), far clipping distances.
        N_samples: Integer, number of samples per ray.
        lindisp: Boolean, if True sample in disparity space.
        perturb: Float, magnitude of random jitter.
        
    Returns:
        xyz_coarse_sampled: Tensor of shape (N_rays, N_samples, 3), 3D coordinates.
        z_vals: Tensor of shape (N_rays, N_samples), sampled depths.
    """
    # Step 1: Generate normalized depth steps
    z_steps = torch.linspace(0, 1, N_samples, device=rays_o.device)
    
    # Step 2: Map to depth or disparity space
    if not lindisp:
        # Linear depth interpolation
        z_vals = near * (1 - z_steps) + far * z_steps
    else:
        # Linear disparity interpolation
        # Add epsilon to avoid division by zero
        inv_near = 1.0 / (near + 1e-6)
        inv_far = 1.0 / (far + 1e-6)
        inv_z_vals = inv_near * (1 - z_steps) + inv_far * z_steps
        z_vals = 1.0 / inv_z_vals
        
    # Step 3: Expand dimensions to match ray batch
    # z_vals is currently (N_samples,) or (1, N_samples) depending on input shape
    # Ensure it broadcasts correctly with rays_o and rays_d
    z_vals = z_vals.expand(rays_o.shape[0], N_samples)
    
    # Step 4: Perturb samples if perturb > 0
    if perturb > 0:
        # Compute midpoints
        z_vals_mid = 0.5 * (z_vals[:, :-1] + z_vals[:, 1:])
        
        # Compute upper and lower bounds for each interval
        # Concatenate midpoints with endpoints
        upper = torch.cat([z_vals_mid, z_vals[:, -1:]], dim=-1)
        lower = torch.cat([z_vals[:, :1], z_vals_mid], dim=-1)
        
        # Generate random noise and perturb
        perturb_rand = perturb * torch.rand(z_vals.shape, device=rays_o.device)
        z_vals = lower + (upper - lower) * perturb_rand
        
    # Step 5: Compute 3D coordinates
    # rays_o: (N_rays, 3), rays_d: (N_rays, 3), z_vals: (N_rays, N_samples)
    # Unsqueeze to broadcast: (N_rays, 1, 3) and (N_rays, N_samples, 1)
    rays_o_expanded = rays_o.unsqueeze(1)  # (N_rays, 1, 3)
    rays_d_expanded = rays_d.unsqueeze(1)  # (N_rays, 1, 3)
    z_vals_expanded = z_vals.unsqueeze(2)  # (N_rays, N_samples, 1)
    
    xyz_coarse_sampled = rays_o_expanded + rays_d_expanded * z_vals_expanded
    
    return xyz_coarse_sampled, z_vals
```

## Subtopics

### [Near-Far Clipping](7709-near-far-clipping.md)
Near-Far Clipping is a post-processing step in ray generation that constrains ray sampling intervals to a specified viewing volume defined by fixed global near and far distances. It ensures that samples are only generated within valid bounds, preventing rays from sampling empty space beyond the far plane or artifacts near the camera origin. Unlike [Inverse CDF Ray Sampling](11412-inverse-cdf-ray-sampling.md) which adapts sampling density to scene content, or [Near-Far Linear/Disparity Ray Generation with Stratification](11841-near-far-linear-disparity-ray-generation-with-stratification.md) which handles stratified intervals, this method applies rigid, uniform bounds to all rays.

### [Inverse CDF Ray Sampling](11412-inverse-cdf-ray-sampling.md)
Inverse CDF Ray Sampling generates sample points along a ray by mapping uniform random noise to depth intervals using the cumulative distribution function (CDF) derived from per-hit probabilities. It solves the problem of stratified sampling within non-uniform depth bins defined by ray-triangle or ray-volume intersections, ensuring that samples are distributed according to the likelihood of each interval while maintaining stratification to reduce variance. It differs from simple uniform sampling by weighting samples toward intervals with higher probability mass, and from pure stratified sampling by allowing arbitrary probability distributions rather than uniform bin weights.

### [Near-Far Linear/Disparity Ray Generation with Stratification](11841-near-far-linear-disparity-ray-generation-with-stratification.md)
Generates sample points along camera rays for volume rendering by sampling z-values (depths) either linearly or in disparity space, with optional stratification to reduce variance. It extends [Near-Far Linear Ray Generation](6841-near-far-linear-ray-generation.md) by supporting disparity-based spacing and depth-guided supersampling, distinguishing it from [Inverse CDF Ray Sampling](11412-inverse-cdf-ray-sampling.md) which uses probabilistic distributions rather than fixed geometric spacing.

### [Depth-Guided Supersampling](12190-depth-guided-supersampling.md)
Depth-Guided Supersampling generates sample positions along a ray by perturbing a depth estimate with Gaussian noise, encouraging the renderer to sample around the estimated surface location. It differs from standard linear spacing by focusing computational effort on likely geometry, and from Inverse CDF sampling by relying on an external depth prior rather than a learned density distribution.

### [Dataset-Specific Near/Far Bound Calculation](12599-dataset-specific-near-far-bound-calculation.md)
Computes the near and far clipping planes for ray sampling based on the dataset type and Normalized Device Coordinates (NDC) configuration. It ensures that the ray sampler operates within a geometrically valid range appropriate for the scene's scale and projection model, serving as the parameterization layer for [Near-Far Linear Ray Generation](6841-near-far-linear-ray-generation.md).

### [Uniform Ray Marching with Occupancy Filtering](13089-uniform-ray-marching-with-occupancy-filtering.md)
Uniform Ray Marching with Occupancy Filtering is a raymarching strategy that samples a fixed number of points along each ray globally, then filters out samples that fall outside of occupied Sparse Point Cloud (SPC) cells. It differs from its sibling [SPC Uniform Raymarching](11412-spc-uniform-raymarching.md), which calculates step sizes based on cell geometry, and [SPC Voxel Raymarching](11413-spc-voxel-raymarching.md), which samples a fixed number of points *within* each intersected cell. This method prioritizes a uniform global sampling density, using the SPC structure solely as a validity mask to discard empty-space samples.

### Near-Far Disparity Linear Ray Generation with Jitter
Generates ray samples using disparity spacing with random jitter within segments, similar to NeRF's sampling but with explicit segment length calculation [S2, unofficial]. It creates linear $tvals$ from 0 to 1, converts them to disparity space using $tvals = 1. / (1. / near * (1 - tvals) + 1. / far * tvals)$, and if jitter > 0, computes midpoints, upper/lower bounds, and random $t_{rand}$ to jitter $tvals$ within segments: $tvals = lower + (upper - lower) * t_{rand}$ [S2, unofficial]. It computes segment lengths by differencing $tvals$, appending a large value ($1e10$), and multiplying by ray direction norm, then returns ray positions as $raypos = campos + raydir * tvals$ [S2, unofficial].

### Linear Seed Line Generation
A method to generate seed points along a line segment between `m_startPt` and `m_endPt` [S5, unofficial]. It calculates `delta` as `(m_endPt - m_startPt) / count` and loops `i` from 0 to `count - 1`, setting `m_h_seeds[i]` to `m_startPt + i * delta` [S5, unofficial]. The parameter `count` specifies the number of seed points to generate [S5, unofficial].

### Near-Far Linear Ray Generation with NDC Depth Simulation
Generates samples in NDC space by initializing sample count $N_{samples}$, retrieving near/far planes, and generating linearly spaced depth values `interpx` between near and far [S6, unofficial]. If training, it adds uniform random noise to `interpx` scaled by step size $(far-near)/N_{samples}$ [S6, unofficial]. It converts linear depth to NDC depth using $ndc\_depth = 2 * ndc\_near\_plane / (1 - ndc\_near\_plane - interpx)$ and rescales to simulated Euclidean depth [S6, unofficial]. It computes ray points using either simulated or linear depth, and masks out points outside the AABB defined by `self.aabb[0]` and `self.aabb[1]` [S6, unofficial]. The parameter `ndc_near_plane` is typically 1.0 [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ for generating sample positions [S1, S3, unofficial].
- **Memory Complexity:** $O(N_{rays} \cdot N_{samples})$ to store the sampled coordinates and depths [S3, unofficial].
- **Performance:** The method is computationally inexpensive as it involves only linear interpolation and basic arithmetic operations [S3, unofficial]. Performance scales linearly with the number of rays and samples [S3, unofficial].
- **Strengths:** Simple to implement, deterministic (without jitter), and provides uniform coverage of the near-far range [S3, unofficial]. Disparity sampling is beneficial for scenes with large depth variations [S3, unofficial].
- **Limitations:** Uniform sampling may waste samples in empty space or undersample complex geometry [S3, unofficial]. It does not adapt to scene content [S2, unofficial].
- **When to Use:** When a simple, uniform sampling strategy is sufficient, or as a baseline for more advanced sampling methods [S3, unofficial].
- **When Not to Use:** When the scene has significant empty space or complex geometry that requires adaptive sampling [S3, unofficial].

## Background
Near-Far Linear Ray Generation is a fundamental technique in volume rendering and neural radiance fields (NeRF) [S2, S3, unofficial]. It was popularized by the NeRF paper (Mildenhall et al., 2020) which used linear and disparity sampling with jitter to generate coarse samples for volume rendering [S2, S3, unofficial]. The method has since been adopted in various other neural rendering frameworks, including TensoRF [S4, unofficial] and PointNeRF [S2, unofficial].

## Sources
[S1] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S3] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
[S4] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S5] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
[S6] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
