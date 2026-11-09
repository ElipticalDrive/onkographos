# Deep Reflectance Volumes

Also called: DRV, DRV rendering, Deep Reflectance Volume Rendering, discretized differentiable volume rendering

## Overview
Deep Reflectance Volumes (DRV) is a differentiable volume rendering method that reconstructs scene appearance from unstructured images captured under collocated point lighting [S1]. It represents a scene using three voxel grids—opacity, surface normal, and reflectance parameters—and optimizes these volumes by minimizing the error between rendered and captured images [S1]. Unlike previous view synthesis methods that accumulate opacity only along view directions, DRV learns global scene opacity accumulated from both view and light directions to compute accurate hard shadows, and unlike volume-based works that learn a single color per voxel, it reconstructs pervoxel reflectance to handle complex materials with high glossiness [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of unstructured images captured under collocated point lighting.
*   **Outputs:** Optimized voxel grids for opacity ($\alpha$), normal ($\mathbf{n}$), and reflectance parameters ($R$) in a transformed space, which are warped to world space for rendering.

**Data Structures:**
The method uses a decoder-like neural network that outputs an 8-channel tensor of size $128 \times 128 \times 128$ in a warped space [S1]. The channel layout is:
*   Channel 1: Opacity $\alpha_w$
*   Channels 2-4: Normal $\mathbf{n}_w$ (3 components)
*   Channels 5-8: Reflectance $R_w$ (3 for diffuse albedo, 1 for specular roughness) [S1].

The warping function requires additional learned parameters:
*   A global affine matrix $W_g$.
*   16 basis affine matrices $\{W_j\}_{j=1}^{16}$.
*   A $32 \times 32 \times 32$ volume $B$ with 16 channels, containing normalized spatial weights for the basis matrices [S1].

**Warping Function:**
To increase effective resolution, world-space positions $\mathbf{x}$ are mapped to a transformed space using [S1]:
$$ W(\mathbf{x}) = \left[ \sum_{j=1}^{16} B_j(\mathbf{x}) W_j \right]^{-1} W_g \mathbf{x} $$
where $B_j(\mathbf{x})$ is the $j$-th channel of volume $B$ sampled at $\mathbf{x}$ [S1].

**Rendering Algorithm:**
The rendering module is based on classical volume ray marching with opacity accumulation [S1]. The scene is represented by opacity $\alpha$, normal $\mathbf{n}$, and reflectance $R$ volumes [S1].

1.  **Ray Sampling:** For a camera position $\mathbf{c}$ and view direction $-\boldsymbol{\omega}_o$, points are sampled along the ray with a fixed step size $\Delta x$:
    $$ \mathbf{x}_s = \mathbf{x}_{s-1} - \boldsymbol{\omega}_o \Delta x = \mathbf{c} - s \boldsymbol{\omega}_o \Delta x $$
    The opacity $\alpha$ is dependent on $\Delta x$, representing one minus the transmission $\tau$ over the segment [S1].

2.  **Radiance and Opacity Accumulation:** Radiance $L_s$ and camera opacity $\alpha_{c \to s}$ are recursively accumulated until the sample $\mathbf{x}_s$ exits the volume [S1]:
    $$ L_s = L_{s-1} + [1 - \alpha_{c \to (s-1)}][1 - \alpha_{l \to (t-1)}]\alpha(\mathbf{x}_s)L(\mathbf{x}_s) $$
    $$ \alpha_{c \to s} = \alpha_{c \to (s-1)} + [1 - \alpha_{c \to (s-1)}] \alpha(\mathbf{x}_s) $$
    $$ \alpha_{l \to t} = \alpha_{l \to (t-1)} + [1 - \alpha_{l \to (t-1)}] \alpha(\mathbf{x}_t) $$
    Note: During training with collocated camera and light, $\alpha_{l \to t}$ is equivalent to $\alpha_{c \to s}$, allowing single-pass accumulation. For non-collocated lighting, two-pass accumulation is required [S1].

3.  **Point-wise Shading:** The radiance contribution $L(\mathbf{x}_s)$ at sample $\mathbf{x}_s$ is computed using the local normal $\mathbf{n}(\mathbf{x}_s)$ and reflectance parameters $R(\mathbf{x}_s)$ [S1]:
    $$ L(\mathbf{x}_s) = f_r(\boldsymbol{\omega}_o, \boldsymbol{\omega}_i, \mathbf{n}(\mathbf{x}_s), R(\mathbf{x}_s)) L_l(\mathbf{x}_s, \boldsymbol{\omega}_i) $$
    where $\boldsymbol{\omega}_i$ is the direction from $\mathbf{x}_s$ to the light position $\mathbf{l}$, and $L_l$ is the light radiance [S1]. The method uses a simplified Disney BRDF model parameterized by diffuse albedo and specular roughness [S1].

**Loss Function:**
The total loss function combines rendering error, opacity regularization, and spatial sparsity regularization [S1]:
$$ \mathcal{L} = \sum_k \| L_k - \tilde{L}_k \|^2 + \beta_1 \sum_k [\log(\alpha_{c_k \to s'}) + \log(1 - \alpha_{c_k \to s'})] + \beta_2 \sum \| \nabla_{\mathbf{x}} \log \alpha(\mathbf{x}) \| $$
where the first term is the rendering loss, the second regularizes accumulated opacity to be 0 or 1, and the third regularizes spatial sparsity of opacity [S1].

**Parameters:**
*   **Volume Bounds:** The volume is a cube located at $[-1, 1]^3$ [S1].
*   **Ray Marching Step Size:** $\Delta x = 1/64$ during training [S1].
*   **Training Batch:** 128×128 pixels sampled from 8 captured images [S1].
*   **Regularization Weights:** Initially $\beta_1 = 0.01, \beta_2 = 0.01$. After 300,000 iterations, weights are increased to $\beta_1 = 1.0, \beta_2 = 0.1$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deep_reflectance_volumes_rendering(camera_pos, view_dir, light_pos, 
                                       opacity_vol, normal_vol, reflectance_vol,
                                       global_affine, basis_affines, weight_vol,
                                       step_size=1/64, max_steps=128):
    """
    Renders a single ray using Deep Reflectance Volumes.
    
    Args:
        camera_pos: 3D vector, camera position c.
        view_dir: 3D vector, view direction -omega_o.
        light_pos: 3D vector, light position l.
        opacity_vol: 3D tensor, opacity alpha in warped space.
        normal_vol: 3D tensor, normal n in warped space.
        reflectance_vol: 3D tensor, reflectance R (albedo, roughness) in warped space.
        global_affine: 4x4 matrix, W_g.
        basis_affines: list of 16 4x4 matrices, {W_j}.
        weight_vol: 3D tensor (32x32x32x16), B.
        step_size: float, Delta x.
        max_steps: int, maximum number of samples.
        
    Returns:
        radiance: float, final pixel color.
    """
    
    # Initialize accumulators
    accumulated_radiance = 0.0
    accumulated_cam_opacity = 0.0
    accumulated_light_opacity = 0.0
    
    # Current position along ray
    current_pos = camera_pos
    
    for s in range(max_steps):
        # Sample point in world space
        # x_s = c - s * omega_o * Delta_x
        # Note: view_dir is -omega_o, so we add view_dir * step_size
        sample_pos_world = current_pos + view_dir * step_size
        
        # Check if sample is within volume bounds [-1, 1]^3
        if not is_inside_volume(sample_pos_world):
            break
            
        # Warp world position to transformed space
        warped_pos = warp_position(sample_pos_world, global_affine, basis_affines, weight_vol)
        
        # Interpolate volumes at warped position
        alpha_w = interpolate_3d(opacity_vol, warped_pos)
        n_w = interpolate_3d(normal_vol, warped_pos)
        R_w = interpolate_3d(reflectance_vol, warped_pos)
        
        # Decode normal (ensure unit length)
        normal = normalize(n_w)
        
        # Compute reflectance (simplified Disney BRDF)
        # R_w contains [albedo_r, albedo_g, albedo_b, roughness]
        albedo = R_w[:3]
        roughness = R_w[3]
        
        # Direction to light
        to_light = light_pos - sample_pos_world
        dist_to_light = length(to_light)
        if dist_to_light < 1e-6:
            break
        omega_i = to_light / dist_to_light
        
        # Compute reflected radiance
        # L(x_s) = f_r(omega_o, omega_i, n(x_s), R(x_s)) * L_l(x_s, omega_i)
        # Assuming point light with radiance L_l
        reflected_radiance = compute_brdf(reflected_radiance, view_dir, omega_i, normal, albedo, roughness)
        light_radiance = get_light_radiance(light_pos, sample_pos_world) # Simplified
        L_sample = reflected_radiance * light_radiance
        
        # Update accumulators
        # L_s = L_{s-1} + [1 - alpha_{c->(s-1)}][1 - alpha_{l->(t-1)}] * alpha(x_s) * L(x_s)
        term = (1.0 - accumulated_cam_opacity) * (1.0 - accumulated_light_opacity) * alpha_w * L_sample
        accumulated_radiance += term
        
        # alpha_{c->s} = alpha_{c->(s-1)} + [1 - alpha_{c->(s-1)}] * alpha(x_s)
        accumulated_cam_opacity += (1.0 - accumulated_cam_opacity) * alpha_w
        
        # alpha_{l->t} = alpha_{l->(t-1)} + [1 - alpha_{l->(t-1)}] * alpha(x_s)
        # Note: During training with collocated light, this is same as cam opacity.
        # For general rendering, track separately.
        accumulated_light_opacity += (1.0 - accumulated_light_opacity) * alpha_w
        
        # Early exit if fully opaque
        if accumulated_cam_opacity >= 1.0:
            break
            
        current_pos = sample_pos_world
        
    return accumulated_radiance

def warp_position(x_world, W_g, basis_affines, B_vol):
    """
    Warps world position x to transformed space.
    W(x) = [sum_{j=1}^{16} B_j(x) W_j]^{-1} W_g x
    """
    # Interpolate weights B(x)
    B_x = interpolate_3d(B_vol, x_world) # Returns 16-channel vector
    
    # Compute weighted sum of basis affine matrices
    weighted_sum = zeros(4, 4)
    for j in range(16):
        weighted_sum += B_x[j] * basis_affines[j]
        
    # Inverse of weighted sum
    inv_weighted_sum = inverse(weighted_sum)
    
    # Apply global affine to x
    x_homogeneous = append_ones(x_world)
    transformed_x = multiply_matrix_vector(W_g, x_homogeneous)
    
    # Apply inverse weighted sum
    result_homogeneous = multiply_matrix_vector(inv_weighted_sum, transformed_x)
    
    # Convert back to 3D (divide by w)
    result_3d = result_homogeneous[:3] / result_homogeneous[3]
    
    return result_3d

def is_inside_volume(pos, bounds_min=-1.0, bounds_max=1.0):
    return all(bounds_min <= p <= bounds_max for p in pos)

def compute_brdf(omega_o, omega_i, normal, albedo, roughness):
    """
    Simplified Disney BRDF.
    Returns reflected radiance factor.
    """
    # Implementation details of simplified Disney BRDF are in supplementary materials [S1].
    # Placeholder for the actual BRDF evaluation.
    # Typically involves diffuse and specular components.
    return albedo # Simplified for example; actual implementation uses roughness.

def get_light_radiance(light_pos, sample_pos):
    """
    Returns radiance from point light at sample_pos.
    """
    # Assuming point light with intensity I
    # L_l = I / dist^2 (simplified)
    dist = length(light_pos - sample_pos)
    if dist < 1e-6:
        return 0.0
    return 1.0 / (dist * dist) # Simplified
```

## Subtopics

### Differentiable Volume Rendering
The core framework that allows backpropagation through the ray marching process. DRV builds on [Differentiable volume rendering](8825-differentiable-volume-rendering.md) by incorporating physically-based shading and dual opacity accumulation.

### Neural Volume Representation
DRV uses a neural decoder to generate voxel grids. This relates to [Neural Radiance Fields (NeRF)](52-nerf.md) but differs by using explicit voxel grids for opacity, normal, and reflectance rather than implicit MLPs for color and density.

### Inverse Rendering
DRV solves the inverse problem of reconstructing scene properties from images. It is a specific instance of [Inverse Rendering and Optimization](8999-inverse-rendering-and-optimization.md).

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of ray samples per pixel and the number of pixels. Training time is significant due to the per-scene optimization of the neural network and the differentiable ray marching.
*   **Memory Complexity:** The method stores four voxel grids (opacity, normal, albedo, roughness) of size $128 \times 128 \times 128$ in warped space, plus the warping parameters. This is memory-intensive compared to implicit representations.
*   **Performance:** The method is able to reconstruct real scenes with challenging non-Lambertian reflectance and complex geometry with occlusions and shadowing [S1]. It accurately generalizes to novel viewpoints and lighting, including non-collocated lighting, rendering photorealistic images that are significantly better than state-of-the-art mesh-based methods [S1].
*   **Strengths:** Handles complex materials with high glossiness, computes accurate hard shadows, supports scene material editing, and generalizes to novel lighting [S1].
*   **Limitations:** Fully computing the general radiative transfer equation requires multiple-scattering computation using Monte Carlo methods, which is computationally expensive and unsuitable for deep learning techniques; thus, DRV simplifies to single scattering [S1]. Training requires collocated camera and lighting for single-pass accumulation, though inference supports non-collocated lighting [S1].

## Background
Deep Reflectance Volumes was introduced by Kriegman et al. from the University of California, San Diego and Adobe Research [S1]. The method was presented in the paper "Deep Reflectance Volumes: Relightable Reconstructions from Multi-view Photometric Images" published in 2020 [S1]. It addresses the limitations of previous view synthesis and volume-based works by learning global scene opacity from both view and light directions and reconstructing pervoxel reflectance [S1].

## Sources
[S1] Deep Reflectance Volumes: Relightable Reconstructions from Multi-view Photometric Images, https://doi.org/10.1007/978-3-030-58580-8_18, fetched 2026-10-26
