# Plenoxel Data Structure

Also called: Plenoxel, Plenoxel grid, Sparse Plenoxel storage, plenoptic voxels

## Overview
Plenoxel is an explicit, sparse 3D grid representation for photorealistic view synthesis that stores opacity and spherical harmonic (SH) coefficients directly, optimized via gradient descent without neural networks [S1]. It differs from its sibling [Differentiable Volumetric Rendering (DVR)](775-differentiable-volumetric-rendering-dvr.md) by using a discrete, sparse voxel structure with trilinear interpolation rather than a continuous implicit field, and from [PlenOctrees](https://arxiv.org/abs/2112.05131) by maintaining continuity in the plenoptic function through interpolation, which is critical for successful optimization [S1].

## How it works
**Inputs:** A set of calibrated 2D images with known camera poses [S1].
**Outputs:** A sparse voxel grid containing opacity ($\sigma$) and spherical harmonic coefficients for RGB channels, optimized to minimize reconstruction error [S1].

### Data Structures
The representation consists of two components [S1]:
1.  **Dense Pointer Grid:** A 3D array where each cell contains a pointer (or `NULL`) [S1]. The grid dimensions are determined by the scene bounds and voxel resolution [G].
2.  **Data Table:** A contiguous array storing values for occupied voxels only [S1]. Each entry contains:
    *   A scalar opacity value $\sigma$ [S1].
    *   A vector of Spherical Harmonic coefficients for the Red, Green, and Blue channels [S1].

**Layout Details:**
*   The SH coefficients are stored contiguously [S1].
*   Degree 2 SH is used, requiring 9 coefficients per color channel, totaling 27 harmonic coefficients per voxel [S1].
*   `NULL` cells in the pointer grid are treated as having zero opacity and zero SH coefficients [S1].

### Algorithm Steps

1.  **Initialization:** Initialize the dense pointer grid with `NULL` and the data table [G].
2.  **Pruning:** Remove empty voxels to manage memory [S1]. Thresholding strategies include weight-based thresholding for synthetic/360° scenes and opacity-based thresholding for forward-facing scenes [S1].
3.  **Sampling:** For each training ray, sample points along the ray path [G].
4.  **Interpolation:** For each sample point, identify the 8 surrounding voxels in the dense pointer grid [G]. Retrieve the pointers to the data table entries [S1]. If a pointer is `NULL`, treat the value as zero [S1]. Trilinearly interpolate the opacity $\sigma$ and the SH coefficients from the 8 neighboring voxel corners to obtain the values at the sample point [S1].
5.  **Color Evaluation:** Evaluate the SH basis functions at the viewing direction for the sample point using the interpolated coefficients to compute the color $\mathbf{c}_i$ for each RGB channel [S1].
6.  **Rendering:** Compute the ray color $\hat{C}(\mathbf{r})$ using the volume rendering equation [S1]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i(1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance, $\sigma_i$ is the opacity, $\delta_i$ is the distance to the next sample, and $\mathbf{c}_i$ is the color [S1].
7.  **Loss Computation:** Calculate the Mean Squared Error (MSE) between the rendered color $\hat{C}(\mathbf{r})$ and the ground truth image color [S1]. Add a Total Variation (TV) regularizer to the loss [S1].
8.  **Optimization:** Update the voxel coefficients (opacity and SH) using gradient descent (e.g., RM-SProp) [S1].
9.  **Clipping:** Clip negative color values to 0 using a ReLU to preserve linearity for trilinear interpolation, unlike NeRF which uses sigmoid [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_plenoxel(ray_origin, ray_direction, grid, data_table, grid_min, grid_max, voxel_size):
    """
    Renders a single ray using the Plenoxel data structure.
    
    Args:
        ray_origin: (x, y, z) tuple
        ray_direction: (x, y, z) tuple (normalized)
        grid: 3D array of pointers (None or index into data_table)
        data_table: List of voxel data objects
        grid_min: (min_x, min_y, min_z) tuple
        grid_max: (max_x, max_y, max_z) tuple
        voxel_size: float, size of one voxel side
        
    Returns:
        color: (r, g, b) tuple
    """
    # 1. Determine ray segment bounds within the grid
    t_min, t_max = compute_ray_grid_intersection(ray_origin, ray_direction, grid_min, grid_max)
    if t_max < 0 or t_min > t_max:
        return (0.0, 0.0, 0.0)
    
    num_samples = 100
    delta_t = (t_max - t_min) / num_samples
    accumulated_transmittance = 1.0
    accumulated_color = [0.0, 0.0, 0.0]
    
    for i in range(1, num_samples + 1):
        t = t_min + i * delta_t
        pos = (
            ray_origin[0] + t * ray_direction[0],
            ray_origin[1] + t * ray_direction[1],
            ray_origin[2] + t * ray_direction[2]
        )
        
        # 2. Trilinear Interpolation
        sigma, sh_r, sh_g, sh_b = trilinear_interpolate(pos, grid, data_table, grid_min, voxel_size)
        
        # 3. Evaluate SH for viewing direction
        r = evaluate_sh_degree2(sh_r, ray_direction)
        g = evaluate_sh_degree2(sh_g, ray_direction)
        b = evaluate_sh_degree2(sh_b, ray_direction)
        
        # 4. Clip negative colors (ReLU)
        r = max(0.0, r)
        g = max(0.0, g)
        b = max(0.0, b)
        
        # 5. Volume Rendering Step
        attenuation = math.exp(-sigma * delta_t)
        alpha = 1.0 - attenuation
        weight = accumulated_transmittance * alpha
        
        accumulated_color[0] += weight * r
        accumulated_color[1] += weight * g
        accumulated_color[2] += weight * b
        accumulated_transmittance *= attenuation
        
        # Early termination if transmittance is very low
        if accumulated_transmittance < 1e-5:
            break
            
    return tuple(accumulated_color)

def compute_ray_grid_intersection(origin, direction, grid_min, grid_max):
    """Computes intersection t-min and t-max of a ray with an axis-aligned bounding box."""
    inv_dir = (1.0/direction[0], 1.0/direction[1], 1.0/direction[2])
    t1 = (grid_min[0] - origin[0]) * inv_dir[0]
    t2 = (grid_max[0] - origin[0]) * inv_dir[0]
    t3 = (grid_min[1] - origin[1]) * inv_dir[1]
    t4 = (grid_max[1] - origin[1]) * inv_dir[1]
    t5 = (grid_min[2] - origin[2]) * inv_dir[2]
    t6 = (grid_max[2] - origin[2]) * inv_dir[2]
    t_min = max(min(t1, t2), min(t3, t4), min(t5, t6))
    t_max = min(max(t1, t2), max(t3, t4), max(t5, t6))
    return t_min, t_max

def trilinear_interpolate(pos, grid, data_table, grid_min, voxel_size):
    """Performs trilinear interpolation of sigma and SH coefficients."""
    # Map world pos to fractional grid coordinates
    fx = (pos[0] - grid_min[0]) / voxel_size
    fy = (pos[1] - grid_min[1]) / voxel_size
    fz = (pos[2] - grid_min[2]) / voxel_size
    
    x0, y0, z0 = math.floor(fx), math.floor(fy), math.floor(fz)
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    dx, dy, dz = fx - x0, fy - y0, fz - z0
    
    # Retrieve pointers from grid (8 corners)
    ptrs = []
    for x in (x0, x1):
        for y in (y0, y1):
            for z in (z0, z1):
                ptrs.append(grid[x][y][z])
                
    # Interpolate scalar opacity
    sigma = trilinear_interp_scalar(ptrs, dx, dy, dz, field='sigma')
    
    # Interpolate SH coefficient vectors (lists of 9 floats)
    sh_r = trilinear_interp_vector(ptrs, dx, dy, dz, field='sh_r')
    sh_g = trilinear_interp_vector(ptrs, dx, dy, dz, field='sh_g')
    sh_b = trilinear_interp_vector(ptrs, dx, dy, dz, field='sh_b')
    
    return sigma, sh_r, sh_g, sh_b

def trilinear_interp_scalar(ptrs, dx, dy, dz, field):
    """Trilinear interpolation for a scalar field."""
    vals = [p[field] if p is not None else 0.0 for p in ptrs]
    return (
        vals[0]*(1-dx)*(1-dy)*(1-dz) +
        vals[1]*dx*(1-dy)*(1-dz) +
        vals[2]*(1-dx)*dy*(1-dz) +
        vals[3]*dx*dy*(1-dz) +
        vals[4]*(1-dx)*(1-dy)*dz +
        vals[5]*dx*(1-dy)*dz +
        vals[6]*(1-dx)*dy*dz +
        vals[7]*dx*dy*dz
    )

def trilinear_interp_vector(ptrs, dx, dy, dz, field):
    """Trilinear interpolation for a vector field (SH coefficients)."""
    vals = [p[field] if p is not None else [0.0]*9 for p in ptrs]
    result = [0.0]*9
    for k in range(9):
        v0 = vals[0][k]; v1 = vals[1][k]; v2 = vals[2][k]; v3 = vals[3][k]
        v4 = vals[4][k]; v5 = vals[5][k]; v6 = vals[6][k]; v7 = vals[7][k]
        result[k] = (
            v0*(1-dx)*(1-dy)*(1-dz) +
            v1*dx*(1-dy)*(1-dz) +
            v2*(1-dx)*dy*(1-dz) +
            v3*dx*dy*(1-dz) +
            v4*(1-dx)*(1-dy)*dz +
            v5*dx*(1-dy)*dz +
            v6*(1-dx)*dy*dz +
            v7*dx*dy*dz
        )
    return result

def evaluate_sh_degree2(coeffs, direction):
    """Evaluates Degree 2 Spherical Harmonics for a given direction."""
    x, y, z = direction
    # Standard SH degree 2 basis functions (unnormalized convention)
    c0 = 0.282095
    c1 = 0.488603
    c2 = 1.092548
    c3 = -1.092548
    c4 = 0.315392
    # Basis vector
    b0 = c0
    b1 = c1 * y
    b2 = c1 * z
    b3 = c1 * x
    b4 = c2 * x * y
    b5 = c2 * y * z
    b6 = c4 * (3 * z * z - 1)
    b7 = c2 * x * z
    b8 = c2 * (x * x - y * y)
    basis = [b0, b1, b2, b3, b4, b5, b6, b7, b8]
    return sum(c * b for c, b in zip(coeffs, basis))
```

### Parameters
*   **SH Degree:** 2 [S1].
*   **Optimization Algorithm:** RM-SProp [S1].
*   **Batch Size:** 5000 rays [S1].
*   **Regularization:** Total Variation (TV) [S1].
*   **Color Clipping:** ReLU (clip negative values to 0) [S1].

### Edge Cases and Pitfalls
*   **Unbounded Scenes:** For forward-facing scenes, use normalized device coordinates. For 360° scenes, surround the grid with multisphere images to encode the background [S1].
*   **Performance Bottleneck:** Reading pointers from the dense grid consumes significant rendering time [S1].
*   **Pruning Strategy:** Use weight-based thresholding for synthetic/360° scenes and opacity-based thresholding for forward-facing scenes to avoid pruning sparsely-supervised features at edges [S1].
*   **Linearity:** Do not use sigmoid for color clamping; use ReLU to preserve linearity for trilinear interpolation [S1].

## Operating characteristics
*   **Training Speed:** On a single Titan RTX GPU, optimization takes 11 minutes for bounded scenes and 27 minutes for unbounded scenes, compared to roughly 1 day for NeRF [S1].
*   **Rendering Speed:** Renders novel viewpoints at interactive rates of 15 fps (unoptimized implementation) [S1].
*   **Memory:** The dense pointer grid is usually not the dominant memory consumer; the data table storing SH coefficients, gradients, and optimizer states dominates [S1].
*   **Strengths:** Two orders of magnitude faster optimization than NeRF with no loss in visual quality; explicit representation allows for easy pruning and conversion to other formats (e.g., PlenOctrees) [S1].
*   **Limitations:** Rendering speed is limited by pointer chasing; requires calibrated images; not suitable for dynamic scenes without extension [S1].

## Background
Plenoxels were introduced by Alex Yu, Sara Fridovich-Keil, Matthew Tancik, Qinhong Chen, Benjamin Recht, and Angjoo Kanazawa from UC Berkeley in 2022 [S1]. The work demonstrates that photorealistic volumetric reconstruction can be achieved using standard tools from inverse problems (data representation, forward model, regularization, optimizer) without neural networks, revealing that the key element of NeRF is the differentiable volumetric renderer rather than the neural network itself [S1]. It extends the concept of PlenOctrees by using a sparse voxel grid with trilinear interpolation [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
