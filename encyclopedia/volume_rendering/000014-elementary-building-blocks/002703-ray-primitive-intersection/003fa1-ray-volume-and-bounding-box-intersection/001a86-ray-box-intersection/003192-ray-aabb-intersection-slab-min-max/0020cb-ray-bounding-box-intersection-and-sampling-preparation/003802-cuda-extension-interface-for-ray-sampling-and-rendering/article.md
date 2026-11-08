# CUDA Extension Interface for Ray Sampling and Rendering


## Overview
This module provides a PyTorch C++ extension containing CUDA kernels for the core geometric and radiometric computations in volume rendering pipelines, specifically ray-sampling preparation, alpha-to-weight accumulation, and density-to-alpha conversion [S1, unofficial]. It serves as the computational bridge between the ray-box intersection logic described in [Ray-Bounding Box Intersection and Sampling Preparation](2364-ray-bounding-box-intersection-and-sampling-preparation.md) and the final volume integration. Unlike high-level rendering abstractions, this interface exposes low-level tensor operations for batched ray processing, enabling efficient GPU execution of point sampling, masking, and differentiable alpha blending [S1, unofficial].

## How it works

### Inputs and Outputs
The interface consists of multiple CUDA kernels exposed via `pybind11`. Each kernel accepts PyTorch tensors as inputs and returns PyTorch tensors or vectors of tensors as outputs. All tensors are assumed to be on the same CUDA device.

### Data Structures
The data structures are implicit in the tensor shapes defined by the kernel signatures.
- **Rays**: Defined by origin `rays_o` (shape `[N_rays, 3]`) and direction `rays_d` (shape `[N_rays, 3]`). Directions are assumed to be normalized.
- **Bounding Box**: Defined by `xyz_min` and `xyz_max` (shape `[3]`), representing the world-space bounds of the volume.
- **Mask Cache**: A 3D tensor `world` (shape `[D, H, W]`) representing a binary occupancy grid.
- **Density/Alpha**: 1D tensors of shape `[N_samples]` or `[N_rays * N_samples]`.

### Kernel Definitions and Logic

#### 1. Ray-Bounding Box Intersection (`infer_t_minmax`)
Computes the entry (`t_min`) and exit (`t_max`) distances of each ray with the axis-aligned bounding box defined by `xyz_min` and `xyz_max`.

**Inputs**:
- `rays_o`: `[N_rays, 3]`
- `rays_d`: `[N_rays, 3]`
- `xyz_min`: `[3]`
- `xyz_max`: `[3]`
- `near`: `float` (minimum allowed $t$)
- `far`: `float` (maximum allowed $t$)

**Outputs**:
- `t_min`: `[N_rays]`
- `t_max`: `[N_rays]`

**Logic**:
For each ray $i$, compute the slab intersections for each axis $j \in \{x, y, z\}$:
$$ t_{enter, j} = \frac{xyz\_min[j] - rays\_o[i, j]}{rays\_d[i, j]} $$
$$ t_{exit, j} = \frac{xyz\_max[j] - rays\_o[i, j]}{rays\_d[i, j]} $$
Handle division by zero (ray parallel to slab) by setting $t_{enter, j} = -\infty$ and $t_{exit, j} = \infty$ if $rays\_d[i, j] = 0$.
$$ t_{min, i} = \max_j(t_{enter, j}) $$
$$ t_{max, i} = \min_j(t_{exit, j}) $$
Clamp results:
$$ t_{min, i} = \max(t_{min, i}, near) $$
$$ t_{max, i} = \min(t_{max, i}, far) $$
If $t_{min, i} > t_{max, i}$, the ray does not intersect the box (or is behind the near plane). The implementation typically handles this by setting $t_{min} = t_{max}$ or marking as invalid, though exact handling of non-intersecting rays depends on the caller's validation step [S1, unofficial].

#### 2. Sample Count Inference (`infer_n_samples`)
Determines the number of sample points required for each ray based on a fixed step distance.

**Inputs**:
- `rays_d`: `[N_rays, 3]`
- `t_min`: `[N_rays]`
- `t_max`: `[N_rays]`
- `stepdist`: `float`

**Outputs**:
- `n_samples`: `[N_rays]` (integer tensor)

**Logic**:
$$ L_i = t_{max, i} - t_{min, i} $$
$$ n\_samples[i] = \left\lceil \frac{L_i}{stepdist} \right\rceil $$
If $L_i \le 0$, $n\_samples[i] = 0$ [S1, unofficial].

#### 3. Ray Start and Direction Inference (`infer_ray_start_dir`)
Prepares the starting point and direction for sampling, potentially adjusting for near/far clipping or specific coordinate systems.

**Inputs**:
- `rays_o`: `[N_rays, 3]`
- `rays_d`: `[N_rays, 3]`
- `t_min`: `[N_rays]`

**Outputs**:
- `start_points`: `[N_rays, 3]`
- `dirs`: `[N_rays, 3]`

**Logic**:
$$ start\_points[i] = rays\_o[i] + t_{min, i} \cdot rays\_d[i] $$
$$ dirs[i] = rays\_d[i] $$
This ensures sampling begins exactly at the entry point of the bounding box [S1, unofficial].

#### 4. Point Sampling on Rays (`sample_pts_on_rays`)
Generates the 3D coordinates of sample points along each ray.

**Inputs**:
- `rays_o`: `[N_rays, 3]`
- `rays_d`: `[N_rays, 3]`
- `xyz_min`: `[3]`
- `xyz_max`: `[3]`
- `near`: `float`
- `far`: `float`
- `stepdist`: `float`

**Outputs**:
- `pts`: `[N_rays, N_max_samples, 3]` (padded with zeros or NaNs for rays with fewer samples)
- `t_vals`: `[N_rays, N_max_samples]` (absolute $t$ values)
- `valid_mask`: `[N_rays, N_max_samples]` (boolean, true if sample is within bounds)

**Logic**:
1. Compute $n\_samples[i]$ for all rays.
2. Determine $N_{max} = \max_i(n\_samples[i])$.
3. For each ray $i$ and sample index $k \in [0, n\_samples[i]-1]$:
   $$ t_{val, i, k} = t_{min, i} + \left(k + 0.5\right) \cdot stepdist $$
   $$ pts[i, k] = rays\_o[i] + t_{val, i, k} \cdot rays\_d[i] $$
4. For $k \ge n\_samples[i]$, mark as invalid [S1, unofficial].

#### 5. NDC Point Sampling (`sample_ndc_pts_on_rays`)
Samples points in Normalized Device Coordinates (NDC), typically used for unbounded scenes or specific camera models.

**Inputs**:
- `rays_o`: `[N_rays, 3]`
- `rays_d`: `[N_rays, 3]`
- `xyz_min`: `[3]`
- `xyz_max`: `[3]`
- `N_samples`: `int` (fixed number of samples per ray)

**Outputs**:
- `pts`: `[N_rays, N_samples, 3]`
- `t_vals`: `[N_rays, N_samples]`

**Logic**:
Similar to `sample_pts_on_rays`, but the step size is derived from the distance between `xyz_min` and `xyz_max` along the ray direction, or a fixed NDC interval, ensuring uniform sampling in NDC space rather than world space [S1, unofficial].

#### 6. Background Point Sampling (`sample_bg_pts_on_rays`)
Samples points in the background region (outside the bounding box) to handle occlusion and background color estimation.

**Inputs**:
- `rays_o`: `[N_rays, 3]`
- `rays_d`: `[N_rays, 3]`
- `t_max`: `[N_rays]`
- `bg_preserve`: `float` (scaling factor for background depth)
- `N_samples`: `int`

**Outputs**:
- `bg_pts`: `[N_rays, N_samples, 3]`

**Logic**:
Points are sampled starting from $t_{max}$ extending into the background. The spacing may be uniform or exponential depending on `bg_preserve` [S1, unofficial].

#### 7. Mask Cache Lookup (`maskcache_lookup`)
Checks if sample points fall within known free space (mask=0) to skip rendering.

**Inputs**:
- `world`: `[D, H, W]` (3D occupancy grid, 0 for free, 1 for occupied)
- `xyz`: `[N_pts, 3]` (sample points)
- `xyz2ijk_scale`: `[3]` (scaling factor from world to grid coordinates)
- `xyz2ijk_shift`: `[3]` (shift from world to grid coordinates)

**Outputs**:
- `mask`: `[N_pts]` (boolean, true if point is in occupied space)

**Logic**:
1. Convert world coordinates to grid indices:
   $$ ijk = (xyz \cdot xyz2ijk\_scale) + xyz2ijk\_shift $$
2. Clamp $ijk$ to $[0, D-1] \times [0, H-1] \times [0, W-1]$.
3. Trilinearly interpolate or nearest-neighbor lookup in `world` to get occupancy [S1, unofficial].

#### 8. Raw to Alpha Conversion (`raw2alpha`)
Converts raw density values to alpha (opacity) values.

**Inputs**:
- `density`: `[N]` (raw density, can be negative)
- `shift`: `float` (offset for numerical stability)
- `interval`: `float` (step distance for this ray segment)

**Outputs**:
- `alpha`: `[N]`
- `exp_density`: `[N]` (intermediate exponential value)

**Logic**:
$$ \sigma' = \max(density + shift, 0) $$
$$ \alpha = 1 - \exp(-\sigma' \cdot interval) $$
$$ exp\_density = \exp(-\sigma' \cdot interval) $$
The `shift` parameter prevents $\exp(-\infty)$ issues when density is very negative [S1, unofficial].

#### 9. Non-Uniform Raw to Alpha (`raw2alpha_nonuni`)
Same as `raw2alpha` but supports per-sample intervals.

**Inputs**:
- `density`: `[N]`
- `shift`: `float`
- `interval`: `[N]` (per-sample step distance)

**Outputs**:
- `alpha`: `[N]`
- `exp_density`: `[N]`

**Logic**:
$$ \sigma' = \max(density + shift, 0) $$
$$ \alpha[i] = 1 - \exp(-\sigma'[i] \cdot interval[i]) $$
$$ exp\_density[i] = \exp(-\sigma'[i] \cdot interval[i]) $$
This is used when sampling is non-uniform along the ray [S1, unofficial].

#### 10. Alpha to Weight Accumulation (`alpha2weight`)
Computes the contribution weight of each sample point in the volume rendering integral.

**Inputs**:
- `alpha`: `[N_rays * N_samples]`
- `ray_id`: `[N_rays * N_samples]` (maps each sample to its ray index)
- `n_rays`: `int`

**Outputs**:
- `weight`: `[N_rays * N_samples]`
- `T`: `[N_rays * N_samples]` (transmittance at each sample)
- `alphainv_last`: `[N_rays]` (inverse transmittance at the end of each ray)

**Logic**:
For each ray $i$, compute transmittance $T$ and weight $W$:
$$ T[i, 0] = 1.0 $$
$$ T[i, k] = T[i, k-1] \cdot (1 - \alpha[i, k-1]) \quad \text{for } k > 0 $$
$$ W[i, k] = T[i, k] \cdot \alpha[i, k] $$
$$ alphainv\_last[i] = T[i, N_{samples}] $$
This is a sequential reduction operation, typically implemented with a parallel prefix sum (scan) on GPU [S1, unofficial].

#### 11. Backward Passes
- `raw2alpha_backward`: Computes gradients of `density` given gradients of `alpha`.
  $$ \frac{\partial \alpha}{\partial \sigma'} = -interval \cdot \exp(-\sigma' \cdot interval) $$
  $$ \frac{\partial \sigma'}{\partial density} = 1 \quad \text{if } density + shift > 0 \text{ else } 0 $$
- `raw2alpha_nonuni_backward`: Same as above but with per-sample `interval`.
- `alpha2weight_backward`: Computes gradients of `alpha` given gradients of `weight`. This involves a backward scan of the transmittance product [S1, unofficial].

### Pseudocode Example: Alpha2Weight Forward

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def alpha2weight_forward(alpha, ray_id, n_rays):
    """
    alpha: [N_rays * N_samples]
    ray_id: [N_rays * N_samples]
    n_rays: int
    """
    N_total = alpha.size(0)
    N_samples = N_total // n_rays
    
    # Initialize T and weight tensors
    T = torch.zeros(N_total, device=alpha.device)
    weight = torch.zeros(N_total, device=alpha.device)
    alphainv_last = torch.zeros(n_rays, device=alpha.device)
    
    # Process each ray
    for r in range(n_rays):
        start = r * N_samples
        end = start + N_samples
        
        # Transmittance calculation
        T[start] = 1.0
        for k in range(1, N_samples):
            T[start + k] = T[start + k - 1] * (1.0 - alpha[start + k - 1])
            
        # Weight calculation
        for k in range(N_samples):
            weight[start + k] = T[start + k] * alpha[start + k]
            
        alphainv_last[r] = T[start + N_samples - 1] * (1.0 - alpha[start + N_samples - 1])
        
    return weight, T, alphainv_last
```
*Note: In practice, this is implemented using CUDA atomic operations or parallel scans for efficiency [S1, unofficial].*

## Operating characteristics

- **Time Complexity**: $O(N_{rays} \cdot N_{samples})$ for sampling and accumulation. The parallel prefix sum in `alpha2weight` is $O(\log N_{samples})$ per ray on GPU.
- **Memory Complexity**: $O(N_{rays} \cdot N_{samples})$ for storing points, transmittance, and weights.
- **Performance**: Highly optimized for GPU execution. The use of CUDA kernels avoids CPU-GPU data transfer overhead. Performance scales with the number of rays and samples, limited by memory bandwidth for large volumes.
- **Strengths**: Efficient batched processing, differentiable, supports non-uniform sampling.
- **Limitations**: Requires careful handling of padding for rays with different sample counts. Memory usage can be high for high-resolution volumes with many samples per ray [S1, unofficial].

## Background

This interface is part of the UnboundedNeRFPytorch implementation, an unofficial PyTorch port of NeRF-related methods for unbounded scenes [S1, unofficial]. It extends standard NeRF rendering pipelines by providing specialized kernels for ray-box intersection and sampling, which are crucial for handling large, unbounded volumes efficiently. The code is based on the `render_utils.cpp` module from the original NeRF implementations, adapted for PyTorch's autograd system [S1, unofficial].

## Sources

[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
