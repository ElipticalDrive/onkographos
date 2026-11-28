# Windowed Sinc Interpolation

Also called: Hamming-windowed sinc interpolation, Sinc interpolation with window function, Sinc interpolation with window function and z-direction blur, Windowed sinc resampling

## Overview
Windowed Sinc Interpolation approximates a continuous volume function by convolving the discrete voxel samples with a truncated sinc kernel modulated by a window function, primarily to reduce aliasing artifacts inherent in ideal sinc interpolation while maintaining high frequency response [S1]. It is distinguished from its siblings by its explicit support for anisotropic voxel resampling via a directional blur factor, making it particularly suitable for converting anisotropic medical imaging data (e.g., breast tomosynthesis) into isotropic grids for volume rendering [S1].

## How it works
**Inputs and Outputs**
*   **Input**: Discrete volume data $V$ defined on a grid with voxel dimensions $\Delta_x, \Delta_y, \Delta_z$, target grid coordinates, window function type $W$, Window Half-Width $WHW$, and Blur Factor $BF(z)$ [S1].
*   **Output**: Resampled volume data $V'$ on the target grid (typically isotropic) [S1].

**Data Structures**
The method operates on standard 3D volume arrays [G].

**Algorithm**
1.  **Coordinate Mapping**: For each target voxel center $\mathbf{p}_{target}$ in the output grid, map the coordinates to the input volume's local coordinate system [S1, Parent].
2.  **Kernel Weight Calculation**: Compute the interpolation weights for all input voxels $\mathbf{p}_i$ within the search radius $WHW$ of $\mathbf{p}_{target}$ [S1].
    *   Calculate the distance vector $\mathbf{d}_i = \mathbf{p}_{target} - \mathbf{p}_i$ [G].
    *   Compute the base sinc weight for each dimension $k \in \{x, y, z\}$:
        $$ w_{sinc, k} = \text{sinc}\left(\frac{d_{i,k}}{\Delta_k}\right) $$
        where $\Delta_k$ is the input voxel size in dimension $k$, and $\text{sinc}(x) = \frac{\sin(\pi x)}{\pi x}$ [S1, Parent].
    *   Apply the window function $W(\cdot)$ to truncate the kernel. For a 1D distance $u$ normalized by $WHW$:
        $$ w_{win, k} = W\left(\frac{d_{i,k}}{WHW \cdot \Delta_k}\right) $$
        [S1].
    *   Apply the z-direction blur factor $BF(z)$ if the input is anisotropic. This modulates intensity fluctuations along the slice thickness axis [S1]. The exact mathematical modulation is not specified in the sources, but it acts as a scaling factor or width modifier for the z-component weight [G].
    *   Combine weights:
        $$ w_i = w_{win, x} \cdot w_{win, y} \cdot w_{win, z} \cdot w_{sinc, x} \cdot w_{sinc, y} \cdot w_{sinc, z} \cdot BF(z) $$
        [S1, G].
3.  **Accumulation**: Sum the weighted contributions of all neighboring voxels:
    $$ V(\mathbf{p}_{target}) = \sum_{i \in \text{neighbors}} w_i V_i $$
    [S1].
4.  **Normalization**: Normalize the result by the sum of weights if the kernel is not normalized by construction [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def windowed_sinc_interpolation(volume, target_grid, voxel_sizes, whw=5, bf_z=2.0, window_func='hamming'):
    """
    Resample an anisotropic volume to a target grid using windowed sinc interpolation.
    
    Args:
        volume: 3D list of lists of lists of voxel values.
        target_grid: List of (x, y, z) tuples representing target coordinates in volume space.
        voxel_sizes: Tuple (dx, dy, dz) for input voxel dimensions.
        whw: Window Half-Width in voxels.
        bf_z: Blur Factor for the z-direction.
        window_func: Name of the window function (e.g., 'hamming').
        
    Returns:
        List of resampled voxel values corresponding to target_grid.
    """
    dx, dy, dz = voxel_sizes
    output_values = []

    for p_target in target_grid:
        x, y, z = p_target
        x_min = max(0, math.floor(x - whw * dx))
        x_max = min(len(volume) - 1, math.ceil(x + whw * dx))
        y_min = max(0, math.floor(y - whw * dy))
        y_max = min(len(volume[0]) - 1, math.ceil(y + whw * dy))
        z_min = max(0, math.floor(z - whw * dz))
        z_max = min(len(volume[0][0]) - 1, math.ceil(z + whw * dz))
        
        sum_weights = 0.0
        sum_values = 0.0
        
        for i in range(x_min, x_max + 1):
            for j in range(y_min, y_max + 1):
                for k in range(z_min, z_max + 1):
                    dist_x = i * dx - x
                    dist_y = j * dy - y
                    dist_z = k * dz - z
                    
                    sinc_x = 1.0 if abs(dist_x) < 1e-9 else math.sin(math.pi * dist_x / dx) / (math.pi * dist_x / dx)
                    sinc_y = 1.0 if abs(dist_y) < 1e-9 else math.sin(math.pi * dist_y / dy) / (math.pi * dist_y / dy)
                    sinc_z = 1.0 if abs(dist_z) < 1e-9 else math.sin(math.pi * dist_z / dz) / (math.pi * dist_z / dz)
                    
                    norm_x = dist_x / (whw * dx)
                    norm_y = dist_y / (whw * dy)
                    norm_z = dist_z / (whw * dz)
                    
                    win_x = apply_window(norm_x, window_func)
                    win_y = apply_window(norm_y, window_func)
                    win_z = apply_window(norm_z, window_func)
                    
                    # Note: Exact mathematical modulation of bf_z is not specified in sources [S1]; 
                    # implemented here as a z-direction amplitude scaling [G].
                    blur_weight = bf_z if abs(dist_z) > 0 else 1.0
                    
                    weight = win_x * win_y * win_z * sinc_x * sinc_y * sinc_z * blur_weight
                    
                    sum_weights += weight
                    sum_values += weight * volume[i][j][k]
        
        output_values.append(sum_values / sum_weights if sum_weights > 0 else 0.0)
        
    return output_values

def apply_window(u, func_name):
    """Apply window function. Only Hamming is explicitly tested in sources [S1]."""
    if abs(u) > 1.0:
        return 0.0
    if func_name == 'hamming':
        return 0.54 - 0.46 * math.cos(2 * math.pi * u)
    # Other windows (Lanczos, Kaiser, etc.) are mentioned but not defined in sources [S1]
    return 1.0 # Fallback
```

**Parameters and Typical Values**
*   $WHW$: Default in VTK is 3. Values up to 5 show visible improvement in smoothness; values $>5$ increase time without noticeable quality gain [S1]. Typical optimal value cited: 5 [S1].
*   $BF(z)$: Default in VTK is 1.0. Increasing to 2.0 or 2.5 improves smoothness and decreases FWHM (structure delineation) but increases time [S1]. Typical optimal value cited: 2.0 [S1].
*   Window Function: Hamming is used in the cited study, though Lanczos, Kaiser, Cosine, Hann, Blackman, and Nuttall are also viable [S1].

**Edge Cases and Pitfalls**
*   **Precision**: The sinc function requires careful handling at $x=0$ to avoid division by zero [G].
*   **Boundaries**: Clamping or mirroring is required when accessing voxels outside the volume bounds during the search window [G].
*   **Anisotropy**: If $BF(z)$ is not tuned for anisotropic data (e.g., $1.0$ mm slice thickness vs $0.085$ mm in-plane), artifacts or poor resolution may persist [S1].

## Operating characteristics
*   **Time Complexity**: Not given in the sources. Rendering time for constant sampling distance ranges from 0.29 s (nearest) to 0.37 s (Nuttall) [S1].
*   **Memory Complexity**: Not given in the sources. The method operates on standard 3D volume arrays [G].
*   **Performance**: Increasing voxel count from ~8M to ~95M has a greater impact on rendering time than the interpolation method itself [S1].
*   **Strengths**: Higher smoothness and lower Full Width at Half Maximum (FWHM) compared to nearest, linear, and cubic interpolators [S1]. Effective for resampling anisotropic voxels to isotropic grids [S1].
*   **Limitations**: Increased interpolation time with higher $WHW$ and $BF(z)$ values [S1]. Requires tuning of $WHW$ and $BF(z)$ for optimal results [S1].

## Background
Windowed Sinc Interpolation is implemented in the Visualization Toolkit (VTK) via the `vtkImageInterpolator` class [S1]. It has been specifically applied to optimize breast tomosynthesis (DBT) volume rendering by resampling anisotropic data ($0.085 \times 0.085 \times 1.0$ mm³) to isotropic grids ($0.085 \times 0.085 \times 0.085$ mm³) [S1]. The method addresses the trade-off between aliasing (from truncation) and ringing (from ideal sinc) by using window functions like Hamming [S1].

## Sources
[S1] Optimization of Breast Tomosynthesis Visualization through 3D Volume Rendering, https://doi.org/10.3390/jimaging6070064, fetched 2026-10-26
