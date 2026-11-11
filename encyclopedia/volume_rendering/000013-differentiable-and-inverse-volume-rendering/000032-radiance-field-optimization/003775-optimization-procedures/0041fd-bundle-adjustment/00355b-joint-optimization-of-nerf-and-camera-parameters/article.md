# Joint Optimization of NeRF and Camera Parameters

Also called: Neural Radiance Fields Without Known Camera Parameters

## Overview
This method jointly optimizes a Neural Radiance Field (NeRF) model and camera parameters (poses and focal length) for forward-facing images, eliminating the need for pre-computed camera poses via Structure-from-Motion (SfM) systems like COLMAP [S1, unofficial]. It simplifies classical two-stage Novel View Synthesis (NVS) pipelines by treating camera extrinsics and intrinsics as learnable parameters alongside the neural network weights [S1, unofficial]. Unlike [Bundle Adjustment](8540-bundle-adjustment.md), which typically refines known poses, this approach initializes poses randomly or with coarse estimates and converges them simultaneously with the scene representation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images $\{I_i\}_{i=1}^N$ with pixel coordinates $\{p_{i,m}\}$, where $M$ is the number of pixels per image.
*   **Outputs:** Optimized NeRF network parameters $\hat{\Theta}$, camera poses $\{\hat{\phi}_i, \hat{t}_i\}_{i=1}^N$ (rotation and translation), and focal length $\hat{f}$.

**Data Structures:**
*   **Camera Poses:** $\hat{\phi}_i \in \mathbb{R}^3$ (e.g., axis-angle or quaternion representation) and $\hat{t}_i \in \mathbb{R}^3$ for each image $i$.
*   **Focal Length:** $\hat{f} \in \mathbb{R}$, a single scalar value.
*   **NeRF Network:** A multi-layer perceptron $F_\Theta$ taking position and view direction as input, outputting color and density.

**Parameters:**
*   **NeRF Architecture:** A smaller variant of the original NeRF [S1, unofficial]. Hidden layer dimensions are halved: the first 9 layers have 128 units (original 256), and the final layer has 64 units (original 128). Positional encoding frequencies and skip connections match the original NeRF implementation [S1, unofficial].
*   **Focal Length Initialization:** Initialized to the input image width $W$ [S1, unofficial].
*   **Scale Factor:** A parameter $s$ (or $\tilde{s}$) used to parameterize focal length, initialized to 1.0 [S1, unofficial].

**Step-by-Step Method:**

1.  **Ray Construction:** For each image $i$ and pixel $m$, construct the ray direction $\hat{d}_{i,m}$ using the current estimated pose $\hat{\phi}_i$, translation $\hat{t}_i$, focal length $\hat{f}$, and pixel coordinates $p_{i,m}$ [S1, unofficial].
2.  **Point Sampling:** Sample points $x_h$ along the ray for heights $h$ ranging from the near plane $h_n$ to the far plane $h_f$ [S1, unofficial].
3.  **NeRF Query:** Query the NeRF network $F_\Theta$ with position $x_h$ and direction $\hat{d}_{i,m}$ to obtain color $c_h$ and density $\sigma_h$ [S1, unofficial].
4.  **Volume Rendering:** Render the pixel value $\hat{I}_{i,m}$ using the collected colors and densities via standard volume rendering integration [S1, unofficial].
5.  **Loss Computation:** Compute the loss $\mathcal{L}$ between the rendered image $\hat{I}_i$ and the ground truth image $I_i$ [S1, unofficial].
6.  **Optimization:** Perform a backward pass and update parameters $[\hat{\phi}_i]$, $[\hat{t}_i]$, $\hat{f}$, and $\hat{\Theta}$ using an optimizer (e.g., Adam) [S1, unofficial].

**Focal Length Parameterization:**
Directly optimizing $f$ in pixel units is numerically difficult due to large magnitudes. Two parameterizations are used:
1.  **1st-order:** $f = s W$, where $s$ is initialized to 1.0 [S1, unofficial].
2.  **2nd-order (Preferred):** $f = \tilde{s}^2 W$, where $\tilde{s}$ is initialized to 1.0. This yields slightly better results (Mean PSNR 22.48 vs 22.26 on LLFF-NeRF) [S1, unofficial].
3.  **Single vs. Dual:** Optimizing a single focal length $f$ yields no performance difference compared to optimizing separate horizontal $f_x$ and vertical $f_y$ focal lengths; $f$ is chosen for simplicity [S1, unofficial].

**Evaluation Alignment:**
To eliminate camera mis-alignment effects during evaluation, optimized parameters are aligned with COLMAP estimates using a Sim(3) transformation via the Umeyama algorithm, followed by gradient-driven camera pose alignment minimizing photometric error on synthesized images while keeping the NeRF model fixed [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

# Assume NeRF_Module is defined elsewhere as a standard MLP
# class NeRF_Module(nn.Module): ...

def construct_ray_direction(poses, translations, focal_length, pixel_coords, image_width):
    """
    Constructs ray directions from camera parameters and pixel coordinates.
    poses: (N, 3) or (N, 3, 3) rotation parameters
    translations: (N, 3)
    focal_length: scalar
    pixel_coords: (M, 2) pixel indices (u, v)
    image_width: int W
    Returns: (N, M, 3) ray directions in world space
    """
    # Implementation depends on specific camera model (e.g., pinhole)
    # Standard pinhole: d = R^T * ( (u - W/2)/f, (v - H/2)/f, 1 )
    # This is a placeholder for the specific geometric derivation in Eq. 4 of [S1, unofficial]
    pass

def sample_points_along_ray(ray_direction, translation, near_plane, far_plane, num_samples):
    """
    Samples points along a ray.
    ray_direction: (3,)
    translation: (3,)
    near_plane: float
    far_plane: float
    num_samples: int
    Returns: (num_samples, 3) points in world space
    """
    # t = linspace(near, far, num_samples)
    # points = translation + t[:, None] * ray_direction
    pass

def render_volume_rendering(colors, densities, ray_direction):
    """
    Standard volume rendering integration.
    colors: (num_samples, 3)
    densities: (num_samples,)
    ray_direction: (3,) - used for view-dependent color if applicable
    Returns: (3,) rendered color
    """
    # alpha = 1 - exp(-sigma * delta_t)
    # weight = alpha * prod(1 - alpha_prev)
    # color = sum(weight * c)
    pass

def joint_optimization(
    images, 
    pixel_coords, 
    image_width, 
    near_plane, 
    far_plane, 
    num_samples,
    num_iterations
):
    """
    Jointly optimizes NeRF and camera parameters.
    """
    N = images.shape[0]
    M = images.shape[1] * images.shape[2] # Flattened pixels
    
    # Initialize parameters
    # Poses: axis-angle or quaternion representation
    poses = nn.Parameter(torch.randn(N, 3)) 
    translations = nn.Parameter(torch.randn(N, 3))
    
    # Focal length parameterization (2nd-order)
    s_tilde = nn.Parameter(torch.tensor(1.0))
    
    # NeRF Network
    nerf = NeRF_Module()
    
    optimizer = torch.optim.Adam([
        {'params': nerf.parameters()},
        {'params': poses},
        {'params': translations},
        {'params': [s_tilde]}
    ], lr=0.001)
    
    for step in range(num_iterations):
        optimizer.zero_grad()
        
        # Calculate actual focal length
        focal_length = s_tilde ** 2 * image_width
        
        total_loss = 0.0
        
        for i in range(N):
            # 1. Construct rays
            # Note: pixel_coords should be indexed per image
            ray_dirs = construct_ray_direction(
                poses[i:i+1], 
                translations[i:i+1], 
                focal_length, 
                pixel_coords[i], 
                image_width
            ) # Shape: (M, 3)
            
            rendered_image = torch.zeros_like(images[i])
            
            for m in range(M):
                # 2. Sample points
                points = sample_points_along_ray(
                    ray_dirs[m], 
                    translations[i], 
                    near_plane, 
                    far_plane, 
                    num_samples
                )
                
                # 3. Query NeRF
                # Flatten points for batched query if necessary
                colors, densities = nerf(points, ray_dirs[m])
                
                # 4. Render
                rendered_pixel = render_volume_rendering(colors, densities, ray_dirs[m])
                rendered_image[m] = rendered_pixel
            
            # 5. Compute Loss
            loss = nn.MSELoss()(rendered_image, images[i])
            total_loss += loss
        
        # 6. Backward pass
        total_loss.backward()
        optimizer.step()
        
        if step % 100 == 0:
            print(f"Step {step}, Loss: {total_loss.item()}")

    return nerf, poses, translations, s_tilde
```

## Operating characteristics
*   **Time Complexity:** Training is slower than fixed-camera NeRF due to the additional gradients flowing through camera parameters. A BLEFF scene with 27 training images takes approximately 5.5 hours on a single GPU [S1, unofficial].
*   **Memory Complexity:** Increases slightly due to the storage of learnable pose parameters ($6N$ floats) and the scale factor, but the dominant memory cost remains the NeRF network and intermediate rendering buffers.
*   **Strengths:** Eliminates the dependency on SfM pipelines (e.g., COLMAP), which can fail on textureless or repetitive scenes. Simplifies the NVS pipeline to a single end-to-end optimization step [S1, unofficial].
*   **Limitations:** Designed for static scenes and forward-facing images; cannot handle dynamic objects [S1, unofficial]. Convergence may be sensitive to initialization. The method does not inherently handle radiometric calibration or viewpoint selection, which are addressed by sibling methods [1121-best-viewpoint-selection-via-opacity-entropy.md] and [1820-radiometric-self-calibration.md].
*   **When to Use:** When camera poses are unknown, unreliable, or when a simplified pipeline is preferred over a two-stage SfM+NeRF approach.
*   **When Not to Use:** For dynamic scenes, or when high-precision metric reconstruction is required without careful evaluation alignment [S1, unofficial].

## Background
The method was introduced in "NeRF--: Neural Radiance Fields Without Known Camera Parameters" [S1, unofficial]. It addresses the limitations of classical two-stage NVS systems that rely on pre-computed camera poses. The authors demonstrated that joint optimization is feasible and effective for forward-facing datasets like LLFF and BLEFF [S1, unofficial]. The approach builds on the differentiable rendering framework of original NeRF [4] but extends the optimization domain to include camera extrinsics and intrinsics [S1, unofficial].

## Sources
[S1] NeRF--: Neural Radiance Fields Without Known Camera Parameters, https://doi.org/10.48550/arxiv.2102.07064, fetched 2026-10-26 (unofficial)
