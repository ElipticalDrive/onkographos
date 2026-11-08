# Uniform Ray Marching

Also called: Fixed-Step Ray Marching Setup, Normalized Step Size Calculation, Ray-to-Sample Generation, Simple ray marching (debug view), Uniform Ray Marching for Visualization

## Overview
Uniform Ray Marching generates 3D sample positions along rays within an Axis-Aligned Bounding Box (AABB) using a fixed spacing between samples [S2, S3, unofficial]. It serves as a fallback strategy for environments lacking adaptive libraries (like `nerfacc`) or as a debug mode that treats an atlas texture as the full volume without brick traversal [S1, S3, unofficial]. It replaces coarse sampling to avoid aliasing artifacts, such as concentric "sampling rings," caused by steep gradients near inner radius cuts [S3, unofficial].

## How it works
**Inputs:**
*   `rays_o`: Tensor of shape `(N, 3)` containing ray origins [S3, unofficial].
*   `rays_d`: Tensor of shape `(N, 3)` containing ray directions [S3, unofficial].
*   `aabb_min`, `aabb_max`: Vectors defining the AABB boundaries [S3, unofficial].
*   `render_step_size`: Scalar float defining the fixed distance between sample points [S3, unofficial].
*   `r_max_rsun`: Optional float specifying an outer sphere radius in solar radii ($R_{sun}$). If `None`, no outer sphere clipping is applied [S3, unofficial].
*   `volumeTexture`: The texture or volume data to sample [S1, unofficial].
*   `normalizedSize`: The size of the normalized coordinate system (typically 1.0 or derived from aspect ratio normalization) [S1, unofficial].

**Outputs:**
*   `ray_idx`: Tensor of shape `(M,)` mapping each sample to its source ray [S3, unofficial].
*   `t_s`: Tensor of shape `(M,)` containing the start distance $t$ for each sample segment [S3, unofficial].
*   `t_e`: Tensor of shape `(M,)` containing the end distance $t$ for each sample segment [S3, unofficial].
*   `valid`: Boolean tensor of shape `(N,)` indicating which original rays had valid intersections [S3, unofficial].
*   `samples`: Optional tensor of sample positions if immediate evaluation is required [S2, unofficial].

**Data Structures:**
*   **Ray Array**: Can be packed as `(N, 6)` where columns 0-2 are origin and 3-5 are direction [S2, unofficial].
*   **Index Tensor**: `idx` of shape `(1, max_steps)` containing `[0, 1, ..., max_steps-1]` [S3, unofficial].
*   **Mask Tensor**: Boolean tensor used to filter valid samples based on intersection and sphere clipping [S3, unofficial].

**Steps:**

1.  **Ray Preprocessing**:
    *   Extract origins and directions from the rays array [S2, unofficial].
    *   Normalize directions: $\hat{d} = \frac{d}{\|d\|}$ [S2, unofficial].
    *   Compute inverse directions: $i_d = \frac{1}{\hat{d}}$ [S2, unofficial].

2.  **AABB Intersection**:
    *   Compute ray-AABB intersection to get near and far $t$-values: $t_{near}, t_{far} = \text{ray\_aabb\_intersect}(rays\_o, rays\_d, aabb\_min, aabb\_max)$ [S3, unofficial].
    *   Initialize valid mask: $ok = (t_{far} > t_{near})$ [S3, unofficial].
    *   Assert $t_{max} > t_{min}$ for all rays [S3, unofficial].

3.  **Optional Outer Sphere Clipping**:
    *   If `r_max_rsun` is provided, clip the ray interval to the sphere intersection [S3, unofficial].
    *   Calculate coefficients for the quadratic equation $|o + t d|^2 = (r_{max}/scale)^2$:
        $$
        \begin{aligned}
        A &= \sum (rays\_d \cdot rays\_d) \\
        B &= 2.0 \cdot \sum (rays\_o \cdot rays\_d) \\
        C &= \sum (rays\_o \cdot rays\_o) - \rho^2
        \end{aligned}
        $$
        where $\rho = \frac{r_{max\_rsun}}{aabb\_scale}$ [S3, unofficial].
    *   Solve for intersection times $t_{in}$ and $t_{out}$ using the quadratic formula with discriminant $disc = B^2 - 4AC$ [S3, unofficial].
    *   Handle numerical safety by clamping $A$ to $1e-20$ and $disc$ to $0.0$ [S3, unofficial].
    *   Update interval bounds: $ts_0 = \max(ts_0, t_{in})$ and $ts_1 = \min(ts_1, t_{out})$ [S3, unofficial].
    *   Update valid mask: $ok = ok \land hit$ [S3, unofficial].

4.  **Determine Valid Rays**:
    *   Final valid rays: $valid = ok \land (ts_1 > ts_0)$ [S3, unofficial].

5.  **Generate Uniform Segments**:
    *   For valid rays, let $a = ts_0[valid]$ and $b = ts_1[valid]$ [S3, unofficial].
    *   Determine the number of steps per ray: $n_{steps} = \lceil \frac{t_{max} - t_{min}}{render\_step\_size} \rceil$, clamped to a minimum of 1 [S3, unofficial].
    *   Determine $max\_steps$ as the maximum value in the $n_{steps}$ tensor [S3, unofficial].
    *   Create `n_samples + 1` linearly spaced points in $[0, 1]$: $u = \text{linspace}(0.0, 1.0, \text{int}(n\_samples) + 1)$ [S3, unofficial].
    *   Compute edge coordinates: $edges = a[:, \text{None}] + (b - a)[:, \text{None}] \cdot u[\text{None}, :]$ [S3, unofficial].
    *   Flatten edges into segment starts $t_s$ and ends $t_e$: $t_s = edges[:, :-1].\text{reshape}(-1)$ and $t_e = edges[:, 1:].\text{reshape}(-1)$ [S3, unofficial].
    *   Generate ray indices mapping segments to valid rays: $ray\_idx = \text{arange}(N_v).\text{repeat\_interleave}(\text{int}(n\_samples))$ [S3, unofficial].

6.  **Sample Generation (Optional Immediate Evaluation)**:
    *   Calculate step size: $stepSize = \frac{rayLength}{numSteps}$ [S1, unofficial].
    *   For each ray, generate $t$ values from $t_{min}$ to $t_{max}$ with step spacing [S2, unofficial].
    *   Compute sample positions: $pos = ori + t \cdot dir$ [S2, unofficial].
    *   Calculate atlas UV: $atlasUV = \frac{pos}{normalizedSize} + 0.5$ [S1, unofficial].
    *   Sample `volumeTexture` at `atlasUV` [S1, unofficial].
    *   Normalize density, apply windowing, and compose into color/alpha [S1, unofficial].
    *   Break if $\alpha > EARLY\_EXIT\_ALPHA$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def uniform_ray_marching(rays_o, rays_d, aabb_min, aabb_max, render_step_size, r_max_rsun=None, aabb_scale=1.0):
    """
    Generates uniform sample intervals along rays within an AABB.
    
    Args:
        rays_o: (N, 3) tensor of ray origins.
        rays_d: (N, 3) tensor of ray directions.
        aabb_min: (3,) tensor of AABB minimum bounds.
        aabb_max: (3,) tensor of AABB maximum bounds.
        render_step_size: Scalar float for fixed step distance.
        r_max_rsun: Optional float for outer sphere radius in solar radii.
        aabb_scale: Scalar for normalizing sphere radius to AABB scale.
        
    Returns:
        ray_idx: (M,) long tensor.
        t_s: (M,) float tensor of segment starts.
        t_e: (M,) float tensor of segment ends.
        valid: (N,) boolean tensor of valid rays.
    """
    N = rays_o.shape[0]
    
    # 1. Ray-AABB Intersection
    # Simplified slab method for AABB intersection
    t_min = torch.full((N,), -float('inf'))
    t_max = torch.full((N,), float('inf'))
    
    for i in range(3):
        inv_d = 1.0 / rays_d[:, i]
        t1 = (aabb_min[i] - rays_o[:, i]) * inv_d
        t2 = (aabb_max[i] - rays_o[:, i]) * inv_d
        
        near = torch.minimum(t1, t2)
        far = torch.maximum(t1, t2)
        
        t_min = torch.maximum(t_min, near)
        t_max = torch.minimum(t_max, far)
        
    # 2. Initial Valid Mask
    ok = t_max > t_min
    
    # 3. Optional Outer Sphere Clipping
    if r_max_rsun is not None:
        rho = r_max_rsun / aabb_scale
        A = torch.sum(rays_d * rays_d, dim=-1)
        B = 2.0 * torch.sum(rays_o * rays_d, dim=-1)
        C = torch.sum(rays_o * rays_o, dim=-1) - rho * rho
        
        # Numerical safety
        A = torch.clamp(A, min=1e-20)
        disc = B * B - 4.0 * A * C
        disc = torch.clamp(disc, min=0.0)
        
        sqrt_disc = torch.sqrt(disc)
        t_in = (-B - sqrt_disc) / (2.0 * A)
        t_out = (-B + sqrt_disc) / (2.0 * A)
        
        # Update bounds
        ts0 = torch.maximum(t_min, t_in)
        ts1 = torch.minimum(t_max, t_out)
        
        hit = (ts1 > ts0)
        ok = ok & hit
        
        t_min = ts0
        t_max = ts1
        
    # 4. Final Valid Rays
    valid = ok & (t_max > t_min)
    
    # 5. Generate Uniform Segments
    # Calculate number of steps per ray
    ray_lengths = t_max - t_min
    # Clamp to min 1 step if valid, else 0
    n_steps = torch.where(valid, torch.ceil(ray_lengths / render_step_size).long(), torch.zeros_like(ray_lengths, dtype=torch.long))
    n_steps = torch.clamp(n_steps, min=1)
    
    max_steps = torch.max(n_steps).item()
    
    # Create index tensor for broadcasting
    idx = torch.arange(max_steps, device=rays_o.device)
    
    # Broadcast n_steps to (N, 1) and create mask
    n_steps_exp = n_steps.unsqueeze(1)
    mask = idx < n_steps_exp
    
    # Compute start and end times
    # t_s = t_min + idx * step_size
    t_s = t_min.unsqueeze(1) + idx.unsqueeze(0) * render_step_size
    # t_e = t_min + (idx + 1) * step_size, clamped to t_max
    t_e = t_min.unsqueeze(1) + (idx + 1).unsqueeze(0) * render_step_size
    t_e = torch.minimum(t_e, t_max.unsqueeze(1))
    
    # Flatten using mask
    t_s_flat = t_s[mask].reshape(-1)
    t_e_flat = t_e[mask].reshape(-1)
    
    # Generate ray indices
    # Expand ray indices [0, N-1] to (N, max_steps) and flatten with mask
    ray_indices_expanded = torch.arange(N, device=rays_o.device).unsqueeze(1).expand(N, max_steps)
    ray_idx_flat = ray_indices_expanded[mask].reshape(-1)
    
    return ray_idx_flat, t_s_flat, t_e_flat, valid

def sample_uniform_shell(rays_o, rays_d, aabb_min, aabb_max, render_step_size, volume_texture, normalized_size, r_max_rsun=None, aabb_scale=1.0, early_exit_alpha=0.95):
    """
    Debug/Visualization mode: samples directly and composites.
    """
    ray_idx, t_s, t_e, valid = uniform_ray_marching(rays_o, rays_d, aabb_min, aabb_max, render_step_size, r_max_rsun, aabb_scale)
    
    # Compute midpoints for sampling
    t_mid = (t_s + t_e) / 2.0
    
    # Compute positions
    # rays_o and rays_d are (N, 3), ray_idx maps to them
    origins = rays_o[ray_idx]
    directions = rays_d[ray_idx]
    positions = origins + directions * t_mid.unsqueeze(-1)
    
    # Normalize positions to atlas UV
    # Assuming positions are in world space and need to be mapped to [0,1] via normalized_size
    atlas_uv = (positions / normalized_size) + 0.5
    
    # Sample volume texture (Placeholder for actual texture lookup logic)
    # In a real implementation, this would call texture3d or similar
    densities = sample_volume_texture(atlas_uv, volume_texture)
    
    # Composition
    # extinction_scale = step_size * atlas_size * 0.5 * LOG2E
    # This formula is specific to the kiln-renderer context [S1, unofficial]
    # For general uniform marching, standard volume rendering equation is used:
    # C = sum(T_i * (1 - exp(-sigma_a_i * delta_i)))
    
    # Simplified composition loop
    # Note: This is a high-level representation. Actual implementation requires 
    # handling of alpha accumulation and early exit.
    
    return densities, valid
```

**Parameters:**
*   `numSteps`: Typically 512 [S1, unofficial].
*   `spacing`: Distance between consecutive samples [S2, unofficial].
*   `box_top`, `box_bottom`: AABB boundaries [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision**: Clamp $A$ to $1e-20$ and discriminant to $0.0$ in quadratic sphere intersection to avoid NaNs [S3, unofficial].
*   **Boundaries**: Assert $t_{max} > t_{min}$ for all rays [S3, unofficial].
*   **Degenerate Input**: Rays with $t_{max} \le t_{min}$ are filtered out by the `valid` mask [S3, unofficial].
*   **Aliasing**: Uniform sampling avoids concentric "sampling rings" caused by steep gradients near inner radius cuts, which can occur with coarse sampling [S3, unofficial].

## Subtopics

### [NeRF Ray Marching Optimizations](9999-nerf-ray-marching-optimizations.md)
A set of three techniques to optimize NeRF ray marching performance with imperceptible error: exponential stepping, empty space skipping, and sample compaction. It builds on Uniform Ray Marching by replacing fixed uniform intervals with adaptive step sizes and skipping empty regions via multiscale occupancy grids.

### Oversampled 3D Ray March
An oversampling technique where each fine sample owns the slab it is the midpoint of, clipped to the brick's segment, ensuring slab thicknesses tile $[0, len]$ exactly at LOD interfaces [S4, unofficial]. Previous empty-space skip probed one stride past each cube's exit face, restarting at every chunk/coarse-floor cube face and stroding 1.9 level voxels. This left material in the last sub-stride unprobed, causing zero contribution from the cube and discontinuities at exit faces [S4, unofficial]. CPU replica worst-case two-cube split error: -3.99% to +0.68%. Browser A/B (HOA heart, pinned L4, 64 chunks, WebGPU): whole-image TV -2.7% x / -2.2% y [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the number of steps per ray. Since $S$ is fixed or bounded by the AABB length divided by `render_step_size`, it is linear with respect to the number of rays and the volume depth along the ray.
*   **Memory Complexity**: $O(N \cdot S)$ to store the sample indices and $t$ values. The output tensors `ray_idx`, `t_s`, and `t_e` have shape $(M,)$ where $M$ is the total number of valid samples [S3, unofficial].
*   **Performance**: Uniform marching is computationally straightforward but may be inefficient for sparse volumes or large empty spaces compared to adaptive methods [S1, S3, unofficial]. It is suitable for dense volumes or as a baseline/debug tool [S1, S3, unofficial].
*   **Strengths**: Simple to implement, deterministic, avoids aliasing artifacts from coarse sampling [S3, unofficial], no need for auxiliary data structures like occupancy grids [S1, unofficial].
*   **Limitations**: Inefficient for sparse data, fixed step size may lead to over-sampling in empty regions or under-sampling in high-frequency regions [S1, unofficial].
*   **When to use**: Debugging, visualization, dense volumes, or when adaptive libraries are unavailable [S1, S3, unofficial].
*   **When not to use**: Large sparse volumes, real-time applications requiring high performance with sparse data, or when adaptive stepping is critical [S1, unofficial].

## Background
Uniform Ray Marching is a fundamental technique in volume rendering, often used as a baseline or fallback method. It is implemented in various open-source projects for volume visualization, including debug modes in WebGPU-native renderers [S1, unofficial], Jax-based volumetric renderers [S2, unofficial], and NeRF-based solar corona renderers [S3, unofficial]. It is also used in multi-LOD streaming contexts to correct discontinuities at LOD interfaces [S4, unofficial].

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S2] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S3] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S4] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
