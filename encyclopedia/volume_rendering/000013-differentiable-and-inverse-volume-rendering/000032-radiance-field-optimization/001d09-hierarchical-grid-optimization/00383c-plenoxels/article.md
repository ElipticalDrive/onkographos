# Plenoxels

Also called: Radiance Fields without Neural Networks

## Overview
Plenoxels is an explicit radiance field representation that replaces the neural network used in NeRF with a sparse 3D voxel grid storing density and spherical harmonic (SH) coefficients [S1, S2, S4, S5]. It solves the computational bottleneck of neural network inference during rendering and optimization, enabling training speeds two orders of magnitude faster than NeRF and real-time rendering capabilities [S1][S6]. Unlike implicit representations, Plenoxels relies on direct data access and trilinear interpolation, trading increased memory consumption for significant gains in efficiency [S1][S8, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ calibrated images with known camera poses (intrinsics and extrinsics). Optional depth priors can be used for initialization/pruning [S2].
*   **Outputs:** Optimized voxel grids for density ($V^D$) and SH coefficients ($V^{SH}$), and rendered novel views.

**Data Structures:**
The scene is represented by two sparse voxel grids, $V^D$ and $V^{SH}$ [S2].
*   **Grid Layout:** The grid is an explicit 3D tensor [S4]. For implementation, it is often stored as a flat array with stride-based indexing [S6][S8, unofficial].
*   **Voxel Content:** Each voxel vertex stores 28 float values [S1][S8, unofficial]:
    *   1 float for density $\sigma$.
    *   9 floats for the Red channel SH coefficients ($k_{\ell m}^R$) [S1].
    *   9 floats for the Green channel SH coefficients ($k_{\ell m}^G$) [S1].
    *   9 floats for the Blue channel SH coefficients ($k_{\ell m}^B$) [S1].
    *   *Note:* The 9 coefficients correspond to Spherical Harmonics of degree $\ell_{max}=2$ (orders $m=-\ell \dots \ell$ for $\ell=0,1$) [S4][S8, unofficial].
*   **Bounds:** The grid typically spans the normalized space $[-1, 1]^3$ [S8, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Initialize $V^D$ and $V^{SH}$ with zeros or small random values [G].
    *   If using depth priors (e.g., from a foundation model like VGGT), prune voxels in free space before training to implicitly regularize geometry [S2]. Apply voxel dilation to prevent over-pruning [S2].

2.  **Ray Generation:**
    *   For each pixel $(x, y)$ in the target image of width $W$ and height $H$:
        *   Compute aspect ratio $AR = W/H$.
        *   Compute focal length $f = 2 \tan(FOV/2)$ [S8, unofficial].
        *   Compute normalized screen coordinates $u = (x + 0.5)/W$, $v = (y + 0.5)/H$ [S8, unofficial].
        *   Compute ray direction in camera space:
            $$ x_{cam} = \text{lerp}(-w/2, w/2, u) $$
            $$ y_{cam} = \text{lerp}(h/2, -h/2, v) $$
            $$ z_{cam} = -1 $$
        *   Normalize the direction vector $\mathbf{d}_{cam}$ and transform it to world space using the camera extrinsic matrix [S8, unofficial]. Transform the ray origin $\mathbf{o}$ similarly [S8, unofficial].

3.  **Ray Marching (Forward Pass):**
    *   Compute ray-grid intersection to find entry ($t_{min}$) and exit ($t_{max}$) distances using the slab method with reciprocal ray directions [S8, unofficial].
    *   Discretize the segment $[t_{min}, t_{max}]$ into $K$ steps (e.g., $K=100$) with step size $\delta = (t_{max} - t_{min})/K$ [S8, unofficial].
    *   For each sample $i$ at distance $t_i$ along the ray:
        *   **Position:** $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$.
        *   **Density Sampling:**
            *   Compute integer lattice coordinates and fractional offsets [S6].
            *   Perform trilinear interpolation of density from $V^D$ using the 8 neighboring voxels [S1][S6].
            *   Apply ReLU to ensure non-negativity: $\sigma_i = \text{ReLU}(\text{interp}(\mathbf{x}_i, V^D))$ [S2].
            *   If $\sigma_i < \sigma_{thresh}$, skip color sampling (early termination optimization) [S6].
        *   **Color Sampling:**
            *   Perform trilinear interpolation of SH coefficients $\mathbf{k}_i$ from $V^{SH}$ for each RGB channel [S2][S6].
            *   Evaluate Spherical Harmonics basis functions $\mathbf{Y}(\mathbf{d})$ for the view direction $\mathbf{d}$ [S8, unofficial].
            *   Compute raw color: $\mathbf{c}_{raw} = \sum_{\ell=0}^{\ell_{max}} \sum_{m=-\ell}^{\ell} \mathbf{k}_{\ell m} Y_{\ell}^m(\mathbf{d})$ [S2].
            *   Apply sigmoid activation to the weighted sum of SH coefficients to clamp color to $[0, 1]$: $\mathbf{c}_i = S(\mathbf{c}_{raw})$, where $S(x) = (1 + \exp(-x))^{-1}$ [S2, S4].
        *   **Accumulation:**
            *   Compute transmittance $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1, S3].
            *   Compute weight $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S1, S3].
            *   Accumulate color: $\hat{C} = \sum_{i=1}^N w_i \mathbf{c}_i$ [S1, S3].
            *   Accumulate alpha: $\hat{\alpha} = \sum_{i=1}^N w_i$ [S1, S3].
            *   **Early Termination:** If $T_i < \text{stop\_thresh}$, break the loop [S6].
            *   **Background:** If no background layers are defined, add $T_N \cdot \text{background\_brightness}$ to the final color [S6].

4.  **Optimization (Backward Pass):**
    *   Compute loss $\mathcal{L} = \mathcal{L}_{MSE} + \lambda_{TV} (\mathcal{L}_{TV}(V^D) + \mathcal{L}_{TV}(V^{SH}))$ [S2].
    *   $\mathcal{L}_{MSE} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r}} \| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \|_2^2$ [S2].
    *   $\mathcal{L}_{TV}$ is the Total Variation loss, computed as the square root of the sum of squared differences between adjacent voxels for density and each SH coefficient [S2].
    *   Compute gradients via automatic differentiation or analytic inversion [S5].
    *   **Transmittance-based Gradient Scaling:** To reduce near-camera floaters, scale gradients by transmittance to counteract the exponential decay of gradients for samples far from the camera [S2].
    *   Update voxel values using an optimizer (e.g., Adam) [S6].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def evaluate_sh_basis(direction):
    """
    Evaluates Spherical Harmonics basis functions of degree 2 (9 coefficients)
    for a given normalized direction vector d = (x, y, z).
    Returns array of 9 floats.
    """
    x, y, z = direction
    basis = np.zeros(9)
    basis[0] = 0.28209479177387814
    basis[1] = 0.488602511902919923 * z
    basis[2] = -0.488602511902919978 * x
    basis[3] = -0.488602511902919978 * y
    basis[4] = 0.94617469575756008 * z**2 - 0.315391565252520045
    basis[5] = -1.09254843059207896 * z * x
    basis[6] = -1.09254843059207896 * z * y
    basis[7] = 0.546274215296039478 * (x**2 - y**2)
    basis[8] = 0.546274215296039478 * 2 * x * y
    return basis

def trilinear_interpolate(grid, pos_grid, stride, size):
    """
    Performs trilinear interpolation on a 3D grid.
    grid: flat array of voxel data.
    pos_grid: continuous position in grid coordinates [0, size-1].
    stride: (stride_x, stride_y, stride_z) for 3D indexing in flat array.
    size: (size_x, size_y, size_z) of the grid.
    Returns interpolated value.
    """
    x, y, z = pos_grid
    ix = int(np.floor(x))
    iy = int(np.floor(y))
    iz = int(np.floor(z))
    fx = x - ix
    fy = y - iy
    fz = z - iz
    
    # Clamp indices to valid range for interpolation
    ix = np.clip(ix, 0, size[0] - 2)
    iy = np.clip(iy, 0, size[1] - 2)
    iz = np.clip(iz, 0, size[2] - 2)
    
    def get_idx(i, j, k):
        return i * stride[0] + j * stride[1] + k * stride[2]
        
    c000 = grid[get_idx(ix, iy, iz)]
    c100 = grid[get_idx(ix+1, iy, iz)]
    c010 = grid[get_idx(ix, iy+1, iz)]
    c110 = grid[get_idx(ix+1, iy+1, iz)]
    c001 = grid[get_idx(ix, iy, iz+1)]
    c101 = grid[get_idx(ix+1, iy, iz+1)]
    c011 = grid[get_idx(ix, iy+1, iz+1)]
    c111 = grid[get_idx(ix+1, iy+1, iz+1)]
    
    c00 = c000 * (1 - fx) + c100 * fx
    c01 = c010 * (1 - fx) + c110 * fx
    c10 = c001 * (1 - fx) + c101 * fx
    c11 = c011 * (1 - fx) + c111 * fx
    
    c0 = c00 * (1 - fy) + c01 * fy
    c1 = c10 * (1 - fy) + c11 * fy
    
    return c0 * (1 - fz) + c1 * fz

def render_plenoxel(ray_origin, ray_direction, grid_density, grid_sh, stride, size, num_samples=100):
    """
    Renders a single ray using Plenoxels representation.
    grid_sh: flat array storing [R0..R8, G0..G8, B0..B8] per voxel (27 floats).
    """
    min_point = np.array([-1.0, -1.0, -1.0])
    max_point = np.array([1.0, 1.0, 1.0])
    
    # Correct slab method for any direction sign
    t1 = (min_point - ray_origin) / ray_direction
    t2 = (max_point - ray_origin) / ray_direction
    t_min = np.max(np.minimum(t1, t2))
    t_max = np.min(np.maximum(t1, t2))
    
    if t_min > t_max:
        return np.array([0.0, 0.0, 0.0]), 0.0
        
    delta_t = (t_max - t_min) / num_samples
    accumulated_color = np.zeros(3)
    accumulated_alpha = 0.0
    transmittance = 1.0
    
    for i in range(num_samples):
        t = t_min + i * delta_t
        pos_world = ray_origin + t * ray_direction
        
        # Convert world position to grid coordinates [0, size-1]
        pos_grid = (pos_world - min_point) / (max_point - min_point) * (np.array(size) - 1)
        
        # Sample density
        sigma = trilinear_interpolate(grid_density, pos_grid, stride, size)
        sigma = np.maximum(sigma, 0.0)
        
        # Sample SH coefficients for each channel
        # grid_sh layout: [R0..R8, G0..G8, B0..B8] per voxel
        # We interpolate each channel separately using the same grid logic
        k_r = trilinear_interpolate(grid_sh[0:9], pos_grid, stride, size)
        k_g = trilinear_interpolate(grid_sh[9:18], pos_grid, stride, size)
        k_b = trilinear_interpolate(grid_sh[18:27], pos_grid, stride, size)
        
        # Evaluate SH Basis
        basis = evaluate_sh_basis(ray_direction)
        
        # Compute raw color
        c_r_raw = np.dot(k_r, basis)
        c_g_raw = np.dot(k_g, basis)
        c_b_raw = np.dot(k_b, basis)
        
        # Apply Sigmoid
        c_r = 1.0 / (1.0 + np.exp(-c_r_raw))
        c_g = 1.0 / (1.0 + np.exp(-c_g_raw))
        c_b = 1.0 / (1.0 + np.exp(-c_b_raw))
        
        color = np.array([c_r, c_g, c_b])
        
        # Accumulate
        weight = transmittance * (1.0 - np.exp(-sigma * delta_t))
        
        accumulated_color += weight * color
        accumulated_alpha += weight
        
        # Update transmittance
        transmittance *= np.exp(-sigma * delta_t)
        
        # Early termination
        if transmittance < 1e-5:
            break
            
    return accumulated_color, accumulated_alpha
```

**Parameters:**
*   **Grid Resolution:** $128 \times 128 \times 128$ for NeRF-Synthetic and TanksAndTemple; $512 \times 512 \times 128$ for LLFF [S4].
*   **SH Degree:** 2 (9 coefficients per channel) [S4][S8, unofficial].
*   **Ray Marching Steps:** 100 partitions along the ray segment within grid bounds [S8, unofficial].
*   **Sigma Threshold:** Minimum density to proceed to color sampling (optimization) [S6].
*   **Stop Threshold:** Transmission threshold for early termination [S6].
*   **Background Brightness:** Constant value added if no background layers are defined [S6].

**Edge Cases and Pitfalls:**
*   **Memory Footprint:** Performance depends excessively on spatial resolution, leading to enormous memory consumption compared to neural network approaches [S4][S6].
*   **Bank Conflicts:** In GPU implementations, accessing SH basis values can cause shared memory bank conflicts [S6].
*   **Clamping:** The sigmoid activation affects gradient computation; the gradient must account for the clamping region [S6].
*   **Ray Indexing:** When generating rays for batched images, pixel indices must be correctly mapped to tensor dimensions (y then x) [S7, unofficial].

## Subtopics

### F2Plenoxels
A variant for few-shot view synthesis that uses depth priors from foundation models to prune voxels and applies transmittance-based gradient scaling to reduce floaters. [F2Plenoxels](929-plenoxels-optimization-configuration.md)

### PlenOctree
A hierarchical extension of Plenoxels that uses an octree structure to reduce memory footprint and improve rendering speed [S2]. [PlenOctree Optimization](1291-plenoctree-optimization.md)

### Sparse Voxel Grid Optimization
General techniques for optimizing sparse voxel representations, including pruning and regularization. [Sparse Voxel Grid Optimization](2568-sparse-voxel-grid-optimization.md)

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ where $N$ is the number of samples per ray, dominated by trilinear interpolation and SH evaluation, which are significantly faster than MLP inference [S2][S8, unofficial].
*   **Memory Complexity:** $O(R^3)$ where $R$ is the grid resolution, as it stores explicit values for every voxel in the bounding box [S4]. This is a major limitation compared to implicit methods [S1].
*   **Performance:** On an Intel Core i7-7700HQ and GTX 1060 at 1024x768, single-threaded CPU rendering took ~7.8s, while OpenMP (4 threads) took ~1.4s [S8, unofficial]. Training is two orders of magnitude faster than NeRF [S1].
*   **Strengths:** Fast training and rendering, real-time visualization possible, explicit geometry representation [S1][S8, unofficial].
*   **Limitations:** High memory usage for high-resolution scenes, fixed resolution limits detail (aliasing artifacts at high zoom) [S4][S6].

## Background
Plenoxels was introduced by Fridovich-Keil et al. in 2022 [S6]. It was proposed as a direct replacement for the neural network in NeRF, leveraging the explicit nature of voxel grids to achieve significant speedups [S1]. The method builds on the idea that radiance fields can be represented analytically using spherical harmonics for view-dependent color and trilinear interpolation for density [S1, S2]. It has been widely adopted as a baseline for fast radiance field methods and extended into hierarchical structures like PlenOctrees [S6].

## Sources
[S1] RGB-D Mapping and Tracking in a Plenoxel Radiance Field, https://doi.org/10.1109/wacv57701.2024.00331, fetched 2026-10-26
[S2] F2Plenoxels: fast voxel radiance fields without neural networks for few-shot view synthesis, https://doi.org/10.1007/s44443-026-00491-2, fetched 2026-10-26
[S3] PeRFception: Perception using Radiance Fields, https://doi.org/10.48550/arxiv.2208.11537, fetched 2026-10-26
[S4] One Is All: Bridging the Gap between Neural Radiance Fields Architectures with Progressive Volume Distillation, https://doi.org/10.1609/aaai.v37i1.25135, fetched 2026-10-26
[S5] Progressive Volume Distillation with Active Learning for Efficient NeRF Architecture Conversion, https://doi.org/10.1007/s11263-026-02815-1, fetched 2026-10-26
[S6] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27
[S7] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27 (unofficial)
[S8] RevelcoS/Plenoxels, https://github.com/RevelcoS/Plenoxels, fetched 2026-10-27 (unofficial)
[S9] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
