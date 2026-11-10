# Progressive Sparse Voxels Optimization

Also called: Progressive Sparse Voxels

## Overview
Progressive Sparse Voxels Optimization is a coarse-to-fine training procedure for optimizing 3D radiance fields represented as sparse voxel grids [S1]. It solves the problem of initializing and maintaining an efficient voxel structure that adapts to scene complexity during training, preventing overfitting in unobserved regions while refining detail where needed [S1]. It sets itself apart from static grid methods by dynamically pruning low-visibility voxels and subdividing high-error voxels based on gradient signals [S1], building on the general framework of [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md).

## How it works
**Inputs:** A set of training images with known camera parameters (position, orientation, intrinsics including horizontal FOV $\theta_{\text{fov-x}}$ and image width $W$) [S1].
**Outputs:** An optimized sparse voxel grid containing Spherical Harmonic (SH) coefficients for color and density values at grid points [S1].

### Data Structures
The scene is represented as an Octree [S1].
*   **Voxel:** Contains SH coefficients (view-dependent color) and references to grid points [S1].
*   **Grid Point:** Stores raw density values [S1]. Grid points are shared among adjacent voxels (8 corners per voxel) [S1].
*   **Layout:** Only leaf nodes are stored in the final Octree structure [S1]. Source voxels are removed upon subdivision [S1].

### Parameters
*   $h_{lv}$: Number of levels for the dense grid initialization in bounded scenes or the main region of unbounded scenes [S1].
*   $h_{out}$: Number of background shell levels for unbounded scenes [S1].
*   $h_{ratio}$: Target ratio of background voxels to main region voxels after initialization [S1].
*   $h_{geo}$: Negative initial raw density value to ensure near-zero activated density ($\exp(h_{geo}) \approx 0$) [S1].
*   $h_{prune}$: Threshold for maximum blending weight to prune voxels [S1].
*   $h_{every}$: Number of training iterations between adaptive pruning and subdivision operations [S1].
*   $h_{rate}$: Threshold for sampling rate; if $v_{rate} < 2h_{rate}$, subdivision priority is set to zero [S1].
*   $h_{percent}$: Percentage of voxels with highest priority selected for subdivision [S1].
*   $\lambda_{\text{ssim}}, \lambda_{\text{T}}, \lambda_{\text{dist}}, \lambda_{\text{R}}, \lambda_{\text{tv}}$: Weights for SSIM, Total Transmittance, Distortion, Regularization, and Total Variation losses [S1].

### Step 1: Scene Initialization
Initialization depends on whether the scene is bounded or unbounded [S1]. All voxels start with raw density $h_{geo}$ and SH coefficients set to zero, except the zero-degree SH component which is set to yield gray color (intensity 0.5) [S1].

**1.1 Bounded Scenes**
1.  Initialize a dense grid with $h_{lv}$ levels [S1].
2.  Remove voxels that are not observed by any training image [S1].
3.  The total number of voxels is $\le (2^{h_{lv}})^3$ [S1].

**1.2 Unbounded Scenes**
1.  Define the **Main Region**: A cuboid centered at the average camera position, with radius equal to the median distance from the center to the cameras [S1]. Initialize a dense grid with $h_{lv}$ levels within this cuboid [S1].
2.  Define the **Background Region**: Allocate $h_{out}$ levels of background shells enclosing the main region [S1]. The radius of the entire scene is $2^{h_{out}}$ times the radius of the main region [S1].
3.  Initialize background shells with the coarsest voxel size [S1]. Each shell level starts with $4^3 - 2^3 = 56$ voxels [S1].
4.  Iteratively subdivide background shell voxels with the highest sampling rate and remove unobserved voxels [S1].
5.  Repeat step 4 until the ratio of the number of background voxels to main region voxels equals $h_{ratio}$ [S1].
6.  The total number of voxels is $\le (1 + h_{ratio})(2^{h_{lv}})^3$ [S1].

### Step 2: Optimization Loop
The optimization proceeds in iterations. Every $h_{every}$ iterations, perform Adaptive Pruning and Subdivision [S1]. The total loss function is:
$$
\mathcal{L} = \mathcal{L}_{\text{mse}} + \lambda_{\text{ssim}} \mathcal{L}_{\text{ssim}} + \lambda_{\text{T}} \mathcal{L}_{\text{T}} + \lambda_{\text{dist}} \mathcal{L}_{\text{dist}} + \lambda_{\text{R}} \mathcal{L}_{\text{R}} + \lambda_{\text{tv}} \mathcal{L}_{\text{tv}}
$$
where $\mathcal{L}_{\text{mse}}$ and $\mathcal{L}_{\text{ssim}}$ are photometric losses, $\mathcal{L}_{\text{T}}$ encourages binary transmittance, $\mathcal{L}_{\text{dist}}$ is distortion loss, $\mathcal{L}_{\text{R}}$ is per-point RGB loss, and $\mathcal{L}_{\text{tv}}$ is total variation loss on the sparse density grid [S1].

### Step 3: Adaptive Pruning
1.  For each voxel, compute the maximum blending weight ($T_i \alpha_i$) across all training cameras [S1]. $T_i$ is the transmittance before the voxel, and $\alpha_i$ is the opacity of the voxel, derived from the volume rendering equation (Eq. 1 in [S1]) [S1].
2.  Remove voxels where the maximum blending weight is lower than $h_{prune}$ [S1].
3.  Update SH coefficients and grid points:
    *   SH coefficients are pruned along with the voxels [S1].
    *   Grid points are removed only if they do not belong to the corners of any remaining voxel [S1].

### Step 4: Subdivision
1.  **Compute Sampling Rate:** For each voxel, calculate the maximum sampling rate $v_{rate}$ to estimate the number of rays hitting the voxel along the image's horizontal axis [S1]:
    $$
    v_{rate} = \max_{i}^{N_{cam}} \frac{v_s}{v_{interval}^i}
    $$
    $$
    v_{interval}^{(i)} = (v_c - r_o^{(i)}) \cdot T \cdot \ell^{(i)} \cdot \tan\left(0.5 \cdot \theta_{\text{fov-x}}^{(i)}\right)
    $$
    Where:
    *   $v_s$: Voxel size (side length) [S1].
    *   $v_c$: Center of the voxel [S1].
    *   $r_o^{(i)}$: Origin of camera $i$ [S1].
    *   $T$: Rotation matrix of camera $i$ [S1].
    *   $\ell^{(i)}$: Camera lookat vector of camera $i$ [S1].
    *   $\theta_{\text{fov-x}}^{(i)}$: Horizontal field of view of camera $i$ [S1].
    *   $N_{cam}$: Number of training cameras [S1].
    *   Note: The source text mentions $W$ (image width) in the variable list but does not explicitly use it in the formula for $v_{interval}$ or $v_{rate}$ [S1].

2.  **Compute Priority:** Accumulate subdivision priority $v_{priority}$ for each voxel by summing over all training pixel rays $r \in R$ observed over the last $h_{every}$ iterations:
    $$
    v_{priority} = \sum_{r \in R} \left\| \alpha(r) \cdot \frac{\partial \mathcal{L}(r)}{\partial \alpha(r)} \right\|
    $$
    Where:
    *   $\alpha(r)$: Opacity contribution of the voxel to ray $r$ [S1].
    *   $\mathcal{L}(r)$: Training loss of ray $r$ [S1].
    *   The gradient is weighted by the alpha values [S1].

3.  **Filter Overfitting:** If a voxel's maximum sampling rate $v_{rate} < 2h_{rate}$, set its $v_{priority}$ to zero to prevent overfitting to few pixels [S1].

4.  **Select Voxels:** Select voxels with priority above the top $h_{percent}$ percentile [S1].
    *   The total number of voxels increases by $h_{percent} \cdot (8 - 1)$ percent [S1].

5.  **Subdivide and Update:**
    *   Subdivide selected voxels into 8 children [S1].
    *   Remove the source voxel from the Octree (only leaf nodes are kept) [S1].
    *   **SH Coefficients:** Duplicate the parent's SH coefficients to the 8 children [S1].
    *   **Grid Points:**
        *   Compute densities of new grid points using trilinear interpolation of the parent's corner grid points [S1].
        *   Merge duplicated grid points (shared between children or neighbors) and average their densities [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Dict, Tuple, Any

def compute_sampling_rate(voxel_center: Tuple[float, float, float], voxel_size: float, cameras: List[Dict]) -> float:
    """Compute max sampling rate v_rate for a voxel."""
    max_rate = 0.0
    for cam in cameras:
        # v_interval = (v_c - r_o) * T * l * tan(0.5 * fov_x)
        # (v_c - r_o) T l represents the voxel z-distance in camera space
        vec_to_center = (voxel_center[0] - cam['origin'][0],
                         voxel_center[1] - cam['origin'][1],
                         voxel_center[2] - cam['origin'][2])
        # T @ l gives the camera lookat vector in world space
        lookat_world = tuple(sum(cam['rot'][i][j] * cam['lookat'][j] for j in range(3)) for i in range(3))
        depth = sum(a * b for a, b in zip(vec_to_center, lookat_world))
        pixel_size = math.tan(0.5 * cam['fov_x'])
        v_interval = depth * pixel_size
        if v_interval > 0:
            rate = voxel_size / v_interval
            if rate > max_rate:
                max_rate = rate
    return max_rate

def is_observed(voxel: Dict, cameras: List[Dict]) -> bool:
    """Check if a voxel is observed by at least one training camera."""
    # Simplified visibility check: voxel center projects within image bounds for any camera
    for cam in cameras:
        vec = (voxel['center'][0] - cam['origin'][0],
               voxel['center'][1] - cam['origin'][1],
               voxel['center'][2] - cam['origin'][2])
        lookat_world = tuple(sum(cam['rot'][i][j] * cam['lookat'][j] for j in range(3)) for i in range(3))
        depth = sum(a * b for a, b in zip(vec, lookat_world))
        if depth > 0:
            return True
    return False

def compute_blending_weight(voxel: Dict, cam: Dict) -> float:
    """Compute T_i * alpha_i for a voxel and camera."""
    # T_i is transmittance before voxel, alpha_i is voxel opacity
    # Derived from volume rendering equation (Eq. 1 in [S1])
    # Placeholder for renderer-specific ray marching logic
    # Returns T_i * alpha_i
    return 0.0  # Implementation depends on the rasterizer

def initialize_scene(cameras: List[Dict], is_bounded: bool, h_lv: int, h_out: int, h_ratio: float, h_geo: float) -> List[Dict]:
    """Initialize the sparse voxel grid according to bounded/unbounded rules."""
    voxels = []
    if is_bounded:
        side = 2 ** h_lv
        for x in range(side):
            for y in range(side):
                for z in range(side):
                    center = ((x + 0.5) / side, (y + 0.5) / side, (z + 0.5) / side)
                    voxels.append({'center': center, 'size': 1.0 / side, 'sh_coeffs': [0.0] * 49, 'grid_corners': [], 'alive': True})
        voxels = [v for v in voxels if is_observed(v, cameras)]
    else:
        # Main region
        cam_positions = [cam['origin'] for cam in cameras]
        center = tuple(sum(p[i] for p in cam_positions) / len(cam_positions) for i in range(3))
        distances = [math.sqrt(sum((p[i] - center[i])**2 for i in range(3))) for p in cam_positions]
        distances.sort()
        radius = distances[len(distances) // 2]
        side = 2 ** h_lv
        for x in range(side):
            for y in range(side):
                for z in range(side):
                    offset = ((x + 0.5) / side) * 2 * radius - radius
                    c = (center[0] + offset, center[1] + offset, center[2] + offset)
                    voxels.append({'center': c, 'size': 2 * radius / side, 'sh_coeffs': [0.0] * 49, 'grid_corners': [], 'alive': True})
        # Background shells
        bg_radius = radius * (2 ** h_out)
        # Generate initial coarse shells (56 voxels per level)
        for level in range(h_out):
            inner_r = radius * (2 ** level)
            outer_r = radius * (2 ** (level + 1))
            shell_size = outer_r - inner_r
            # Create 56 voxels per shell level (simplified generation)
            for i in range(56):
                # Placeholder for shell voxel coordinate generation
                pass
        # Iteratively subdivide background based on sampling rate until ratio matches h_ratio
        # while ratio_bg_to_main(bg_voxels, main_voxels) < h_ratio:
        #     candidates = get_shell_voxels(bg_voxels)
        #     for v in candidates:
        #         v_rate = compute_sampling_rate(v['center'], v['size'], cameras)
        #         if is_observed(v, cameras):
        #             subdivide_voxel(v)
        #         else:
        #             v['alive'] = False
        pass
    # Initialize parameters
    for v in voxels:
        v['density_raw'] = h_geo
        v['sh_coeffs'][0] = 0.5  # Zero-degree SH for gray color
    return voxels

def optimize_step(voxels: List[Dict], cameras: List[Dict], iteration: int, h_every: int, h_prune: float, h_rate: float, h_percent: float) -> List[Dict]:
    """Perform pruning and subdivision every h_every iterations."""
    if iteration % h_every != 0:
        return voxels
        
    # 1. Adaptive Pruning
    pruned_indices = []
    for i, v in enumerate(voxels):
        if not v['alive']:
            continue
        max_blending_weight = 0.0
        for cam in cameras:
            bw = compute_blending_weight(v, cam)
            if bw > max_blending_weight:
                max_blending_weight = bw
        if max_blending_weight < h_prune:
            pruned_indices.append(i)
            
    for i in reversed(pruned_indices):
        voxels[i]['alive'] = False
        
    # 2. Subdivision
    # Collect rays from last h_every iterations (implementation depends on training loop)
    rays = []  # List of ray objects with alpha_contribution and gradient_loss_wrt_alpha methods
    
    priorities = []
    for v in voxels:
        if not v['alive']:
            continue
        v_rate = compute_sampling_rate(v['center'], v['size'], cameras)
        priority = 0.0
        for r in rays:
            if r.hits(v):
                alpha = r.alpha_contribution(v)
                grad = r.gradient_loss_wrt_alpha(v)
                priority += math.sqrt(sum(a**2 for a in grad))  # L2 norm
        if v_rate < 2 * h_rate:
            priority = 0.0
        priorities.append((v, priority))
        
    # Select top h_percent
    priorities.sort(key=lambda x: x[1], reverse=True)
    num_to_subdivide = max(1, int(len(priorities) * h_percent / 100))
    voxels_to_subdivide = [p[0] for p in priorities[:num_to_subdivide]]
    
    # Subdivide
    new_voxels = []
    for v in voxels_to_subdivide:
        # Generate 8 children
        half_size = v['size'] / 2.0
        for dx in [-1, 1]:
            for dy in [-1, 1]:
                for dz in [-1, 1]:
                    child_center = (v['center'][0] + dx * half_size,
                                    v['center'][1] + dy * half_size,
                                    v['center'][2] + dz * half_size)
                    child = {'center': child_center, 'size': half_size, 
                             'sh_coeffs': list(v['sh_coeffs']), 'grid_corners': [], 'alive': True}
                    # Trilinear interpolation for new grid points
                    child['density_raw'] = v['density_raw']  # Simplified; actual trilinear interpolation of corners required
                    new_voxels.append(child)
        v['alive'] = False  # Remove source voxel
        
    # Merge grid points and average densities
    all_alive = [v for v in voxels + new_voxels if v['alive']]
    grid_points = {}
    for v in all_alive:
        for corner in v['grid_corners']:
            key = tuple(corner)
            if key in grid_points:
                grid_points[key]['count'] += 1
                grid_points[key]['density_sum'] += corner['density']
            else:
                grid_points[key] = {'count': 1, 'density_sum': corner['density']}
    for key in grid_points:
        grid_points[key]['density'] = grid_points[key]['density_sum'] / grid_points[key]['count']
        
    return voxels + new_voxels
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels and training iterations [S1]. Subdivision increases the voxel count by $h_{percent} \cdot 7$ percent per step [S1]. Pruning reduces it [S1]. The cost of computing sampling rates and priorities scales with the number of voxels and rays [S1].
*   **Memory Complexity:** Scales with the number of active voxels [S1]. The initialization bounds the voxel count to $(1 + h_{ratio})(2^{h_{lv}})^3$ for unbounded scenes [S1].
*   **Strengths:** Adapts resolution to scene complexity; prevents overfitting in unobserved regions via pruning and sampling rate thresholds; handles unbounded scenes effectively with background shells [S1].
*   **Limitations:** Requires careful tuning of hyperparameters ($h_{lv}, h_{out}, h_{ratio}, h_{prune}, h_{rate}, h_{percent}$) [S1]. The subdivision process can be computationally intensive if many voxels are selected [S1].
*   **When to use:** When optimizing radiance fields from sparse or unstructured camera views where static grids are inefficient or when scene bounds are unknown (unbounded) [S1].

## Background
Progressive Sparse Voxels Optimization was introduced in the paper "Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering" [S1]. It is part of the SVRaster framework, designed for real-time rendering of high-fidelity radiance fields [S1]. The method addresses the limitations of static voxel grids by introducing a progressive optimization strategy that adapts the voxel structure during training [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
