# SAMURAI Optimization Schedule

Also called: BARF-A baseline comparison, SAMURAI training procedure

## Overview
The SAMURAI optimization schedule is a staged training procedure for jointly estimating camera poses, geometry (shape), Bidirectional Reflectance Distribution Functions (BRDF), and illumination from unconstrained real-world image collections [S1]. It solves the problem of decomposing complex scenes captured under varying backgrounds and illuminations where ground-truth poses are unavailable or noisy [S1]. It distinguishes itself from alternatives like BARF by using a camera multiplexing strategy and specific fading weights ($\lambda$) to handle coarse pose initialization and high-frequency texture reconstruction, which BARF fails to do accurately when initialization is too noisy [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A collection of images captured under varying illuminations and viewpoints [S1].
*   Rough camera pose initialization (e.g., from COLMAP or direction-based initialization) [S1].
*   Volume bounds $v$ defining the spatial extent of the scene [S1].

**Outputs:**
*   Optimized camera parameters (extrinsics and intrinsics, including focal length) [S1].
*   A neural reflectance volume representing shape and BRDF [S1].
*   Illumination parameters [S1].
*   Explicit mesh with material properties (extracted post-training) [S1].

### Data Structures
The method relies on standard neural volume representations. Specific internal data structure layouts (e.g., grid resolution, hash table sizes) are not given in the sources [S1]. The optimization maintains separate parameter sets for the network weights and camera parameters [S1].

### Numbered Steps

1.  **Initialization:** Initialize the neural network and camera parameters. Use rough pose initialization (e.g., directions or COLMAP) [S1].
2.  **Optimization Loop:** Iterate through training steps $t$. Update parameters using two distinct Adam optimizers with different learning rates and decay schedules [S1]:
    *   **Network Optimizer:** Learning rate $\eta_n = 10^{-4}$, decayed by an order of magnitude every 300,000 steps [S1].
    *   **Camera Optimizer:** Learning rate $\eta_c = 3 \times 10^{-3}$, decayed by an order of magnitude every 70,000 steps [S1].
3.  **Schedule Transition (Fading Weights):** Apply three fading variables $\lambda_c, \lambda_b, \lambda_a$ to transition the optimization smoothly [S1]:
    *   $\lambda_c$: Controls image resolution and the number of active multiplex cameras. It increases resolution and reduces active cameras over time [S1].
    *   $\lambda_b$: Fades the optimization from direct color $\tilde{c}$ to BRDF parameters [S1].
    *   $\lambda_a$: Scales specific losses as defined in the broader method context [S1].
4.  **Frequency Annealing:** Perform BARF-style frequency annealing in the early stages of training to stabilize optimization [S1].
5.  **Delayed Focal Length:** Delay the optimization of the focal length to the later stages of training [S1].
6.  **Camera Multiplexing:** Use a multiplexing strategy for cameras to prevent them from getting stuck in local minima and to stabilize training [S1].
7.  **Volume Padding Loss:** Apply a volume padding loss to prevent cameras from moving too far outside the volume bounds $v$ [S1]:
    $$ L_{Bounds} = \max((v - |t_j|)^2, 0) $$
    where $t_j$ is the camera translation vector [S1].
8.  **Evaluation Protocol:**
    *   Hold out every 16th image for testing [S1].
    *   Optimize cameras and illuminations on test images independently.
    *   Do not allow test image optimization to affect the main decomposition network or camera training [S1].
    *   Perform Procrustes analysis on recovered camera poses to evaluate pose accuracy [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def samurai_optimization_schedule(images, rough_poses, volume_bound_v, num_steps=1000000):
    """
    Implements the SAMURAI optimization schedule.
    
    Args:
        images: List of input images.
        rough_poses: Initial camera poses (extrinsics/intrinsics).
        volume_bound_v: Scalar or vector defining volume bounds.
        num_steps: Total number of optimization steps.
        
    Returns:
        Optimized network weights, camera parameters, BRDF, and illumination.
    """
    # Initialize parameters
    network_weights = initialize_network()
    camera_params = initialize_cameras(rough_poses)
    brdf_params = initialize_brdf()
    illumination_params = initialize_illumination()
    
    # Optimizers
    # Network: Adam(lr=1e-4, decay every 300k steps)
    network_optimizer = Adam(network_weights, lr=1e-4, decay_step=300000, decay_factor=0.1)
    # Camera: Adam(lr=3e-3, decay every 70k steps)
    camera_optimizer = Adam(camera_params, lr=3e-3, decay_step=70000, decay_factor=0.1)
    
    # Schedule parameters
    # lambda_c: increases resolution, reduces active multiplex cameras
    # lambda_b: fades direct color c_tilde to BRDF
    # lambda_a: scales specific losses
    lambda_c = 0.0
    lambda_b = 0.0
    lambda_a = 1.0
    
    for step in range(num_steps):
        # 1. Update fading weights based on step
        # lambda_c increases over time to boost resolution and reduce active cameras
        lambda_c = update_lambda_c(step, num_steps)
        # lambda_b increases to transition from direct color to BRDF
        lambda_b = update_lambda_b(step, num_steps)
        
        # 2. Select batch of images
        batch_images = sample_batch(images)
        
        # 3. Render and compute losses
        # Use BARF frequency annealing in early stages
        frequencies = get_barf_frequencies(step)
        
        rendered_images = render_volume(
            network_weights, 
            camera_params, 
            brdf_params, 
            illumination_params, 
            frequencies=frequencies
        )
        
        # Compute reconstruction loss
        loss_recon = compute_reconstruction_loss(rendered_images, batch_images)
        
        # Apply fading weights
        # Note: Exact formula for combining lambda_b with direct color vs BRDF is not fully specified in sources [S1]
        # Implementation note: Blend direct color optimization and BRDF optimization using lambda_b
        loss_color = compute_color_loss(rendered_images, batch_images)
        loss_brdf = compute_brdf_loss(brdf_params)
        
        # Scale losses by lambda_a (details of specific losses scaled by lambda_a are not given [S1])
        total_loss = lambda_a * (loss_recon + loss_color + loss_brdf)
        
        # 4. Add Volume Padding Loss
        # Prevents cameras from going too far into/outside volume bound v
        # L_Bounds = max((v - |t_j|)^2, 0)
        loss_bounds = compute_volume_padding_loss(camera_params, volume_bound_v)
        total_loss += loss_bounds
        
        # 5. Update parameters
        # Network update
        network_optimizer.zero_grad()
        # Note: Losses dependent on network weights must be detached from camera optimizer graph
        network_loss = detach_camera_dependent_losses(total_loss)
        network_loss.backward()
        network_optimizer.step()
        
        # Camera update
        camera_optimizer.zero_grad()
        # Note: Losses dependent on camera params must be detached from network optimizer graph
        camera_loss = detach_network_dependent_losses(total_loss)
        camera_loss.backward()
        camera_optimizer.step()
        
        # 6. Delayed Focal Length
        # In later stages, enable focal length optimization
        if step > late_stage_threshold:
            enable_focal_length_optimization(camera_params)
            
    return network_weights, camera_params, brdf_params, illumination_params

def update_lambda_c(step, total_steps):
    # Increases over time to increase resolution and reduce active multiplex cameras
    # Exact function not given in sources [S1]
    return min(1.0, step / (total_steps * 0.5))

def update_lambda_b(step, total_steps):
    # Fades direct color optimization to BRDF optimization
    # Exact function not given in sources [S1]
    return min(1.0, step / (total_steps * 0.7))
```

### Edge Cases and Pitfalls
*   **High-Frequency Textures:** High-frequency repeating textures (e.g., the "cape" scene) produce slightly blurry results because reconstruction requires near-perfect camera poses, which are unavailable in challenging in-the-wild datasets [S1].
*   **Coarse Initialization:** If the initial pose is too noisy, the method may fail to align camera poses accurately, though the multiplexing strategy mitigates this compared to BARF [S1].
*   **Shadowing and Inter-reflections:** The BRDF and illumination decomposition does not model shadowing and inter-reflections, as the method primarily tackles object decomposition [S1].
*   **Evaluation Isolation:** Test images must not affect the main training loop; cameras and illuminations are optimized on test images independently for evaluation [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** Not explicitly given in the sources [S1]. The method uses two separate optimizers with different decay schedules, implying potentially higher computational overhead due to dual optimization loops [S1].
*   **Memory Complexity:** Not explicitly given in the sources [S1]. The method stores per-image appearances in latent vectors (similar to BARF-A) and maintains a neural reflectance volume [S1].

### Measured Performance
*   **Varying Illumination Datasets (No GT Poses):**
    *   SAMURAI: PSNR 23.46, SSIM 0.90 [S1].
    *   BARF-A: PSNR 16.9, SSIM 0.79 [S1].
*   **Varying Illumination Datasets (With Poses):**
    *   SAMURAI: PSNR 22.84, SSIM 0.89 [S1].
    *   BARF-A: PSNR 19.7, SSIM 0.89 [S1].
*   **Ablation Study (Garbage Truck/NeRD Car):**
    *   Full Method: PSNR 24.31, SSIM 0.92 [S1].
    *   Without Regularization: PSNR 21.77, SSIM 0.86 [S1].
*   **Single Illumination Datasets:**
    *   SAMURAI (Directions Init): PSNR 21.08, SSIM 0.76 [S1].
    *   GNeRF (Random Init): PSNR 20.3, SSIM 0.61 [S1] (Note: Table 3 in [S1] lists GNeRF PSNR as 20.3 and SSIM as 0.61, but the text says "GNeRF (random init) PSNR 20.3 and SSIM 0.61". The table also lists NeRD/Neural-PIL with GT poses having higher scores).

### Scaling
*   The method is designed for unconstrained real-world image collections, including online image collections where COLMAP fails [S1].
*   It scales to AR/VR, gaming, and material editing applications by extracting explicit meshes [S1].

### Strengths and Limitations
*   **Strengths:**
    *   Handles varying illuminations and backgrounds [S1].
    *   Works without GT poses, outperforming BARF-A significantly [S1].
    *   Enables BRDF decomposition and relighting [S1].
    *   Extracts explicit meshes for downstream applications [S1].
*   **Limitations:**
    *   Requires rough pose initialization (does not work from scratch like GNeRF on simple scenes, but GNeRF fails on challenging in-the-wild datasets) [S1].
    *   Produces blurry results for high-frequency textures due to imperfect pose estimation [S1].
    *   Does not model shadowing and inter-reflections [S1].

### When to Use
*   Use when joint estimation of camera, shape, BRDF, and illumination is required from unconstrained, in-the-wild image collections with varying illuminations [S1].
*   Use when ground-truth poses are unavailable or noisy, and COLMAP fails [S1].
*   Use when explicit mesh extraction with material properties is needed for AR/VR or gaming [S1].

### When Not to Use
*   Do not use if shadowing and inter-reflections are critical to the scene representation [S1].
*   Do not use if near-perfect reconstruction of high-frequency repeating textures is required and perfect poses are not available [S1].

## Background

### Origin
The SAMURAI optimization schedule was introduced in the paper "SAMURAI: Shape and Material from Unconstrained Real-World Arbitrary Image Collections" [S1].

### History and Development
The method was developed to address the limitations of existing techniques (like BARF-A) in handling coarse pose initialization and complex BRDF decomposition in unconstrained settings [S1]. It builds on BARF's frequency annealing but adds camera multiplexing and specific fading schedules to improve robustness [S1].

### Usage
The method is used for novel view synthesis, relighting, and extracting explicit meshes with material properties for applications in AR/VR, gaming, and material editing [S1].

## Sources

[S1] SAMURAI: Shape and Material from Unconstrained Real-World Arbitrary Image Collections, https://doi.org/10.52202/068431-1914, fetched 2026-10-29
