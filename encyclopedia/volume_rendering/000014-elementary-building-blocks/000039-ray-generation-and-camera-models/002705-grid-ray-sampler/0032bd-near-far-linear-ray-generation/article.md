# Near-Far Linear Ray Generation

Also called: NeRF Near-Far Linear Ray Generation, Near-Far Disparity Linear Ray Generation

## Overview
Near-Far Linear Ray Generation generates a list of 3D sampling points along a ray segment defined by near ($t_{min}$) and far ($t_{max}$) distances for a specified number of points [S1, S2, S3, S4, unofficial]. It solves the problem of determining where to evaluate the volume density and color fields along camera rays by providing a structured set of depths [S1, S3, unofficial]. It differs from [Near-Middle-Far Ray Generation](6817-near-middle-far-ray-generation.md) by using a single contiguous interval rather than splitting the ray into distinct near, middle, and far regions, and from [Refined Ray Generation via PDF Sampling](6837-refined-ray-generation-via-pdf-sampling.md) by using fixed geometric spacing rather than importance sampling based on probability distributions [S2, S3, S4, unofficial].

## How it works
**Inputs:**
*   `rays_o`: Ray origins, shape $(N_{rays}, 3)$ [S3, S4, unofficial].
*   `rays_d`: Ray directions, shape $(N_{rays}, 3)$ [S3, S4, unofficial].
*   `near`: Near clipping plane distance, shape $(N_{rays}, 1)$ or scalar [S3, S4, unofficial].
*   `far`: Far clipping plane distance, shape $(N_{rays}, 1)$ or scalar [S3, S4, unofficial].
*   `N_samples`: Number of depth samples per ray [S3, unofficial].
*   `lindisp`: Boolean flag. If `True`, samples are spaced linearly in disparity space ($1/z$); if `False`, they are spaced linearly in depth ($z$) [S3, S4, unofficial].
*   `perturb`: Float. Amount of uniform random noise to add to break sampling symmetry. $0$ for no jitter, $1$ for full jitter within intervals [S3, S4, unofficial].
*   `bbox_3D` (Optional): Axis-Aligned Bounding Box (AABB) defined by min and max corners. If provided, ray intervals are clipped to the intersection with this box [S4, unofficial].

**Outputs:**
*   `xyz_coarse_sampled`: 3D sample coordinates, shape $(N_{rays}, N_{samples}, 3)$ [S3, S4, unofficial].
*   `z_vals`: Sampled depths along the ray, shape $(N_{rays}, N_{samples})$ [S3, S4, unofficial].

**Data Structures:**
*   Depths are stored as floating-point tensors.
*   3D coordinates are computed on-the-fly or stored as a 3D tensor.

**Steps:**

1.  **Decompose Rays:** Extract origins, directions, near, and far values from the input ray tensor [S4, unofficial].
    $$ \text{rays}_o = \text{rays}[:, 0:3] $$
    $$ \text{rays}_d = \text{rays}[:, 3:6] $$
    $$ \text{near} = \text{rays}[:, 6:7] $$
    $$ \text{far} = \text{rays}[:, 7:8] $$
    [S3, unofficial]

2.  **Generate Normalized Steps:** Create a linearly spaced vector of normalized depths $z_{steps}$ from $0$ to $1$ with $N_{samples}$ points [S3, S4, unofficial].
    $$ z_{steps} = \text{linspace}(0, 1, N_{samples}) $$
    [S3, S4, unofficial]

3.  **Compute Depth Values ($z_{vals}$):**
    *   If `lindisp` is `False` (Linear Depth):
        $$ z_{vals} = \text{near} \cdot (1 - z_{steps}) + \text{far} \cdot z_{steps} $$
        [S3, S4, unofficial]
    *   If `lindisp` is `True` (Linear Disparity):
        $$ z_{vals} = \frac{1}{\frac{1}{\text{near}} \cdot (1 - z_{steps}) + \frac{1}{\text{far}} \cdot z_{steps}} $$
        [S3, S4, unofficial]
    *   Expand $z_{vals}$ to match the batch dimension $N_{rays}$ [S3, unofficial].

4.  **Perturb Samples (Optional):** If `perturb > 0`, add uniform random noise to break symmetry [S3, S4, unofficial].
    *   Compute interval midpoints:
        $$ z_{vals\_mid} = 0.5 \cdot (z_{vals}[:, :-1] + z_{vals}[:, 1:]) $$
        [S3, unofficial]
    *   Define interval bounds:
        $$ \text{upper} = \text{concat}(z_{vals\_mid}, z_{vals}[:, -1:]) $$
        $$ \text{lower} = \text{concat}(z_{vals}[:, :1], z_{vals\_mid}) $$
        [S3, unofficial]
    *   Generate random noise and perturb:
        $$ \text{perturb\_rand} = \text{perturb} \cdot \text{rand}(z_{vals}.shape) $$
        $$ z_{vals} = \text{lower} + (\text{upper} - \text{lower}) \cdot \text{perturb\_rand} $$
        [S3, unofficial]
    *   *Note:* Some implementations compute jitter differently, e.g., using segment lengths and random offsets within segments [S2, unofficial]. The method above is the standard NeRF approach [S3, unofficial].

5.  **Compute 3D Coordinates:**
    $$ \text{xyz\_coarse\_sampled} = \text{rays}_o[:, \text{None}, :] + \text{rays}_d[:, \text{None}, :] \cdot z_{vals}[:, :, \text{None}] $$
    [S3, S4, unofficial]

**Parameters:**
*   `N_samples`: Default 64 [S3, unofficial].
*   `lindisp`: Default `False` [S3, unofficial].
*   `perturb`: Default 1.0 (full jitter) in some implementations, 0.0 in others [S3, S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In disparity sampling, if `near` or `far` is 0, division by zero occurs. Ensure `near > 0` [S3, S4, unofficial].
*   **Precision:** When `near` and `far` differ significantly, linear depth sampling may under-sample distant regions. Disparity sampling mitigates this but can over-sample near regions [S3, unofficial].
*   **Bounding Box Clipping:** If `bbox_3D` is provided, rays that do not intersect the box should be handled (e.g., by skipping or returning empty samples) [S4, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def near_far_linear_ray_generation(rays_o, rays_d, near, far, N_samples, lindisp=False, perturb=1.0):
    """
    Generate 3D sample points along camera rays.
    
    Args:
        rays_o: Tensor of shape (N_rays, 3), ray origins.
        rays_d: Tensor of shape (N_rays, 3), ray directions.
        near: Tensor of shape (N_rays, 1) or scalar, near clipping distance.
        far: Tensor of shape (N_rays, 1) or scalar, far clipping distance.
        N_samples: int, number of samples per ray.
        lindisp: bool, if True, sample in disparity space.
        perturb: float, amount of jitter (0 to 1).
        
    Returns:
        xyz_coarse_sampled: Tensor of shape (N_rays, N_samples, 3).
        z_vals: Tensor of shape (N_rays, N_samples).
    """
    # Ensure near and far are tensors of shape (N_rays, 1)
    if near.dim() == 0:
        near = near.expand(rays_o.shape[0], 1)
    if far.dim() == 0:
        far = far.expand(rays_o.shape[0], 1)
        
    # Step 2: Generate normalized steps
    z_steps = torch.linspace(0, 1, N_samples, device=rays_o.device)
    
    # Step 3: Compute depth values
    if lindisp:
        # Linear disparity sampling
        # z_vals = 1 / (1/near * (1 - z_steps) + 1/far * z_steps)
        # Handle potential division by zero if near or far is 0
        inv_near = 1.0 / near
        inv_far = 1.0 / far
        z_vals = 1.0 / (inv_near * (1 - z_steps) + inv_far * z_steps)
    else:
        # Linear depth sampling
        z_vals = near * (1 - z_steps) + far * z_steps
        
    # Expand z_vals to match ray batch dimension
    # z_steps is (N_samples,), z_vals is (N_rays, N_samples)
    # Note: torch.linspace creates a 1D tensor. Broadcasting handles the rest if we expand correctly.
    # In the formula above, near and far are (N_rays, 1), z_steps is (N_samples,).
    # PyTorch broadcasting: (N_rays, 1) * (1, N_samples) -> (N_rays, N_samples)
    # However, z_steps needs to be reshaped or broadcasted correctly.
    # z_steps is (N_samples,). near is (N_rays, 1).
    # (N_rays, 1) * (N_samples,) -> (N_rays, N_samples) is valid in PyTorch if dimensions align.
    # Let's be explicit:
    z_steps_expanded = z_steps.unsqueeze(0).expand(rays_o.shape[0], -1) # (N_rays, N_samples)
    
    if lindisp:
        inv_near = 1.0 / near
        inv_far = 1.0 / far
        z_vals = 1.0 / (inv_near * (1 - z_steps_expanded) + inv_far * z_steps_expanded)
    else:
        z_vals = near * (1 - z_steps_expanded) + far * z_steps_expanded
        
    # Step 4: Perturb samples
    if perturb > 0:
        # Compute midpoints
        z_vals_mid = 0.5 * (z_vals[:, :-1] + z_vals[:, 1:]) # (N_rays, N_samples - 1)
        
        # Compute upper and lower bounds
        # upper: (N_rays, N_samples)
        # lower: (N_rays, N_samples)
        upper = torch.cat([z_vals_mid, z_vals[:, -1:]], dim=-1)
        lower = torch.cat([z_vals[:, :1], z_vals_mid], dim=-1)
        
        # Generate random noise
        perturb_rand = perturb * torch.rand(z_vals.shape, device=rays_o.device)
        
        # Perturb
        z_vals = lower + (upper - lower) * perturb_rand
        
    # Step 5: Compute 3D coordinates
    # rays_o: (N_rays, 3), rays_d: (N_rays, 3), z_vals: (N_rays, N_samples)
    # xyz_coarse_sampled: (N_rays, N_samples, 3)
    xyz_coarse_sampled = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(2)
    
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

### Near-Far Disparity Linear Ray Generation with Jitter
Generates ray samples using disparity spacing with random jitter within segments, similar to NeRF's sampling but with explicit segment length calculation [S2, unofficial]. It creates linear $t_{vals}$ from 0 to 1, converts them to disparity space, and applies jitter by computing midpoints, upper/lower bounds, and random offsets [S2, unofficial]. The segment lengths are computed by differencing $t_{vals}$ and multiplying by the ray direction norm [S2, unofficial].

### Linear Seed Line Generation
A method to generate seed points along a line segment between `m_startPt` and `m_endPt` [S5, unofficial]. It calculates `delta` as `(m_endPt - m_startPt) / count` and sets `m_h_seeds[i]` to `m_startPt + i * delta` for `i` from 0 to `count - 1` [S5, unofficial].

### Near-Far Linear Ray Generation with NDC Depth Simulation
Generates linearly spaced depth values between near and far planes, optionally jittering them during training [S6, unofficial]. It converts linear depth to NDC depth using `ndc_depth = 2 * ndc_near_plane / (1 - ndc_near_plane - interpx)` and rescales to simulated Euclidean depth [S6, unofficial]. It also checks if samples are outside an AABB and masks them out [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ for generating depths and 3D coordinates [S3, S4, unofficial].
*   **Memory Complexity:** $O(N_{rays} \cdot N_{samples})$ to store the sampled depths and 3D coordinates [S3, S4, unofficial].
*   **Performance:** Efficient and deterministic. The primary cost is the generation of sample points, which is typically negligible compared to the volume rendering integration step [S3, S4, unofficial].
*   **Strengths:** Simple to implement, computationally cheap, and provides a uniform coverage of the ray segment [S3, S4, unofficial].
*   **Limitations:** Does not adapt to scene content, potentially wasting samples in empty space or under-sampling complex regions [S3, S4, unofficial].
*   **When to Use:** As a baseline sampler, for coarse sampling in multi-stage rendering pipelines, or when scene complexity is low [S3, S4, unofficial].
*   **When Not to Use:** When high-frequency details are present and uniform sampling leads to aliasing, or when the scene has large empty regions [S3, S4, unofficial].

## Background
Near-Far Linear Ray Generation is a fundamental technique in volume rendering and neural radiance fields (NeRF) [S1, S2, S3, S4, unofficial]. It was popularized by the NeRF paper [S3, S4, unofficial], which introduced linear and disparity sampling with jitter to break sampling symmetry and improve reconstruction quality [S3, unofficial]. The method is widely used in various neural rendering frameworks due to its simplicity and effectiveness [S1, S2, S4, unofficial].

## Sources
[S1] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S3] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
[S4] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S5] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
[S6] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
