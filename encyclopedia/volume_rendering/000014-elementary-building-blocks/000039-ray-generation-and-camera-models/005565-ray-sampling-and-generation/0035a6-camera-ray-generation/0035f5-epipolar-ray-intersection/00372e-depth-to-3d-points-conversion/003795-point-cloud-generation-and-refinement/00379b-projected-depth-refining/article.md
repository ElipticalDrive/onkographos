# Projected Depth Refining

Also called: Diffusion-based iterative depth refinement, Generative Depth Refinement Module, Scale-consistent depth integration

## Overview
Projected Depth Refining is a post-processing technique that enhances the geometric fidelity of depth maps projected from a 3D point cloud, addressing the loss of fine details and scale ambiguity inherent in initial projections. It sets itself apart from sibling methods like [Depth-Guided Optimization](14749-depth-guided-optimization.md) by explicitly integrating high-resolution RGB priors via diffusion models to recover high-frequency geometric details, rather than relying solely on geometric consistency or optimization constraints.

## How it works
The method operates in two primary contexts: a generative refinement for high-fidelity reconstruction [S1] and a computational refinement for efficient surface sampling [S2]. The core algorithm described in [S1] follows these steps:

1.  **Initial Projection**: The dense 3D point cloud (obtained from a prior step such as Neural Bundle Adjustment) is projected into a 2D depth map $D_0$ for a target view. This initial map is limited by the resolution of the source geometry estimator (e.g., DUSt3R) and lacks fine details [S1].
2.  **Latent Encoding**: The initial depth map $D_0$ and the corresponding RGB image $I$ are encoded into latent representations $z^{(D)}$ and $z^{(I)}$ using separate latent encoders [S1].
3.  **Diffusion-Based Refinement**: The latent depth representation $z^{(D)}$ is refined iteratively in the latent code space using a diffusion model [S1].
    *   **Forward Process**: Gaussian noise is added to the initial latent $z_0^{(D)}$ over $T$ steps. At step $t$, the noisy latent $z_t^{(D)}$ is computed as:
        $$ z_t^{(D)} = \sqrt{\bar{\alpha}_t} z_0^{(D)} + \sqrt{1 - \bar{\alpha}_t} \epsilon $$
        where $\epsilon \sim \mathcal{N}(0, I)$, and $\bar{\alpha}_t = \prod_{s=1}^{t} (1 - \beta_s)$, with $\{\beta_1, \dots, \beta_T\}$ being the noise variance schedule [S1].
    *   **Reverse Process**: The noise is progressively removed to recover the refined latent representation. This process is guided by the RGB latent $z^{(I)}$ to ensure texture-consistent geometric details [S1].
4.  **Scale-Consistent Integration**: To prevent scale drift during diffusion, a scale-consistent integration path constrains the depth scale at each iteration [S1].
5.  **Output**: The result is a scale-consistent, detail-rich depth map (and optionally normal maps) [S1].

In the context of [S2], the refinement focuses on computational efficiency for NeRF sampling:
1.  **Projection**: Project the saved 3D point cloud onto a novel view to generate a rough depth image [S2].
2.  **Point Removal**: Remove unnecessary 3D points from the point cloud to reduce noise and ambiguity in the depth estimation [S2].
3.  **Hole Filling**: Fill holes in the projected depth image using computationally efficient methods [S2].
4.  **Usage**: The refined depth image is used for surface-based sampling during novel view rendering, restricting 3D sampling to the vicinity of the estimated surface [S2].

### Pseudocode (Generative Variant [S1])

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def projected_depth_refining(point_cloud, rgb_image, camera_params, diffusion_model, latent_encoder_depth, latent_encoder_rgb, noise_schedule):
    """
    Refines a projected depth map using diffusion-based iterative refinement.
    
    Args:
        point_cloud: 3D point cloud (N, 3)
        rgb_image: Input RGB image (H, W, 3)
        camera_params: Camera intrinsic and extrinsic parameters
        diffusion_model: Pre-trained diffusion model for depth refinement
        latent_encoder_depth: Encoder for depth maps
        latent_encoder_rgb: Encoder for RGB images
        noise_schedule: List of beta values {beta_1, ..., beta_T}
        
    Returns:
        refined_depth_map: Refined depth map (H, W)
    """
    # Step 1: Initial Projection
    # Project point cloud to 2D depth map D0
    # Handle occlusions by keeping the closest point per pixel
    D0 = project_point_cloud_to_depth(point_cloud, camera_params)
    
    # Step 2: Latent Encoding
    # Encode depth and RGB into latent space
    z_D = latent_encoder_depth(D0)
    z_I = latent_encoder_rgb(rgb_image)
    
    # Step 3: Diffusion-Based Refinement
    T = len(noise_schedule)
    z_current = z_D
    
    # Forward process: Add noise
    # Note: In implementation, this is often done by sampling noise at step T
    # and then reversing. Here we simulate the iterative reverse process
    # guided by the RGB prior.
    
    # Initialize with noisy latent at step T
    epsilon = random_gaussian_noise_like(z_D)
    alpha_bar_T = product(1 - beta for beta in noise_schedule)
    z_T = sqrt(alpha_bar_T) * z_D + sqrt(1 - alpha_bar_T) * epsilon
    
    # Reverse process: Denoise
    for t in reversed(range(1, T + 1)):
        beta_t = noise_schedule[t-1]
        alpha_t = 1 - beta_t
        alpha_bar_t = product(1 - beta for beta in noise_schedule[:t])
        alpha_bar_prev = product(1 - beta for beta in noise_schedule[:t-1]) if t > 1 else 1
        
        # Predict noise or mean from diffusion model
        # The model takes noisy latent z_t, step t, and condition z_I
        predicted_noise = diffusion_model.predict_noise(z_current, t, z_I)
        
        # Compute mean for reverse step
        # Standard DDIM or DDPM reverse step formula
        # z_prev = 1/sqrt(alpha_t) * (z_t - (1-alpha_t)/sqrt(1-alpha_bar_t) * predicted_noise)
        # + sigma_t * epsilon_pred
        # For simplicity, using a standard reverse update:
        
        coeff1 = sqrt(alpha_bar_prev) * (1 - alpha_t) / (1 - alpha_bar_t)
        coeff2 = sqrt(alpha_t) * (1 - alpha_bar_prev) / (1 - alpha_bar_t)
        
        z_prev = coeff1 * z_current + coeff2 * predicted_noise
        
        # Add noise if t > 1, else deterministic
        if t > 1:
            sigma_t = sqrt((1 - alpha_bar_prev) / (1 - alpha_bar_t) * (1 - alpha_t))
            epsilon_t = random_gaussian_noise_like(z_current)
            z_prev = z_prev + sigma_t * epsilon_t
            
        z_current = z_prev
        
    # Step 4: Scale-Consistent Integration
    # Apply scale constraints during the process or post-process
    # This ensures the global scale matches the point cloud
    z_refined = apply_scale_consistency(z_current, D0)
    
    # Decode back to depth map
    refined_depth_map = latent_encoder_depth.decode(z_refined)
    
    return refined_depth_map

def project_point_cloud_to_depth(point_cloud, camera_params):
    """
    Projects 3D points to 2D depth map.
    Returns depth map with infinity for empty pixels.
    """
    H, W = camera_params.image_size
    depth_map = np.full((H, W), np.inf)
    
    # Project each point
    for point in point_cloud:
        # Transform point to camera coordinates
        point_cam = transform_to_camera(point, camera_params.extrinsics)
        
        # Project to 2D
        u, v = project_3d_to_2d(point_cam, camera_params.intrinsics)
        
        # Check bounds
        if 0 <= u < W and 0 <= v < H:
            depth = point_cam.z
            if depth < depth_map[v, u]:
                depth_map[v, u] = depth
                
    return depth_map
```

## Subtopics

### Generative Depth Refinement
A module that employs a diffusion model to integrate high-resolution RGB information into the point cloud, enhancing geometric details and smoothness. It is chosen when high-fidelity mesh reconstruction is required. It differs from standard hole-filling by using generative priors to hallucinate plausible geometric details consistent with the RGB texture. It includes a diffusion model for iterative refinement and a scale-consistent integration path to constrain the depth scale for each iteration [S1].

### Computational Depth Refinement
A method focused on efficiency for NeRF-based rendering. It involves removing unnecessary 3D points and filling holes in the projected depth image. It is chosen when the goal is to enable surface-based sampling for novel view synthesis rather than explicit mesh generation. It sets itself apart by prioritizing computational speed and robustness for sampling over high-frequency detail recovery [S2].

## Operating characteristics
- **Time Complexity**: The generative variant [S1] is computationally expensive due to the iterative diffusion process (T steps) and latent encoding/decoding. The computational variant [S2] is efficient, relying on simple projection and hole-filling.
- **Memory Complexity**: Requires storing the 3D point cloud, RGB images, and latent representations. The diffusion model adds significant memory overhead during inference [S1].
- **Strengths**: The generative variant produces scale-consistent, detail-rich depth and normal maps, overcoming the resolution limits of initial geometry estimators like DUSt3R [S1]. The computational variant enables efficient surface-based sampling, improving NeRF rendering quality with minimal overhead [S2].
- **Limitations**: The generative variant depends on the quality of the RGB prior and the diffusion model's training data. It may hallucinate details not present in the scene. The computational variant may leave artifacts if the point cloud is sparse or noisy [S2].
- **When to use**: Use the generative variant for high-fidelity 3D reconstruction and mesh extraction. Use the computational variant for efficient novel view synthesis with NeRF [S1, S2].

## Background
The concept of refining projected depth maps originated from the need to improve NeRF rendering by restricting sampling to the object surface [S2]. The generative approach was introduced in GBR (Generative Bundle Refinement) to address geometry accuracy challenges in Gaussian Splatting by integrating diffusion models with neural bundle adjustment [S1].

## Sources
[S1] GBR: Generative Bundle Refinement for High-Fidelity Gaussian Splatting With Enhanced Mesh Reconstruction, https://doi.org/10.1109/tcsvt.2025.3596613, fetched 2026-10-29
[S2] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
