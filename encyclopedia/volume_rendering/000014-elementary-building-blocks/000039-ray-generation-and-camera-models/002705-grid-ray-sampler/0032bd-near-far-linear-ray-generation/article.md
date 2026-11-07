# Near-Far Linear Ray Generation

Also called: Coarse Sampling, NeRF Near-Far Linear Ray Generation, Near-Far Disparity Linear Ray Generation, Ray Marching Sampling, Ray Marching with Linear/Disparity Sampling

## Overview
Near-Far Linear Ray Generation is a method for sampling 3D points along camera rays within a specified near and far depth range. It solves the problem of determining where to evaluate a volume or neural radiance field along a ray by distributing samples either linearly in depth or linearly in disparity (inverse depth). It sets itself apart from its siblings by providing the foundational coarse sampling strategy used in NeRF and its variants, distinguishing itself from [Near-Middle-Far Ray Generation](6817-near-middle-far-ray-generation.md) by using uniform spacing rather than a three-stage split, and from [Refined Ray Generation via PDF Sampling](6837-refined-ray-generation-via-pdf-sampling.md) by not using importance sampling weights for initial placement.

## How it works
The method generates a list of 3D sampling points along a ray segment defined by $t_{min}$ and $t_{max}$ (or near and far distances) for a given number of points [S1, S2, S3, unofficial]. It supports two primary spacing modes: linear depth and linear disparity [S2, S3, unofficial]. It may also include random jitter within segments to break symmetry and reduce aliasing [S2, S3, unofficial]. If a 3D bounding box (AABB) is provided, ray intersections are computed using the DDA slab method to constrain samples to the valid volume [S3, unofficial].

### Inputs and Outputs
**Inputs:**
*   `rays_o`: Ray origins, shape $(N, 3)$ where $N$ is the number of rays.
*   `rays_d`: Ray directions, shape $(N, 3)$.
*   `near`: Near plane distance, shape $(N,)$ or scalar.
*   `far`: Far plane distance, shape $(N,)$ or scalar.
*   `num_samples` (or `point_count`): Number of samples per ray, integer.
*   `lindisp`: Boolean flag. If `True`, use linear disparity spacing; if `False`, use linear depth spacing [S3, unofficial].
*   `perturb` (or `jitter`): Float value controlling the amount of random jitter added to sample positions. 0.0 for no jitter, 1.0 for full jitter [S2, S3, unofficial].
*   `bbox_3D` (optional): Axis-aligned bounding box for clipping [S3, unofficial].

**Outputs:**
*   `xyz_coarse_sampled`: 3D coordinates of the sampled points, shape $(N, num\_samples, 3)$.
*   `ts` (optional): The normalized or depth values associated with each sample, shape $(N, num\_samples)$.

### Data Structures
*   **Ray Batch**: Tensors for origins, directions, near, and far values.
*   **Sample Points**: A tensor of shape $(N, num\_samples, 3)$ storing the final 3D coordinates.
*   **Depth Values**: A tensor of shape $(N, num\_samples)$ storing the $z$ values (depths) before conversion to 3D.

### Steps

1.  **Decompose Rays**: Extract ray origins ($rays\_o$), directions ($rays\_d$), near distances ($near$), and far distances ($far$) [S3, unofficial].
2.  **Generate Normalized Steps**: Create a linear space of values from 0 to 1 with `num_samples` steps. Let this be $z\_steps$ [S3, unofficial].
    $$ z\_steps = \text{linspace}(0, 1, num\_samples) $$
3.  **Compute Depth Values ($z\_vals$)**:
    *   If `lindisp` is `False` (Linear Depth):
        $$ z\_vals = near \cdot (1 - z\_steps) + far \cdot z\_steps $$
        [S2, S3, unofficial]
    *   If `lindisp` is `True` (Linear Disparity):
        $$ z\_vals = \frac{1}{\frac{1}{near} \cdot (1 - z\_steps) + \frac{1}{far} \cdot z\_steps} $$
        [S2, S3, unofficial]
4.  **Apply Jitter (Optional)**:
    If `perturb` > 0:
    *   Compute midpoints between adjacent samples:
        $$ z\_vals\_mid = \frac{z\_vals[:, :-1] + z\_vals[:, 1:]}{2} $$
        [S3, unofficial]
    *   Define lower and upper bounds for each interval. For the first sample, the lower bound is $near$. For the last, the upper bound is $far$. For intermediate samples, bounds are adjacent $z\_vals$.
        [S2, S3, unofficial]
    *   Generate random noise $rand\_vals \sim U(0, 1)$ with shape $(N, num\_samples)$.
    *   Perturb the values:
        $$ z\_vals = lower + (upper - lower) \cdot perturb \cdot rand\_vals $$
        [S3, unofficial]
        *Note: [S2, unofficial] describes a slightly different jitter mechanism using segment lengths and random offsets, but [S3, unofficial] provides the standard NeRF-style perturbation logic. Both achieve similar anti-aliasing effects.*
5.  **Bounding Box Clipping (Optional)**:
    If `bbox_3D` is provided:
    *   Compute inverse directions: $inv\_ray\_d = 1.0 / (rays\_d + \epsilon)$, where $\epsilon = 10^{-6}$ [S3, unofficial].
    *   Calculate intersection intervals $t_{min}$ and $t_{max}$ for the AABB using the slab method [S3, unofficial].
    *   Clamp the ray's near and far values to the valid intersection range $[t_{min}, t_{max}]$ [S3, unofficial].
    *   Re-compute $z\_vals$ using the clamped near and far values.
6.  **Compute 3D Coordinates**:
    Convert the depth values $z\_vals$ to 3D coordinates using the ray equation:
    $$ xyz\_coarse\_sampled = rays\_o.unsqueeze(1) + rays\_d.unsqueeze(1) \cdot z\_vals.unsqueeze(2) $$
    [S3, unofficial]
    *Note: [S2, unofficial] describes an alternative implementation using normalized $t$ values and cumulative sums of segment lengths, which is mathematically equivalent but structurally different. The depth-based approach ([S3, unofficial]) is more common in modern NeRF implementations.*

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def near_far_linear_ray_generation(
    rays_o: torch.Tensor,  # (N, 3)
    rays_d: torch.Tensor,  # (N, 3)
    near: torch.Tensor,    # (N,) or scalar
    far: torch.Tensor,     # (N,) or scalar
    num_samples: int,
    lindisp: bool = False,
    perturb: float = 1.0,
    bbox_3d: torch.Tensor = None  # (2, 3) min/max corners, optional
) -> torch.Tensor:
    """
    Generates 3D sampling points along rays using near-far linear or disparity spacing.
    
    Args:
        rays_o: Ray origins.
        rays_d: Ray directions.
        near: Near plane distances.
        far: Far plane distances.
        num_samples: Number of samples per ray.
        lindisp: If True, use linear disparity sampling.
        perturb: Amount of jitter (0.0 to 1.0).
        bbox_3d: Optional AABB for clipping.
        
    Returns:
        xyz_coarse_sampled: (N, num_samples, 3)
    """
    # 1. Decompose rays
    # Ensure near and far are tensors of shape (N,)
    if near.dim() == 0:
        near = near.expand(rays_o.shape[0])
    if far.dim() == 0:
        far = far.expand(rays_o.shape[0])
        
    # 2. Generate normalized steps
    # linspace(0, 1, num_samples) creates values [0, ..., 1]
    z_steps = torch.linspace(0, 1, num_samples, device=rays_o.device) # (num_samples,)
    
    # 3. Compute depth values
    # Expand near/far to (N, 1) for broadcasting
    near_exp = near.unsqueeze(1) # (N, 1)
    far_exp = far.unsqueeze(1)   # (N, 1)
    
    if not lindisp:
        # Linear depth
        z_vals = near_exp * (1 - z_steps) + far_exp * z_steps # (N, num_samples)
    else:
        # Linear disparity
        # z_vals = 1 / (1/near * (1-z_steps) + 1/far * z_steps)
        inv_near = 1.0 / (near_exp + 1e-6)
        inv_far = 1.0 / (far_exp + 1e-6)
        z_vals = 1.0 / (inv_near * (1 - z_steps) + inv_far * z_steps)
        
    # 4. Apply Jitter
    if perturb > 0:
        # Compute midpoints
        # z_vals shape: (N, num_samples)
        # z_vals_mid shape: (N, num_samples - 1)
        z_vals_mid = (z_vals[:, :-1] + z_vals[:, 1:]) * 0.5
        
        # Define bounds
        # Lower bound: near for first sample, z_vals[:-1] for others
        # Upper bound: far for last sample, z_vals[1:] for others
        # To simplify, we can use the midpoints to define intervals
        
        # Create a tensor for lower bounds
        lower = torch.cat([near_exp, z_vals[:, :-1]], dim=1) # (N, num_samples)
        # Create a tensor for upper bounds
        upper = torch.cat([z_vals[:, 1:], far_exp], dim=1)   # (N, num_samples)
        
        # Generate random noise
        rand_vals = torch.rand_like(z_vals) # (N, num_samples)
        
        # Perturb
        z_vals = lower + (upper - lower) * perturb * rand_vals
        
    # 5. Bounding Box Clipping (Optional)
    if bbox_3d is not None:
        # DDA Slab Method
        # bbox_3d: (2, 3) -> [min_x, min_y, min_z], [max_x, max_y, max_z]
        inv_ray_d = 1.0 / (rays_d + 1e-6) # (N, 3)
        
        # Calculate t_min and t_max for each slab
        t_min = (bbox_3d[0] - rays_o.unsqueeze(1)) * inv_ray_d # (N, 3)
        t_max = (bbox_3d[1] - rays_o.unsqueeze(1)) * inv_ray_d # (N, 3)
        
        # The intersection interval is [max(t_min), min(t_max)]
        t_enter = torch.max(t_min, dim=1).values # (N,)
        t_exit = torch.min(t_max, dim=1).values  # (N,)
        
        # Clamp near and far to the valid intersection range
        # If ray doesn't intersect, t_enter > t_exit, but we assume valid rays for this method
        near = torch.clamp(near, min=t_enter, max=t_exit)
        far = torch.clamp(far, min=t_enter, max=t_exit)
        
        # Re-compute z_vals with clamped near/far
        if not lindisp:
            z_vals = near.unsqueeze(1) * (1 - z_steps) + far.unsqueeze(1) * z_steps
        else:
            inv_near = 1.0 / (near.unsqueeze(1) + 1e-6)
            inv_far = 1.0 / (far.unsqueeze(1) + 1e-6)
            z_vals = 1.0 / (inv_near * (1 - z_steps) + inv_far * z_steps)
            
        # Re-apply jitter if needed (simplified: just re-perturb within new bounds)
        if perturb > 0:
            lower = torch.cat([near.unsqueeze(1), z_vals[:, :-1]], dim=1)
            upper = torch.cat([z_vals[:, 1:], far.unsqueeze(1)], dim=1)
            rand_vals = torch.rand_like(z_vals)
            z_vals = lower + (upper - lower) * perturb * rand_vals

    # 6. Compute 3D Coordinates
    # rays_o: (N, 3), rays_d: (N, 3), z_vals: (N, num_samples)
    # Expand dimensions for broadcasting
    rays_o_exp = rays_o.unsqueeze(1) # (N, 1, 3)
    rays_d_exp = rays_d.unsqueeze(1) # (N, 1, 3)
    z_vals_exp = z_vals.unsqueeze(2) # (N, num_samples, 1)
    
    xyz_coarse_sampled = rays_o_exp + rays_d_exp * z_vals_exp # (N, num_samples, 3)
    
    return xyz_coarse_sampled
```

### Parameters
*   `num_samples`: Number of samples per ray. Typical values range from 48 to 256 for coarse sampling in NeRF [S1, S3, unofficial].
*   `perturb`: Jitter magnitude. 0.0 disables jitter; 1.0 enables full randomization within segments [S3, unofficial].
*   `lindisp`: Boolean. `False` for linear depth (standard), `True` for linear disparity (better for distant scenes) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Division by Zero**: When computing inverse disparity ($1/near$), ensure `near` is not zero. Add a small epsilon ($10^{-6}$) to denominators [S3, unofficial].
*   **Non-intersecting Rays**: If `bbox_3D` is used and a ray does not intersect the box, $t_{enter} > t_{exit}$. The clamping logic should handle this, or rays should be filtered out beforehand.
*   **Precision**: Using `float32` is standard, but `float64` may be needed for very large scenes to avoid precision loss in depth calculations [G].
*   **Jitter Symmetry**: Without jitter, samples are perfectly aligned across adjacent pixels, causing aliasing. Jitter breaks this symmetry [S2, S3, unofficial].

## Subtopics

### Near-Far Disparity Linear Ray Generation with Jitter
A variant that explicitly uses disparity spacing ($1/z$) and includes random jitter within segments. It is chosen when the scene has significant depth variation and disparity sampling provides better resolution at far distances. It differs from standard linear depth by mapping uniform steps in disparity space to non-uniform steps in depth space. Implementation involves converting linear $t$ values to disparity space before computing 3D positions [S2, unofficial].

### Near-Far Clipping
A component that clips ray samples to a specified near and far plane. It ensures samples are within the valid viewing volume. Parameters include `near_plane` and `far_plane`. It is used to prevent sampling outside the region of interest or behind the far clipping plane. It is a prerequisite for bounding box intersection logic [S4, unofficial].

### Linear Seed Line Generation
A method to generate seed points along a line segment between a start point (`m_startPt`) and an end point (`m_endPt`). It calculates a delta vector and iteratively adds it to the start point. It is used in vector field visualization and glyph generation. It differs from ray generation by operating on a fixed line segment rather than camera rays. Parameter `count` determines the number of seeds [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the number of samples. The computation is vectorizable and highly parallelizable on GPUs.
*   **Memory Complexity**: $O(N \cdot S \cdot 3)$ for storing the 3D coordinates.
*   **Performance**: Very fast, as it involves only basic arithmetic operations. It is the bottleneck only if $S$ is extremely large.
*   **Strengths**: Simple to implement, deterministic (without jitter), provides uniform coverage of the depth range.
*   **Limitations**: Uniform sampling may waste samples in empty space or undersample regions of high detail. It does not adapt to the scene content.
*   **When to Use**: As a coarse sampling stage in multi-stage rendering pipelines (e.g., NeRF).
*   **When Not to Use**: When high efficiency is required in sparse scenes (use importance sampling) or when the depth range is extremely large and disparity sampling is not appropriate.

## Background
The method originates from the NeRF (Neural Radiance Fields) paper by Mildenhall et al. (2020) [S2, S3, unofficial]. It was introduced as the standard way to sample points along rays for volume rendering with neural networks. The use of linear disparity sampling was proposed to handle scenes with large depth ranges more effectively. The jittering technique was added to reduce aliasing artifacts caused by the regular grid of samples. It has become a foundational component in many subsequent neural rendering methods, including TensoRF [S3, unofficial] and PointNeRF [S2, unofficial].

## Sources
[S1] Rintarooo/volume_rendering_plot, https://github.com/Rintarooo/volume_rendering_plot, fetched 2026-10-26 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S3] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S4] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S5] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
