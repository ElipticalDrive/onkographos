# Uniform Ray Marching

Also called: Fixed-Step Ray Marching Setup, Normalized Step Size Calculation, Parallel Ray Sampling (Fallback), Ray-to-Sample Generation, Simple ray marching (debug view), Uniform Ray Marching for Visualization

## Overview
Uniform Ray Marching generates 3D sample positions along rays within an Axis-Aligned Bounding Box (AABB) using a fixed spacing between samples [S2, S4, unofficial]. It serves as a debug rendering mode that performs direct atlas sampling without brick traversal or indirection [S1, unofficial], and as a PyTorch-native fallback for ray sampling that does not rely on custom CUDA extensions [S3, S4, unofficial]. It is also used as a dense uniform sampler for Line-of-Sight (LOS) integral visualization to avoid aliasing artifacts caused by steep gradients near inner radius cuts [S4, unofficial]. It builds on [Raycasting Step Size & Aspect Ratio Normalization](3734-raycasting-step-size-aspect-ratio-normalization.md) by utilizing normalized step sizes and scaling factors to map arbitrary volume dimensions into a unit cube coordinate system [S1, S4, unofficial].

## How it works
**Inputs:**
*   `rays_o`: Tensor of ray origins, shape `(N, 3)` [S4, unofficial].
*   `rays_d`: Tensor of ray directions, shape `(N, 3)` [S4, unofficial].
*   `aabb_min`, `aabb_max`: Vectors defining the AABB boundaries [S2, S4, unofficial].
*   `render_step_size`: Scalar float defining the fixed distance between sample points [S4, unofficial].
*   `n_samples`: Integer specifying the number of uniform sub-segments per ray (alternative input to `render_step_size`) [S4, unofficial].
*   `r_max_rsun`: Optional float specifying an outer sphere radius in solar radii ($R_{sun}$) for clipping [S4, unofficial].
*   `atlasSize`: Size of the texture atlas for coordinate normalization [S1, unofficial].
*   `numSteps`: Fixed number of steps (alternative to `render_step_size`) [S1, unofficial].
*   `spacing`: Distance between consecutive samples (alternative to `render_step_size`) [S2, unofficial].

**Outputs:**
*   `sample_positions`: 3D coordinates of sample points, shape `(N, M, 3)` or flattened `(M_total, 3)` [S2, S4, unofficial].
*   `t_s`: Start distances along rays for each segment, shape `(M,)` [S4, unofficial].
*   `t_e`: End distances along rays for each segment, shape `(M,)` [S4, unofficial].
*   `ray_idx`: Indices mapping segments to original rays, shape `(M,)` [S4, unofficial].
*   `valid`: Boolean mask of rays with valid intersections, shape `(N,)` [S4, unofficial].

**Data Structures:**
*   **Rays Array**: In some implementations, rays are packed into a single array of shape `(N, 6)` containing `[origin_xyz, direction_xyz]` [S2, unofficial].
*   **Depth/Distance Tensors**: 1D tensors storing `t_min`, `t_max`, and sampled depths `sampled_depth` [S3, S4, unofficial].
*   **Index Tensors**: Integer tensors for mapping samples to rays and managing padding/masking [S3, S4, unofficial].

**Steps:**

1.  **Ray Preparation**: Extract ray origins and directions. Normalize directions to unit length [S2, unofficial]. Compute inverse directions if required for AABB intersection logic [S2, unofficial].
2.  **AABB Intersection**: Compute the near (`t_min` or `t_near`) and far (`t_max` or `t_far`) intersection distances for each ray with the AABB [S2, S4, unofficial].
    *   Formula: Standard ray-AABB intersection algorithm [S2, S4, unofficial].
3.  **Sphere Clipping (Optional)**: If an outer sphere radius `r_max_rsun` is provided, clip the ray interval to the sphere intersection [S4, unofficial].
    *   Calculate coefficients for the quadratic equation $|o + t d|^2 = (r_{max}/scale)^2$:
        $$ A = \sum (rays_d^2) $$
        $$ B = 2.0 \cdot \sum (rays_o \cdot rays_d) $$
        $$ C = \sum (rays_o^2) - \rho^2 $$
        where $\rho = r_{max\_rsun} / aabb\_scale$ [S4, unofficial].
    *   Solve for intersection times $t_{in}$ and $t_{out}$ using the quadratic formula. Handle numerical safety by clamping $A$ to $1e-20$ and the discriminant $disc = B^2 - 4AC$ to $0.0$ [S4, unofficial].
    *   Update interval bounds: $t_{min} = \max(t_{min}, t_{in})$ and $t_{max} = \min(t_{max}, t_{out})$ [S4, unofficial].
4.  **Validity Check**: Filter rays where $t_{min} < t_{max}$ (and $t_{far} > t_{near}$) [S2, S4, unofficial]. Initialize a valid mask `ok = t_far > t_near` [S4, unofficial].
5.  **Step Size / Number Calculation**:
    *   **Method A (Fixed Step Size)**: Calculate the number of steps per ray as $n_{steps} = \lceil (t_{max} - t_{min}) / render\_step\_size \rceil$, clamped to a minimum of 1 [S4, unofficial]. Determine `max_steps` as the maximum value in this tensor [S4, unofficial].
    *   **Method B (Fixed Number of Steps)**: Calculate step size as $stepSize = rayLength / numSteps$ [S1, unofficial].
    *   **Method C (Fixed Spacing)**: Use the provided `spacing` parameter directly [S2, unofficial].
6.  **Sample Generation**:
    *   **Method A (Segment Edges)**: For valid rays, generate $n_{samples} + 1$ linearly spaced points in $[0, 1]$: $u = \text{linspace}(0.0, 1.0, n_{samples} + 1)$ [S4, unofficial]. Compute edge coordinates: $edges = t_{min}[:, None] + (t_{max} - t_{min})[:, None] \cdot u[None, :]$ [S4, unofficial]. Flatten edges into segment starts $t_s$ and ends $t_e$ [S4, unofficial].
    *   **Method B (Arange)**: Generate delta offsets: $delta = \text{arange}(\lfloor max\_ray\_length / MARCH\_SIZE \rfloor)$ [S3, unofficial]. Scale delta by `MARCH_SIZE` (or `render_step_size`) [S3, S4, unofficial]. Compute sampled depths: $sampled\_depth = t_{min}[:, :1] + delta$ [S3, unofficial].
    *   **Method C (Loop)**: For each ray, generate $t$ values from $t_{min}$ to $t_{max}$ with the fixed spacing [S2, unofficial].
7.  **Position Computation**: Compute sample positions as $pos = ori + t \cdot dir$ [S2, unofficial].
8.  **Index and Mask Construction**:
    *   Create an index tensor `idx` of shape `(1, max_steps)` containing $[0, 1, \dots, max\_steps-1]$ [S4, unofficial].
    *   Broadcast `n_steps` to shape `(N, 1)` and create a boolean mask where `idx < n_steps` [S4, unofficial].
    *   Compute `ray_idx` by expanding ray indices $[0, N-1]$ to shape `(N, max_steps)` and flattening with the mask [S4, unofficial].
    *   **Invalid Depth Handling**: Identify invalid depths based on comparisons with `min_depth` and `max_depth`. Mask invalid depths with a constant `MAX_DEPTH` (e.g., 10000.0) [S3, unofficial]. Sort invalid entries to the end and trim outputs to `num_max_steps` (maximum valid steps per ray) [S3, unofficial].
9.  **Sampling (Atlas Mode)**: If using atlas sampling, normalize positions to UV coordinates: $atlasUV = (pos / normalizedSize) + 0.5$ [S1, unofficial]. Sample the `volumeTexture` at `atlasUV` [S1, unofficial].
10. **Composition**: Normalize density, apply windowing, and compose into color/alpha. Break early if $\alpha > EARLY\_EXIT\_ALPHA$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def uniform_ray_marching(rays_o, rays_d, aabb_min, aabb_max, render_step_size=None, n_samples=None, r_max_rsun=None):
    """
    Generates uniform sample positions along rays within an AABB.
    
    Args:
        rays_o: (N, 3) tensor of ray origins.
        rays_d: (N, 3) tensor of ray directions.
        aabb_min: (3,) tensor of AABB minimum bounds.
        aabb_max: (3,) tensor of AABB maximum bounds.
        render_step_size: Scalar float for fixed step distance.
        n_samples: Integer for fixed number of samples per ray.
        r_max_rsun: Optional float for outer sphere radius clipping.
        
    Returns:
        t_s: (M,) tensor of segment start distances.
        t_e: (M,) tensor of segment end distances.
        ray_idx: (M,) tensor of ray indices for each segment.
        valid: (N,) boolean tensor of valid rays.
    """
    # 1. Ray Preparation
    # Directions are assumed normalized or normalized here.
    # Normalize directions
    rays_d = rays_d / torch.norm(rays_d, dim=1, keepdim=True)
    
    # 2. AABB Intersection
    # Standard ray-AABB intersection
    # t_min = max((aabb_min - o) / d) for d > 0, min((aabb_min - o) / d) for d < 0
    # t_max = min((aabb_max - o) / d) for d > 0, max((aabb_max - o) / d) for d < 0
    # Implementation depends on specific intersection routine.
    t_min, t_max = ray_aabb_intersect(rays_o, rays_d, aabb_min, aabb_max)
    
    # 3. Sphere Clipping (Optional)
    if r_max_rsun is not None:
        # Calculate coefficients for quadratic equation |o + t d|^2 = (r_max/scale)^2
        # Assuming aabb_scale is 1.0 or provided. Let rho = r_max_rsun / aabb_scale.
        rho = r_max_rsun # Assuming aabb_scale=1 for simplicity, or pass it
        A = torch.sum(rays_d * rays_d, dim=1)
        B = 2.0 * torch.sum(rays_o * rays_d, dim=1)
        C = torch.sum(rays_o * rays_o, dim=1) - rho * rho
        
        # Clamp A for numerical safety
        A = torch.clamp(A, min=1e-20)
        disc = B * B - 4.0 * A * C
        disc = torch.clamp(disc, min=0.0)
        
        sqrt_disc = torch.sqrt(disc)
        t_in = (-B - sqrt_disc) / (2.0 * A)
        t_out = (-B + sqrt_disc) / (2.0 * A)
        
        # Update interval bounds
        t_min = torch.maximum(t_min, t_in)
        t_max = torch.minimum(t_max, t_out)
        
    # 4. Validity Check
    ok = t_max > t_min
    valid = ok
    
    # 5. Step Size / Number Calculation
    if n_samples is not None:
        # Method A: Fixed number of samples
        n_steps = torch.full_like(t_min, n_samples, dtype=torch.long)
        max_steps = n_samples
    elif render_step_size is not None:
        # Method B: Fixed step size
        ray_lengths = t_max - t_min
        n_steps = torch.ceil(ray_lengths / render_step_size).long()
        n_steps = torch.clamp(n_steps, min=1)
        max_steps = torch.max(n_steps).item()
    else:
        raise ValueError("Either n_samples or render_step_size must be provided")
        
    # 6. Sample Generation
    # Create index tensor for steps
    idx = torch.arange(max_steps, device=rays_o.device).unsqueeze(0) # (1, max_steps)
    
    # Broadcast n_steps to (N, 1) and create mask
    n_steps_exp = n_steps.unsqueeze(1) # (N, 1)
    mask = idx < n_steps_exp # (N, max_steps)
    
    # Compute start times t_s
    # t_s = t_min + idx * step_size
    # If using fixed step size, step_size is render_step_size
    # If using fixed n_samples, step_size varies per ray: (t_max - t_min) / n_samples
    if render_step_size is not None:
        step_size = render_step_size
    else:
        step_size = (t_max - t_min).unsqueeze(1) / n_samples # (N, 1)
        
    t_s = t_min.unsqueeze(1) + idx * step_size # (N, max_steps)
    
    # Compute end times t_e
    t_e = t_min.unsqueeze(1) + (idx + 1) * step_size # (N, max_steps)
    t_e = torch.minimum(t_e, t_max.unsqueeze(1)) # Clamp to t_max
    
    # Flatten using mask
    t_s_flat = t_s[mask]
    t_e_flat = t_e[mask]
    
    # 7. Index Construction
    ray_indices = torch.arange(len(rays_o), device=rays_o.device) # (N,)
    ray_idx_flat = ray_indices.repeat_interleave(n_steps) # (M,)
    
    # Trim to valid rays if necessary, though mask handles invalid rays implicitly if t_min >= t_max
    # The mask `ok` ensures t_min < t_max. 
    # However, if we want to return only samples from valid rays:
    final_mask = mask & ok.unsqueeze(1)
    t_s_flat = t_s[final_mask]
    t_e_flat = t_e[final_mask]
    ray_idx_flat = ray_indices.repeat_interleave(n_steps)[final_mask]
    
    return t_s_flat, t_e_flat, ray_idx_flat, ok

def ray_aabb_intersect(origins, directions, aabb_min, aabb_max):
    """
    Computes t_min and t_max for ray-AABB intersection.
    origins: (N, 3)
    directions: (N, 3)
    aabb_min: (3,)
    aabb_max: (3,)
    Returns: t_min (N,), t_max (N,)
    """
    # Compute inverse directions
    inv_dir = 1.0 / directions
    
    # Compute t values for min and max bounds
    t1 = (aabb_min - origins) * inv_dir
    t2 = (aabb_max - origins) * inv_dir
    
    # t_min is the maximum of the minimums
    t_min = torch.maximum(t1, t2).max(dim=1).values
    # t_max is the minimum of the maximums
    t_max = torch.minimum(t1, t2).min(dim=1).values
    
    return t_min, t_max
```

**Parameters:**
*   `numSteps`: 512 (typical value for debug/atlas mode) [S1, unofficial].
*   `MAX_DEPTH`: 10000.0 (constant for marking invalid depths) [S3, unofficial].
*   `EARLY_EXIT_ALPHA`: Threshold for early termination in composition [S1, unofficial].
*   `LOG2E`: Constant for extinction scale calculation ($1/\ln(2)$) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Invalid Depths**: Depths outside the valid range must be masked with `MAX_DEPTH` and sorted to the end to allow trimming to the maximum number of valid samples per ray [S3, unofficial].
*   **Numerical Safety**: Clamp $A$ to $1e-20$ and discriminant to $0.0$ in sphere intersection to avoid NaNs [S4, unofficial].
*   **Steep Gradients**: Uniform sampling may cause aliasing artifacts (concentric 'sampling rings') near inner radius cuts; dense uniform sampling is recommended to avoid this [S4, unofficial].
*   **Oversampling Artifacts**: In LOD contexts, improper slab ownership can cause discontinuities at exit faces [S5, unofficial].

## Subtopics

### [NeRF Ray Marching Optimizations](9999-nerf-ray-marching-optimizations.md)
A set of three techniques to optimize NeRF ray marching performance with imperceptible error: exponential stepping, empty space skipping, and sample compaction. It builds on Uniform Ray Marching by replacing fixed uniform intervals with adaptive step sizes and skipping empty regions via multiscale occupancy grids.

### [Voxel-Aware Masked Ray Sampling](11092-voxel-aware-masked-ray-sampling.md)
Voxel-Aware Masked Ray Sampling is a ray sampling strategy for sparse voxel fields that samples points only within intersected voxels, handling variable numbers of intersections per ray by padding and masking, and supporting hierarchical sampling ratios. It differs from standard NeRF uniform or stratified sampling by using voxel-based intersection sampling with probabilities proportional to distance. It builds on Uniform Ray Marching by replacing fixed intervals with adaptive steps derived from voxel geometry.

### Oversampled 3D Ray March
An oversampling technique where each fine sample owns the slab it is the midpoint of, clipped to the brick's segment, ensuring slab thicknesses tile $[0, len]$ exactly at LOD interfaces [S5, unofficial]. Previous empty-space skip probed one stride past each cube's exit face, restarting at every chunk/coarse-floor cube face and stroding 1.9 level voxels, leaving material in the last sub-stride unprobed and causing zero contribution from the cube and discontinuities at exit faces [S5, unofficial]. CPU replica worst-case two-cube split error: -3.99% to +0.68% [S5, unofficial]. Browser A/B (HOA heart, pinned L4, 64 chunks, WebGPU): whole-image TV -2.7% x / -2.2% y [S5, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of rays and $M$ is the number of samples per ray [S4, unofficial].
*   **Memory Complexity**: $O(N \cdot M)$ to store sample positions and indices [S4, unofficial].
*   **Performance**: Provides a dense, uniform sampling of the volume, which can be computationally expensive for high-resolution volumes or large step counts [S4, unofficial]. It is suitable for debug visualization and scenarios where uniform coverage is required to avoid aliasing [S1, S4, unofficial].
*   **Strengths**: Simple to implement, deterministic, avoids artifacts from adaptive stepping in uniform regions, serves as a robust fallback for hardware without custom extensions [S1, S3, S4, unofficial].
*   **Limitations**: Inefficient for sparse volumes or large empty spaces, as it samples everywhere uniformly [S1, unofficial]. May require a large number of samples to achieve high quality, leading to high memory and compute usage [S4, unofficial].
*   **When to Use**: Debugging, visualization, dense uniform sampling requirements, fallback environments [S1, S3, S4, unofficial].
*   **When Not to Use**: Large sparse volumes, real-time applications with strict performance constraints where empty space skipping is beneficial [S1, unofficial].

## Background
Uniform Ray Marching is a fundamental technique in volume rendering, often used as a baseline or fallback method. It is implemented in various open-source projects for volume rendering, including debug modes in WebGPU-native renderers [S1, unofficial], Jax-based volumetric renderers [S2, unofficial], and PyTorch-native implementations for Neural Sparse Voxel Fields [S3, unofficial] and Coronal NeRF [S4, unofficial]. It is also discussed in the context of LOD compensation and slab tiling fixes in multi-LOD rendering systems [S5, unofficial].

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
[S2] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S3] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S4] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S5] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
