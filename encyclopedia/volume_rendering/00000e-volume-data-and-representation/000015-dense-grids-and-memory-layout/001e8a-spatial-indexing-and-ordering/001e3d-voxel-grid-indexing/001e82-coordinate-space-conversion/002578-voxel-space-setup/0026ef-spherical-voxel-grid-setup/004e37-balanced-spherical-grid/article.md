# Balanced Spherical Grid

Also called: Balanced Spherical Feature Grid, EgoNeRF grid, Yin and Yang grid

## Overview
The Balanced Spherical Grid is a neural volume representation designed for egocentric omnidirectional view synthesis that replaces Cartesian grids to solve the inefficiency of uniform resolution in large-scale unbounded scenes [S1]. It avoids polar singularities by combining two offset quasi-uniform spherical grids (Yin and Yang) and handles unbounded depth by partitioning the radial dimension exponentially and placing an environment map at infinity [S1]. Unlike Cartesian feature grids which suffer from non-uniform ray-grid hits and wasted memory on distant objects, this method aligns with egocentric rays and enables factorization for performance enhancement [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of egocentric panoramic images with corresponding camera parameters (position and orientation) [S1].
*   **Outputs:** A radiance field representation defined by a density grid $G_\sigma$ and an appearance grid $G_a$, allowing the computation of volume density $\sigma(\mathbf{x})$ and color $c(\mathbf{x}, \mathbf{d})$ for any 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S1].

**Data Structures**
The representation consists of two grids, $G_\sigma$ and $G_a$, built upon the balanced spherical coordinate system [S1].

1.  **Density Grid ($G_\sigma$):**
    *   **Layout:** $\mathbb{R}^{2N_y^r \times N_y^\theta \times N_y^\phi}$ [S1].
    *   **Content:** Single channel storing explicit volume density values [S1].
    *   **Dimensions:**
        *   Radial: $2N_y^r$ (combining Yin and Yang radial partitions) [S1].
        *   Angular: $N_y^\theta \times N_y^\phi$ (partitions for colatitude and longitude) [S1].

2.  **Appearance Grid ($G_a$):**
    *   **Layout:** $\mathbb{R}^{2N_y^r \times N_\theta^y \times N_\phi^y \times C}$ [S1].
    *   **Content:** $C$-dimensional neural appearance features [S1].
    *   **Dimensions:** Same spatial dimensions as $G_\sigma$, plus the feature dimension $C$ [S1].

**Coordinate System and Grid Construction**
The grid uses spherical coordinates $(r, \theta, \phi)$ where $\theta$ is colatitude and $\phi$ is longitude [S1]. To avoid singularities at the poles found in naive spherical grids, the angular domain is covered by two overlapping grids: Yin and Yang [S1].

1.  **Yin Grid Angular Domain:**
    Defined by the ranges:
    $$ \pi/4 \leq \theta \leq 3\pi/4 $$
    $$ -3\pi/4 \leq \phi \leq 3\pi/4 $$
    [S1]

2.  **Yang Grid Angular Domain:**
    The Yang grid is an offset version of the Yin grid. Its axis is located at the equator of the Yin grid. The coordinate transformation from Yang to Yin is defined by the matrix $M$:
    $$ \begin{bmatrix} x^{\text{Yin}} \\ y^{\text{Yin}} \\ z^{\text{Yin}} \end{bmatrix} = M \begin{bmatrix} x^{\text{Yang}} \\ y^{\text{Yang}} \\ z^{\text{Yang}} \end{bmatrix}, \quad M = \begin{bmatrix} -1 & 0 & 0 \\ 0 & 0 & 1 \\ 0 & 1 & 0 \end{bmatrix} $$
    [S1]

3.  **Angular Partition Sizes:**
    For a grid $y \in \{\text{Yin, Yang}\}$, the partition sizes are uniform in angles:
    $$ \Delta \theta^y = \frac{\pi}{2} \frac{1}{N_\theta^y} $$
    $$ \Delta \phi^y = \frac{3\pi}{2} \frac{1}{N_\phi^y} $$
    where $N_\theta^y$ and $N_\phi^y$ are the number of partitions for $\theta$ and $\phi$ respectively [S1].

4.  **Radial Partitions:**
    The radial direction is partitioned exponentially to cover large spaces and maintain similar cell lengths in angular and radial directions [S1].
    *   **Formula:**
        $$ r_i^y = r_0 k^{i-1} $$
        $$ r_{\max} = r_0 k^{N_r^y-1} $$
    *   **Parameters:**
        *   $r_0$: Radius of the first spherical shell. For distances less than $r_0$, the grid interval is set to $r_0$ [S1].
        *   $k$: Exponential growth constant [S1].
        *   $N_r^y$: Number of radial partitions for grid $y$ [S1].
        *   $R_{\max}$: Radius of the scene bounding sphere, limiting the size of the environment representation [S1].

5.  **Environment Map:**
    An environment map $E \in \mathbb{R}^{H \times W \times 3}$ is placed at infinite depth to represent unbounded scenes (e.g., skies) [S1]. The spherical grid remains bounded by $R_{\max}$ [S1].

**Radiance Field Evaluation**
The volume density and color are obtained via trilinear interpolation $\mathcal{T}$ from the grids and a small MLP $f_{\text{MLP}}$ [S1]:
$$ \sigma(\mathbf{x}) = \mathcal{T}(G_\sigma, \mathbf{x}) $$
$$ c(\mathbf{x}, \mathbf{d}) = f_{\text{MLP}}(\mathcal{T}(G_a, \mathbf{x}), \mathbf{d}) $$
[S1]

**Hierarchical Sampling**
To efficiently sample sparse occupied regions without allocating additional memory for a coarse grid, the method exploits the dense geometry feature grid $G_\sigma$ [S1]:
1.  **Coarse Sampling:** Sample $N_c$ points along the ray.
2.  **Coarse Density Estimation:** Obtain density $\sigma$ from a coarser version of $G_\sigma$. This coarser grid is obtained on-the-fly by applying a non-learnable convolution to $G_\sigma$ [S1].
3.  **Fine Sampling:** Use importance sampling based on the coarse density estimate to sample $N_f$ fine points [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def get_balanced_spherical_grid_params(N_theta_yin, N_phi_yin, N_r_yin, r0, k, R_max):
    """
    Calculate grid parameters for Yin and Yang grids.
    """
    # Angular partition sizes for Yin grid
    delta_theta_yin = (np.pi / 2) / N_theta_yin
    delta_phi_yin = (3 * np.pi / 2) / N_phi_yin
    
    # Yang grid uses same partition sizes as Yin (implied by symmetry and minimal overlap)
    delta_theta_yang = delta_theta_yin
    delta_phi_yang = delta_phi_yin
    
    # Radial partitions for Yin grid
    # r_i = r0 * k^(i-1)
    # Max radius check: r0 * k^(N_r - 1) should be <= R_max
    radial_indices_yin = np.arange(1, N_r_yin + 1)
    radial_scales_yin = r0 * (k ** (radial_indices_yin - 1))
    
    # Yang grid radial partitions are identical in structure
    radial_scales_yang = radial_scales_yin
    
    return {
        'delta_theta_yin': delta_theta_yin,
        'delta_phi_yin': delta_phi_yin,
        'delta_theta_yang': delta_theta_yang,
        'delta_phi_yang': delta_phi_yang,
        'radial_scales_yin': radial_scales_yin,
        'radial_scales_yang': radial_scales_yang,
        'R_max': R_max
    }

def transform_yang_to_yin_coords(x_yang, y_yang, z_yang):
    """
    Transform coordinates from Yang grid basis to Yin grid basis.
    M = [[-1, 0, 0], [0, 0, 1], [0, 1, 0]]
    """
    x_yin = -x_yang
    y_yin = z_yang
    z_yin = y_yang
    return x_yin, y_yin, z_yin

def spherical_to_cartesian(r, theta, phi):
    """
    Convert spherical coordinates to Cartesian.
    theta: colatitude, phi: longitude
    """
    x = r * np.sin(theta) * np.cos(phi)
    y = r * np.sin(theta) * np.sin(phi)
    z = r * np.cos(theta)
    return x, y, z

def get_grid_index_for_point(x, y, z, grid_params, grid_type='Yin'):
    """
    Determine which grid cell (theta, phi, r) a point falls into.
    Returns indices or None if outside bounds.
    """
    # Convert to spherical
    r = np.sqrt(x**2 + y**2 + z**2)
    theta = np.arccos(z / r) if r > 0 else 0
    phi = np.arctan2(y, x)
    
    # Handle phi range [-pi, pi] to match grid definition if necessary
    # Grid phi range is -3pi/4 to 3pi/4
    
    if grid_type == 'Yin':
        # Yin bounds
        if not (np.pi/4 <= theta <= 3*np.pi/4):
            return None
        if not (-3*np.pi/4 <= phi <= 3*np.pi/4):
            return None
        
        # Calculate indices
        # theta range is pi/2 (pi/4 to 3pi/4)
        # phi range is 3pi/2 (-3pi/4 to 3pi/4)
        
        idx_theta = int((theta - np.pi/4) / grid_params['delta_theta_yin'])
        idx_phi = int((phi - (-3*np.pi/4)) / grid_params['delta_phi_yin'])
        
        # Radial index
        # Find i such that r_i <= r < r_{i+1}
        # r_i = r0 * k^(i-1)
        # If r < r0, interval is r0.
        if r < grid_params['radial_scales_yin'][0]:
            idx_r = 0
        else:
            # Solve r = r0 * k^(i-1) => i-1 = log(r/r0)/log(k) => i = 1 + log(r/r0)/log(k)
            # Use floor for lower bound index
            idx_r = int(1 + np.log(r / grid_params['radial_scales_yin'][0]) / np.log(grid_params['radial_scales_yin'][1] / grid_params['radial_scales_yin'][0]))
            
            # Clamp to max
            if idx_r >= len(grid_params['radial_scales_yin']):
                idx_r = len(grid_params['radial_scales_yin']) - 1
                
        return (idx_r, idx_theta, idx_phi)
        
    elif grid_type == 'Yang':
        # Transform point to Yin basis
        x_yin, y_yin, z_yin = transform_yang_to_yin_coords(x, y, z)
        # Recalculate spherical for transformed point
        r_yin = np.sqrt(x_yin**2 + y_yin**2 + z_yin**2)
        theta_yin = np.arccos(z_yin / r_yin) if r_yin > 0 else 0
        phi_yin = np.arctan2(y_yin, x_yin)
        
        # Check Yin bounds on transformed point
        if not (np.pi/4 <= theta_yin <= 3*np.pi/4):
            return None
        if not (-3*np.pi/4 <= phi_yin <= 3*np.pi/4):
            return None
            
        idx_theta = int((theta_yin - np.pi/4) / grid_params['delta_theta_yin'])
        idx_phi = int((phi_yin - (-3*np.pi/4)) / grid_params['delta_phi_yin'])
        
        # Radial index for Yang is same scale
        if r_yin < grid_params['radial_scales_yang'][0]:
            idx_r = 0
        else:
            idx_r = int(1 + np.log(r_yin / grid_params['radial_scales_yang'][0]) / np.log(grid_params['radial_scales_yang'][1] / grid_params['radial_scales_yang'][0]))
            if idx_r >= len(grid_params['radial_scales_yang']):
                idx_r = len(grid_params['radial_scales_yang']) - 1
                
        return (idx_r, idx_theta, idx_phi)
        
    return None

def interpolate_grid_value(grid_tensor, idx_r, idx_theta, idx_phi, x, y, z, grid_params, grid_type='Yin'):
    """
    Trilinear interpolation from the grid tensor.
    """
    # Get grid cell bounds for interpolation weights
    # This requires retrieving the specific r, theta, phi bounds for the indices
    # For brevity, assuming helper functions get_bounds() exist
    
    # 1. Determine bounds
    if grid_type == 'Yin':
        r_min = grid_params['radial_scales_yin'][idx_r] if idx_r > 0 else grid_params['radial_scales_yin'][0] # Simplified
        r_max = grid_params['radial_scales_yin'][idx_r + 1] if idx_r + 1 < len(grid_params['radial_scales_yin']) else grid_params['R_max']
        # Theta/Phi bounds calculation similar to index derivation
        theta_min = np.pi/4 + idx_theta * grid_params['delta_theta_yin']
        theta_max = theta_min + grid_params['delta_theta_yin']
        phi_min = -3*np.pi/4 + idx_phi * grid_params['delta_phi_yin']
        phi_max = phi_min + grid_params['delta_phi_yin']
    else:
        # Transform point to Yin basis for bounds lookup
        x_yin, y_yin, z_yin = transform_yang_to_yin_coords(x, y, z)
        r_yin = np.sqrt(x_yin**2 + y_yin**2 + z_yin**2)
        theta_yin = np.arccos(z_yin / r_yin)
        phi_yin = np.arctan2(y_yin, x_yin)
        
        r_min = grid_params['radial_scales_yang'][idx_r]
        r_max = grid_params['radial_scales_yang'][idx_r + 1]
        theta_min = np.pi/4 + idx_theta * grid_params['delta_theta_yin']
        theta_max = theta_min + grid_params['delta_theta_yin']
        phi_min = -3*np.pi/4 + idx_phi * grid_params['delta_phi_yin']
        phi_max = phi_min + grid_params['delta_phi_yin']
        
    # 2. Compute interpolation weights (normalized 0-1)
    # Note: Exact weight calculation depends on non-linear radial spacing
    # For radial, use log-space or direct ratio if linear approximation is acceptable for [G]
    # Standard trilinear assumes linear spacing. For exponential radial, 
    # one might interpolate in log(r) space or use the specific cell boundaries.
    # Here we assume standard trilinear on the tensor indices for the [G] implementation detail.
    
    # ... (Standard trilinear interpolation logic using grid_tensor values at 8 corners) ...
    
    return interpolated_value
```

## Operating characteristics
*   **Time Complexity:** Faster convergence compared to MLP-based methods (NeRF, mip-NeRF 360) and higher performance than Cartesian grid methods (TensoRF, DVGO) [S1]. The factorization of feature tensors into vectors and matrices further enhances performance [S1].
*   **Memory Complexity:** Lower memory footprint than Cartesian grids for large-scale scenes because resolution adapts to distance (coarse for far, fine for near) [S1]. No additional memory is allocated for a coarse density grid; the dense geometry feature grid $G_\sigma$ is reused for coarse sampling [S1].
*   **Strengths:** Efficient for large-scale unbounded scenes; avoids polar artifacts; aligns with egocentric camera rays; fast training and rendering [S1].
*   **Limitations:** Bounded by $R_{\max}$ for the spherical grid part; requires egocentric omnidirectional input [S1].
*   **When to use:** Egocentric view synthesis, VR asset creation from casual 360 video, large-scale indoor/outdoor environments [S1].
*   **When not to use:** Non-egocentric views, scenes where Cartesian symmetry is preferred, or when unbounded depth exceeds the environment map's utility [S1].

## Background
The Balanced Spherical Grid was introduced in EgoNeRF by Yin et al. in 2023 [S1]. It was developed to address the limitations of Cartesian feature grids in representing large-scale real-world environments for VR assets, specifically the inefficiency of uniform resolution and non-uniform ray-grid hits in egocentric scenarios [S1]. The method builds on recent acceleration techniques for NeRF using feature grids [S1].

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
