# Voxel Aggregation Methods

Also called: Ray-based voxel aggregation, Differentiable voxel aggregation

## Overview
Voxel aggregation methods determine the final pixel color and properties by combining voxel values along a ray cast from the camera. They solve the problem of mapping 3D volumetric data to a 2D image while maintaining differentiability with respect to the voxel inputs, enabling gradient-based optimization in differentiable rendering pipelines [S1, unofficial]. Unlike sibling methods that may rely on static spatial queries or instance propagation, these methods explicitly traverse the voxel grid along viewing rays to accumulate or select values [S1, unofficial].

## How it works
**Inputs:** A voxel grid containing values (e.g., occupancy probability, signed distance, color, opacity) and camera parameters (origin, direction).
**Outputs:** Pixel color, opacity, depth, and optionally normals.

The general process involves casting a ray from each pixel through the voxel grid and applying a specific aggregation function to the sampled voxels.

### 1. Occupancy Probability (Max)
Yan et al. [7] associate an occupancy probability with each pixel.
1. Cast a ray from the pixel.
2. Sample all voxels intersected by the ray.
3. Compute the pixel value as the maximum occupancy probability among the sampled voxels.
$$ C_{pixel} = \max_{v \in \text{ray}} P_{occ}(v) $$
Where $P_{occ}(v)$ is the occupancy probability of voxel $v$.

### 2. Depth and Color Probability (Tulsiani et al.)
Tulsiani et al. [29] compute the probability that a ray stops at a certain distance, allowing rendering of depth maps, color images, and foreground masks.
1. Cast a ray.
2. For each voxel $v$ at distance $d_v$ with opacity $\alpha_v$, compute the probability that the ray terminates at $v$.
3. Aggregate color and depth weighted by these termination probabilities.
This method extends simple max-aggregation by modeling the ray as a probabilistic event stopping at a specific depth.

### 3. Emission-Absorption Models (Henzler et al.)
Henzler et al. [30] introduce bidirectional emission-absorption (EA) light radiation models, in addition to visual hull (VH) and absorption-only (AO) models similar to Tulsiani et al.
- **AO:** Standard volume rendering where light is absorbed.
- **EA:** Voxels can emit light, requiring integration of emission and absorption along the ray.
$$ C_{out} = \int_{t_n}^{t_f} \sigma(t) c(t) e^{-\int_{t_n}^{t} \sigma(s) ds} dt + C_{bg} e^{-\int_{t_n}^{t_f} \sigma(s) ds} $$
Where $\sigma$ is density, $c$ is color/emission, and $C_{bg}$ is background color.

### 4. Differentiable Ray Marching (Lombardi et al.)
Lombardi et al. [31] iteratively accumulate color and opacity probabilities.
1. Cast a ray.
2. Iterate through voxels $v_i$ along the ray.
3. Accumulate color $C$ and opacity $\alpha$:
   $$ C_{accum} = \sum_{i} c_i \alpha_i \prod_{j < i} (1 - \alpha_j) $$
   $$ \alpha_{accum} = \sum_{i} \alpha_i \prod_{j < i} (1 - \alpha_j) $$
4. **Pitfall/Guard:** Discard remaining voxels when cumulative opacity $\alpha_{accum}$ reaches a maximum threshold (e.g., 1.0) to prevent further impact on color and save computation.

### 5. SDF Surface Extraction (Jiang et al.)
Jiang et al. [32] handle voxels representing Signed Distance Functions (SDF).
1. Cast a ray.
2. Locate the surface voxel $v_s$ closest to the camera (where SDF value is near zero).
   - *Note:* Locating the surface point is not a differentiable operation [S1, unofficial].
3. Compute final pixel color as a weighted sum of the surface voxel and its neighboring voxels.
4. Compute normal vector $\mathbf{n}$ at the surface point based on voxel values around the surface point for shading.
   - *Note:* Although surface location is non-differentiable, the pixel color is differentiable with respect to the voxels around the surface point [S1, unofficial].

### Pseudocode: Differentiable Ray Marching (Lombardi et al.)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aggregate_ray_march(voxel_grid, ray_origin, ray_direction, max_steps=100, alpha_threshold=0.99):
    """
    Aggregates voxel values along a ray using iterative accumulation.
    voxel_grid: 3D array of (color, opacity) tuples.
    ray_origin: (x, y, z)
    ray_direction: (dx, dy, dz) normalized.
    """
    current_pos = ray_origin
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    transmittance = 1.0
    
    for step in range(max_steps):
        # Sample voxel at current_pos
        # Assume sample_voxel returns (color, opacity)
        color, opacity = sample_voxel(voxel_grid, current_pos)
        
        # Contribution of this voxel
        contribution = color * opacity * transmittance
        accumulated_color += contribution
        accumulated_opacity += opacity * transmittance
        
        # Update transmittance
        transmittance *= (1.0 - opacity)
        
        # Early termination if fully opaque
        if accumulated_opacity >= alpha_threshold:
            break
            
        # Move along ray
        current_pos += ray_direction * step_size
        
    return accumulated_color, accumulated_opacity
```

## Subtopics

### [StreamAgg](19823-streamagg.md)
A streaming dense voxel aggregation module that performs temporal warping of cached voxel features and aggregates them with current frame features to predict 3D occupancy. It sets itself apart by propagating dense voxel features through a recurrent streaming buffer rather than relying solely on static spatial aggregation.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels intersected by the ray. Early termination (Lombardi et al.) can reduce average case complexity.
- **Memory Complexity:** Depends on voxel grid storage. StreamAgg requires additional memory for the recurrent streaming buffer.
- **Strengths:** Differentiable with respect to voxel inputs, enabling gradient-based optimization. Flexible support for various output types (depth, color, masks).
- **Limitations:** Locating surface points (SDF methods) is non-differentiable, though gradients can be approximated via neighboring voxels. Performance depends on ray length and voxel density.
- **When to use:** Differentiable rendering pipelines, neural rendering, and optimization tasks where gradients through the volume are required.

## Background
Originating from differentiable rendering research, these methods were developed to enable gradient-based optimization of 3D representations. Key contributions include Yan et al. [7] for occupancy probability, Tulsiani et al. [29] for depth/color probability, Henzler et al. [30] for emission-absorption models, Lombardi et al. [31] for differentiable ray marching, and Jiang et al. [32] for SDF-based surface extraction.

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
