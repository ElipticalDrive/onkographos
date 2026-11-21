# EgoNeRF


## Overview
EgoNeRF is an efficient adaptation of NeRF [1185-nerf-neural-radiance-fields.md] for large-scale scenes with casual input, utilizing a balanced spherical feature grid and maintaining uniform ray hit rates for individual cells for scenes captured with a short video of omnidirectional cameras [S1]. It addresses the inefficiency and artifact issues of Cartesian grids in wide-field-of-view settings by employing a spherical coordinate system that matches egocentric capture geometry [S1]. Unlike [Mip-NeRF](3381-mip-nerf.md) which relies on a proposal MLP for resampling, EgoNeRF shares a single density grid for coarse and fine sampling, enabling faster convergence and superior preservation of fine structures [S1].

## How it works
EgoNeRF represents the scene using a continuous volumetric field parameterized by a balanced spherical feature grid and a lightweight MLP, optimized via differentiable volume rendering [G].

### Inputs and Outputs
- **Inputs**: Multi-view equirectangular images from an omnidirectional camera; camera poses estimated via Structure-from-Motion (SfM) [S1].
- **Outputs**: A neural volume representation allowing synthesis of novel views; rendered images with high fidelity to ground truth [S1].

### Data Structures
The core data structure is the **Balanced Spherical Feature Grid** [S1]. Unlike Cartesian grids (e.g., [DVGO](1185-nerf-neural-radiance-fields.md) or [TensoRF](1185-nerf-neural-radiance-fields.md)), this grid is defined in spherical coordinates $(r, \theta, \phi)$ to match the geometry of egocentric capture [S1].

1.  **Radial Partition**: Uniformly spaced concentric shells [S1].
2.  **Angular Partition**: Uses a **Yin-Yang grid** combined with a uniform radial partition to minimize valence variations at the poles [S1].
3.  **Feature Storage**: Each grid cell stores feature vectors, which are processed via vector-matrix factorization [S1].
4.  **Density Grid ($G_\sigma$)**: A shared grid used to query volume density for both coarse and fine sampling stages [S1].

### Parameters
- **Hardware**: Experiments conducted on a single RTX-3090 GPU [S1].
- **Ray Batch Size**: Same as baseline methods (DVGO, TensoRF) for fair comparison [S1].
- **Grid Resolution**: Same number of feature grids as DVGO and TensoRF [S1].
- **Training Steps**: High performance achieved at 5,000 steps [S1].

### Algorithm Steps
1.  **Pose Estimation**: Obtain camera poses from input video frames using the OpenMVG SfM library [S1].
2.  **Grid Initialization**: Initialize the balanced spherical feature grid with uniform radial partitions and Yin-Yang angular partitions [S1].
3.  **Ray Sampling**:
    - Cast rays from the camera through each pixel [G].
    - Sample points along the ray [G].
    - Query the shared density grid $G_\sigma$ for volume density at these points [S1].
4.  **Resampling**: Employ resampling techniques to refine sample distribution based on the shared density grid $G_\sigma$, rather than relying on a separate proposal MLP [S1].
5.  **Feature Aggregation**:
    - Retrieve features from the spherical grid for each sample point [G].
    - Apply vector-matrix factorization to combine features [S1].
6.  **Rendering**:
    - Compute color and density for each sample [G].
    - Perform volume rendering (compositing) to produce the final pixel color [G].
7.  **Optimization**:
    - Minimize the difference between rendered images and ground truth images [G].
    - Update grid features and MLP parameters via gradient descent [G].
8.  **Environment Map**: Include an environment map component to handle infinitely far regions, particularly in outdoor scenes [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def volume_render_ray(ray_origin, ray_direction, t_min, t_max, 
                      grid_sigma, grid_spherical, env_map, num_samples=128):
    """
    Renders a single ray using the shared density grid and spherical feature grid.
    Implements standard volume rendering compositing:
    C(r) = sum_{i=1}^N T_i * (1 - exp(-sigma_i * delta_i)) * c_i
    T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    """
    # 1. Coarse sampling along the ray
    coarse_ts = [t_min + i * (t_max - t_min) / (num_samples - 1) for i in range(num_samples)]
    coarse_points = [ray_origin + ray_direction * t for t in coarse_ts]
    
    # 2. Query density from shared grid for coarse samples
    coarse_sigmas = [grid_sigma.query(p) for p in coarse_points]
    
    # 3. Resampling based on density (importance sampling)
    # Generate fine samples weighted by coarse density
    fine_ts = []
    fine_weights = [1.0 - math.exp(-s * (t_max - t_min) / (num_samples - 1)) for s in coarse_sigmas]
    fine_weights = [w / sum(fine_weights) for w in fine_weights]
    # Simple stratified resampling for implementation clarity
    for i in range(num_samples):
        t = t_min + (i + 0.5) * (t_max - t_min) / num_samples
        fine_ts.append(t)
    fine_points = [ray_origin + ray_direction * t for t in fine_ts]
    
    # 4. Fine rendering loop
    T = 1.0
    color_accum = [0.0, 0.0, 0.0]
    delta_t = (t_max - t_min) / (num_samples - 1)
    
    for i, point in enumerate(fine_points):
        sigma = grid_sigma.query(point)
        alpha = 1.0 - math.exp(-sigma * delta_t)
        weight = alpha * T
        
        # Query spherical grid and apply vector-matrix factorization [G]
        r = math.sqrt(point[0]**2 + point[1]**2 + point[2]**2)
        theta = math.atan2(math.sqrt(point[0]**2 + point[1]**2), point[2])
        phi = math.atan2(point[1], point[0])
        features = grid_spherical.sample(r, theta, phi)
        color = [sum(f * w for f, w in zip(features, [1.0, 0.5, 0.25])) for _ in range(3)] # Placeholder factorization [G]
        
        color_accum = [color_accum[j] + weight * color[j] for j in range(3)]
        T *= (1.0 - alpha)
        
        if T < 1e-4:
            break
            
    # 5. Background accumulation for unbounded scenes
    background = env_map.query(ray_direction)
    final_color = [color_accum[j] + T * background[j] for j in range(3)]
    return final_color, T

def train_step(images, poses, grid_sigma, grid_spherical, env_map, lr=0.01):
    """
    One step of optimization. Computes loss and updates parameters.
    """
    total_loss = 0.0
    for image, pose in zip(images, poses):
        for u, v in image.pixels:
            ray_origin = pose.camera_center
            ray_direction = pose.get_ray_direction(u, v)
            rendered_color, _ = volume_render_ray(ray_origin, ray_direction, 
                                                  pose.t_min, pose.t_max,
                                                  grid_sigma, grid_spherical, env_map)
            target_color = image.get_pixel(u, v)
            # MSE loss [G]
            loss = sum((r - t)**2 for r, t in zip(rendered_color, target_color)) / 3.0
            total_loss += loss
            
    # Gradient descent update [G]
    gradients = compute_gradients(total_loss, [grid_sigma, grid_spherical, env_map])
    update_parameters(gradients, lr)
    return total_loss
```

### Edge Cases and Pitfalls
- **Camera Pose Errors**: Erroneous pose estimates cause rays to hit neighboring cells in the feature grid. This results in **spotty artifacts** in EgoNeRF, whereas MLP-based methods (like standard NeRF) tend to produce **blurry rendering** in the same scenario [S1].
- **Pole Distortion**: Equirectangular images have distortion near poles. The authors additionally measure weighted-to-spherically uniform PSNR and SSIM (PSNRWS, SSIMWS) to account for this distortion during evaluation [S1].
- **Missing Components**:
    - Removing the balanced spherical grid components (uniform radial partition, Yin-Yang grid) degrades performance [S1].
    - Removing both radial and angular balanced grids (using uniform spherical grids) causes the biggest drop in performance, except for PSNR in indoor scenes [S1].
    - Not employing resampling techniques and using a double number of ray samples reduces performance [S1].
    - Removing the environment map in outdoor scenes shows blurry artifacts in infinitely far regions and reduces performance [S1].

## Operating characteristics
- **Speed**: EgoNeRF achieves high performance very quickly. It reaches competitive results at 5,000 training steps, taking only **10 minutes** of wall-clock time [S1]. In contrast, [Mip-NeRF](3381-mip-nerf.md) requires approximately **8 hours** to outperform EgoNeRF's 5k-step results [S1].
- **Accuracy**: On the OmniBlender dataset, EgoNeRF (full) achieves:
    - Indoor: PSNR 33.11, LPIPS 0.142, SSIM 0.902 [S1].
    - Outdoor: PSNR 30.56, LPIPS 0.087, SSIM 0.904 [S1].
- **Strengths**:
    - Efficient for large-scale, unbounded scenes captured with omnidirectional cameras [S1].
    - Preserves fine structures (e.g., thin wires, street lamps) better than [Mip-NeRF](3381-mip-nerf.md), which often misses them due to reliance on the proposal MLP's initial guess [S1].
    - Avoids the "wasted cells" and low-resolution center issues of Cartesian grids (TensoRF, DVGO) in large scenes [S1].
- **Limitations**:
    - Sensitive to camera pose errors, leading to spotty artifacts [S1].
    - Performance degrades if the balanced spherical grid structure is simplified to uniform spherical grids [S1].
- **When to Use**: Ideal for egocentric, large-scale scene reconstruction from casual omnidirectional video where fast training and high fidelity to fine details are required [S1].
- **When Not to Use**: If camera poses are highly inaccurate and spotty artifacts are unacceptable, or if the scene is small and bounded where Cartesian grids might suffice [S1].

## Background
EgoNeRF is a variant of NeRF [1185-nerf-neural-radiance-fields.md] introduced to address the limitations of standard NeRF and its variants in large-scale, egocentric settings [S1]. While [Mip-NeRF](3381-mip-nerf.md) adapts to unbounded scenes by warping space, EgoNeRF focuses on the geometric suitability of the feature grid for spherical capture [S1]. It was developed to enable efficient synthesis of novel views from short videos captured with omnidirectional cameras, leveraging a balanced spherical feature grid to maintain uniform ray hit rates [S1].

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
