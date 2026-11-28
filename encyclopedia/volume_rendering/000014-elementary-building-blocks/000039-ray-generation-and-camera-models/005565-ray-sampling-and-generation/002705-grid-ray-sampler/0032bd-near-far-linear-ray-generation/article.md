# Near-Far Linear Ray Generation

Also called: NeRF Near-Far Linear Ray Generation, Near-Far Disparity Linear Ray Generation

## Overview
Near-Far Linear Ray Generation generates a list of 3D sampling points along a camera ray segment defined by near ($t_{min}$) and far ($t_{max}$) distances for a specified number of samples [S1, S4, unofficial]. It sets itself apart from its siblings by providing the foundational uniform or disparity-based spacing logic upon which stratified, probabilistic, or depth-guided variants build [S2, S3, S4, unofficial]. Unlike [Inverse CDF Ray Sampling](11412-inverse-cdf-ray-sampling.md) which adapts density to learned probabilities, or [Depth-Guided Supersampling](12190-depth-guided-supersampling.md) which uses external depth priors, this method relies on fixed geometric intervals [S2, S3, S4, unofficial].

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $(N_{rays}, 3)$ [S3, S4, unofficial].
- `rays_d`: Ray directions, shape $(N_{rays}, 3)$ [S3, S4, unofficial].
- `near`: Near plane distances, shape $(N_{rays}, 1)$ [S3, S4, unofficial].
- `far`: Far plane distances, shape $(N_{rays}, 1)$ [S3, S4, unofficial].
- `N_samples`: Integer, number of depth samples per ray [S3, unofficial].
- `lindisp`: Boolean. If `True`, samples in disparity space ($1/z$); if `False`, samples in linear depth space [S3, S4, unofficial].
- `perturb`: Float. Amount of random jitter to add to samples (0.0 for no jitter) [S3, S4, unofficial].
- `bbox_3D`: Optional 3D Axis-Aligned Bounding Box (AABB) for clipping [S4, unofficial].

**Outputs:**
- `xyz_coarse_sampled`: 3D coordinates of sampled points, shape $(N_{rays}, N_{samples}, 3)$ [S3, S4, unofficial].

**Data Structures:**
- Tensors for ray parameters and sampled depths are used, typically standard floating-point formats [S3, S4, unofficial].

**Steps:**

1. **Decompose Ray Parameters:** Extract origins, directions, near, and far values from the input tensor [S4, unofficial].
   $$ \text{rays}_o, \text{rays}_d = \text{rays}[:, 0:3], \text{rays}[:, 3:6] $$
   $$ \text{near}, \text{far} = \text{rays}[:, 6:7], \text{rays}[:, 7:8] $$ [S3, unofficial]

2. **Generate Normalized Depth Steps:** Create a linearly spaced vector from 0 to 1 with $N_{samples}$ steps [S3, S4, unofficial].
   $$ z_{steps} = \text{linspace}(0, 1, N_{samples}) $$ [S3, S4, unofficial]

3. **Compute Depth Values ($z_{vals}$):**
   - If `lindisp` is `False` (linear depth):
     $$ z_{vals} = \text{near} \cdot (1 - z_{steps}) + \text{far} \cdot z_{steps} $$ [S3, S4, unofficial]
   - If `lindisp` is `True` (linear disparity):
     $$ z_{vals} = \frac{1}{\frac{1}{\text{near}} \cdot (1 - z_{steps}) + \frac{1}{\text{far}} \cdot z_{steps}} $$ [S3, S4, unofficial]

4. **Expand Dimensions:** Expand $z_{vals}$ to match the batch dimension of rays [S3, unofficial].
   $$ z_{vals} = \text{expand}(z_{vals}, (N_{rays}, N_{samples})) $$ [S3, unofficial]

5. **Apply Bounding Box Clipping (Optional):** If `bbox_3D` is provided, compute intersection intervals using the slab method (DDA) and clamp $z_{vals}$ to valid ranges [S4, unofficial].
   - Compute inverse direction: $\text{inv\_rays\_d} = 1.0 / (\text{rays}_d + \epsilon)$ [S4, unofficial].
   - Calculate $t_{min}, t_{max}$ for the AABB and clamp to $[\text{near}, \text{far}]$ [S4, unofficial].
   - *Note: If clipping alters the valid range, $z_{vals}$ must be re-sampled or interpolated within the new bounds. The sources describe clipping as a constraint step but do not provide the explicit re-sampling formula for clipped rays in the main linear generation block [S4, unofficial].*

6. **Perturb Samples (Optional):** If `perturb > 0`, add uniform random noise to break sampling symmetry [S3, S4, unofficial].
   - Compute interval midpoints:
     $$ z_{vals\_mid} = 0.5 \cdot (z_{vals}[:, :-1] + z_{vals}[:, 1:]) $$ [S3, unofficial]
   - Define interval bounds:
     $$ \text{upper} = \text{concat}([z_{vals\_mid}, z_{vals}[:, -1:]], \text{dim}=-1) $$
     $$ \text{lower} = \text{concat}([z_{vals}[:, :1], z_{vals\_mid}], \text{dim}=-1) $$ [S3, unofficial]
   - Generate random noise:
     $$ \text{perturb\_rand} = \text{perturb} \cdot \text{rand}(z_{vals}.shape) $$ [S3, unofficial]
   - Apply jitter:
     $$ z_{vals} = \text{lower} + (\text{upper} - \text{lower}) \cdot \text{perturb\_rand} $$ [S3, unofficial]

7. **Compute 3D Coordinates:** Combine ray origin, direction, and sampled depths [S3, S4, unofficial].
   $$ \text{xyz\_coarse\_sampled} = \text{rays}_o.\text{unsqueeze}(1) + \text{rays}_d.\text{unsqueeze}(1) \cdot z_{vals}.\text{unsqueeze}(2) $$ [S3, S4, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def near_far_linear_ray_generation(rays, N_samples, lindisp=False, perturb=1.0, bbox_3d=None):
    """
    Generate 3D sampling points along camera rays.
    
    Args:
        rays: Tensor of shape (N_rays, 8). 
              Columns 0-2: origin (x,y,z), 
              Columns 3-5: direction (x,y,z), 
              Column 6: near distance, 
              Column 7: far distance.
        N_samples: int, number of samples per ray.
        lindisp: bool, if True sample in disparity space.
        perturb: float, amount of jitter (0.0 to disable).
        bbox_3d: tuple (min_xyz, max_xyz) or None.
        
    Returns:
        xyz_coarse_sampled: Tensor of shape (N_rays, N_samples, 3).
    """
    # 1. Decompose ray parameters
    rays_o = rays[:, 0:3]  # (N_rays, 3)
    rays_d = rays[:, 3:6]  # (N_rays, 3)
    near = rays[:, 6:7]    # (N_rays, 1)
    far = rays[:, 7:8]     # (N_rays, 1)
    
    # 2. Generate normalized depth steps
    z_steps = torch.linspace(0, 1, N_samples, device=rays.device) # (N_samples,)
    
    # 3. Compute depth values
    if not lindisp:
        # Linear depth
        z_vals = near * (1 - z_steps) + far * z_steps
    else:
        # Linear disparity
        z_vals = 1.0 / (1.0 / near * (1 - z_steps) + 1.0 / far * z_steps)
        
    # 4. Apply Bounding Box Clipping (Optional)
    # The sources describe AABB clipping via DDA but do not provide the full re-sampling logic for clipped rays in this specific function [S4, unofficial].
    if bbox_3d is not None:
        inv_rays_d = 1.0 / (rays_d + 1e-6)
        # Standard slab method to compute t_min/t_max for AABB intersection would go here
        # z_vals would then be clamped to the valid range [t_min, t_max]
        
    # 5. Perturb samples
    if perturb > 0:
        # Compute midpoints
        z_vals_mid = 0.5 * (z_vals[:, :-1] + z_vals[:, 1:]) # (N_rays, N_samples - 1)
        
        # Compute upper and lower bounds
        upper = torch.cat([z_vals_mid, z_vals[:, -1:]], dim=-1) # (N_rays, N_samples)
        lower = torch.cat([z_vals[:, :1], z_vals_mid], dim=-1) # (N_rays, N_samples)
        
        # Generate random noise
        perturb_rand = perturb * torch.rand(z_vals.shape, device=rays.device)
        
        # Apply jitter
        z_vals = lower + (upper - lower) * perturb_rand
        
    # 6. Compute 3D coordinates
    xyz_coarse_sampled = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(2)
    
    return xyz_coarse_sampled
```

**Parameters:**
- `N_samples`: Default 64 [S3, unofficial].
- `lindisp`: Default `False` (linear depth) [S3, S4, unofficial].
- `perturb`: Default 1.0 (full jitter) [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** When computing disparity ($1/z$), if `near` is 0, division by zero occurs. Add a small epsilon ($\epsilon$) to denominators [G].
- **Precision:** Floating-point precision issues may arise when `near` and `far` differ significantly in magnitude, especially in disparity space [S3, unofficial].
- **Boundary Conditions:** If `bbox_3D` is used, rays that do not intersect the box must be handled (e.g., by discarding or clamping to the box boundaries) [S4, unofficial].

## Subtopics

### [Near-Far Clipping](7709-near-far-clipping.md)
Constrains ray sampling intervals to a specified viewing volume defined by fixed global near and far distances. Ensures samples are only generated within valid bounds, preventing rays from sampling empty space beyond the far plane or artifacts near the camera origin. Applies rigid, uniform bounds to all rays. Choose when rigid viewing volume constraints are required.

### [Inverse CDF Ray Sampling](11412-inverse-cdf-ray-sampling.md)
Samples points along rays using inverse CDF sampling based on probability distributions associated with depth intervals. Maps uniform random noise to depth intervals using the cumulative distribution function derived from per-hit probabilities. Replaces uniform spacing with probability-weighted stratification. Choose when adaptive density based on learned scene probabilities is needed.

### [Near-Far Linear/Disparity Ray Generation with Stratification](11841-near-far-linear-disparity-ray-generation-with-stratification.md)
Generates sample points along camera rays by sampling z-values (depths) either linearly or in disparity space, with optional stratification to reduce variance. Extends the base method by supporting disparity-based spacing and depth-guided supersampling. Choose when variance reduction via stratification is desired.

### [Depth-Guided Supersampling](12190-depth-guided-supersampling.md)
Generates sample positions along a ray by perturbing a depth estimate with Gaussian noise, encouraging the renderer to sample around the estimated surface location. Relies on an external depth prior rather than a learned density distribution. Choose when external depth priors are available.

### [Dataset-Specific Near/Far Bound Calculation](12599-dataset-specific-near-far-bound-calculation.md)
Computes the near and far clipping planes for ray sampling based on the dataset type and Normalized Device Coordinates (NDC) configuration. Ensures that the ray sampler operates within a geometrically valid range appropriate for the scene's scale and projection model. Choose when dataset-specific parameterization is required.

### [Uniform Ray Marching with Occupancy Filtering](13089-uniform-ray-marching-with-occupancy-filtering.md)
Samples a fixed number of points along each ray globally, then filters out samples that fall outside of occupied Sparse Point Cloud (SPC) cells. Prioritizes a uniform global sampling density, using the SPC structure solely as a validity mask. Choose when uniform global density is prioritized over cell-specific sampling.

### [Near-Far Linear Ray Generation with Stratified Sampling](14590-near-far-linear-ray-generation-with-stratified-sampling.md)
Generates ray samples along a near-far segment by dividing the depth range into equal intervals and drawing one uniform random sample from each interval, ensuring low-discrepancy coverage. Guarantees that every sub-interval contributes exactly one sample, reducing variance. Choose when low-discrepancy coverage is required.

### Near-Far Disparity Linear Ray Generation with Jitter
Generates ray samples using disparity spacing with random jitter within segments, similar to NeRF's sampling but with explicit segment length calculation [S2, unofficial]. It creates linear $t_{vals}$ from 0 to 1 with `point_count` steps, converts them to disparity space using $t_{vals} = 1. / (1. / \text{near} \cdot (1 - t_{vals}) + 1. / \text{far} \cdot t_{vals})$, and applies jitter if `jitter > 0` by computing midpoints, upper/lower bounds, and random $t_{rand}$ to jitter $t_{vals}$ [S2, unofficial]. Segment lengths are computed by differencing $t_{vals}$, appending a large value ($1e10$), and multiplying by ray direction norm [S2, unofficial]. Ray positions are computed directly from $t_{vals}$ or middle-point $t$s as $\text{raypos} = \text{campos} + \text{raydir} \cdot t_{vals}$ [S2, unofficial].

### Linear Seed Line Generation
A method to generate seed points along a line segment between `m_startPt` and `m_endPt` [S5, unofficial]. It calculates `delta` as $(\text{m\_endPt} - \text{m\_startPt}) / \text{count}$ and loops $i$ from 0 to `count - 1`, setting $\text{m\_h\_seeds}[i] = \text{m\_startPt} + i \cdot \text{delta}$ [S5, unofficial]. The parameter `count` specifies the number of seed points to generate [S5, unofficial].

### Near-Far Linear Ray Generation with NDC Depth Simulation
Generates samples in NDC space by initializing sample count $N_{samples}$, retrieving near/far clipping planes, and generating linearly spaced depth values `interpx` between near and far [S6, unofficial]. If training, uniform random noise is added to `interpx` scaled by step size $(\text{far}-\text{near})/N_{samples}$ [S6, unofficial]. Linear depth is converted to NDC depth using $\text{ndc\_depth} = 2 \cdot \text{ndc\_near\_plane} / (1 - \text{ndc\_near\_plane} - \text{interpx})$ [S6, unofficial]. The depth is then rescaled to simulated Euclidean depth using `util.grad_rescale(interpx, ndc_depth)` [S6, unofficial]. Ray points are computed using either simulated or linear depth, and a mask is generated to check if points are outside the AABB defined by `self.aabb[0]` and `self.aabb[1]` [S6, unofficial]. The parameter `ndc_near_plane` is typically 1.0 [S6, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ for generating depth values and 3D coordinates [S3, S4, unofficial].
- **Memory Complexity:** $O(N_{rays} \cdot N_{samples})$ to store the sampled points [S3, S4, unofficial].
- **Scaling:** Scales linearly with the number of rays and samples [S3, S4, unofficial].
- **Strengths:** Simple to implement; provides uniform coverage of the ray segment; disparity sampling is beneficial for scenes with large depth variations [S3, S4, unofficial].
- **Limitations:** Uniform sampling may waste samples in empty space; disparity sampling can introduce precision issues near the camera [S3, S4, unofficial].
- **When to use:** When a simple, uniform sampling strategy is sufficient [S3, S4, unofficial].
- **When not to use:** When the scene has complex geometry requiring adaptive sampling; when computational resources are limited and stratified or importance sampling is preferred [S3, S4, unofficial].

## Background
Near-Far Linear Ray Generation is a fundamental technique in volume rendering and neural radiance fields (NeRF) for discretizing camera rays into sample points [S1, S4, unofficial]. It was popularized by the NeRF paper [S3, unofficial] and is widely used in various implementations such as TensoRF [S4, unofficial] and PointNeRF [S2, unofficial]. The method allows for efficient evaluation of neural networks along rays to render images or extract 3D structures [S1, S3, S4, unofficial].

## Sources
[S1] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S3] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
[S4] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S5] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
[S6] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
