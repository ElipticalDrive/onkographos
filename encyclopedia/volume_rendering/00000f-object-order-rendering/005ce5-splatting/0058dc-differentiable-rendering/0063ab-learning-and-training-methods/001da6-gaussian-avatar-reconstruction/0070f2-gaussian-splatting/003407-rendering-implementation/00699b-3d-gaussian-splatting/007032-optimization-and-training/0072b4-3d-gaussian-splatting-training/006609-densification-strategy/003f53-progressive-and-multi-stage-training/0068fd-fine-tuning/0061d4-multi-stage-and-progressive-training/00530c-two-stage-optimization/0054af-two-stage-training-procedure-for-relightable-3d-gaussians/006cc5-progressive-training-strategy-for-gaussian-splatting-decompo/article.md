# Progressive Training Strategy for Gaussian Splatting Decomposition

Also called: Multi-stage optimization for intrinsic decomposition, Staged optimization for geometry, albedo, and lighting

## Overview
This method is a multi-stage optimization strategy designed to decompose scene geometry, appearance (albedo), and illumination from multi-view images using 3D Gaussian Splatting [S1]. It prevents the model from overfitting to appearance details and stabilizes the learning of lighting parameters by decoupling the optimization of geometry, albedo, and lighting into distinct phases. It builds on [Two-stage training procedure for Relightable 3D Gaussians](15262-two-stage-training-procedure-for-relightable-3d-gaussians.md) by extending the decomposition to include explicit lighting priors and a three-stage refinement process.

## How it works
The method optimizes a set of 3D Gaussians to represent the scene, decomposing them into geometric attributes (position, covariance, opacity) and non-geometric attributes (albedo, view-dependent appearance). The optimization is divided into three sequential stages to ensure stable convergence.

### Inputs and Outputs
**Inputs:**
*   A set of multi-view images of the scene.
*   Camera poses for each input image.
*   (Optional) Semantic masks for the input images [S1].
*   (Optional) Ground truth normal maps for synthetic scenes [S1].

**Outputs:**
*   Optimized 3D Gaussian attributes:
    *   Geometry: Position $\mathbf{p}_k$, covariance $\mathbf{\Sigma}_k$, opacity $\alpha_k$.
    *   Appearance: Albedo $\mathbf{c}_{a,k}$, view-dependent coefficients (if applicable).
    *   Embeddings: Per-Gaussian embedding $\Lambda_k$ and per-image embedding $\Xi_j$.
*   Lighting parameters: Sunlight direction and color, ambient irradiance map.

### Data Structures
The appearance transformation module uses an MLP $f_{app}$ to map embeddings to appearance.
*   Per-Gaussian embedding $\Lambda_k$: Size 24 [S1].
*   Per-image embedding $\Xi_j$: Size 32 [S1].
*   Appearance MLP $f_{app}$: One hidden layer of size 128 [S1].

### Loss Functions
The total loss is a weighted sum of several components [S1]:
$$ L = \lambda_1 (L_1 + L_{D-SSIM}) + \lambda_2 (L_{normal} + L_{distortion}) + \lambda_3 L_{np} + \lambda_4 L_{mask} + \lambda_5 L_{sp} + \lambda_6 L_{amb} $$

Where:
*   $L_1$: L1 loss between rendered and ground truth images.
*   $L_{D-SSIM}$: D-SSIM loss for rendered images.
*   $L_{normal}$: Normal consistency loss.
*   $L_{distortion}$: Distortion loss.
*   $L_{np}$: Normal prior loss.
*   $L_{mask}$: Semantic mask loss.
*   $L_{sp}$: Sunlight color prior loss.
*   $L_{amb}$: Ambient irradiance map loss.

### Parameters
*   $\lambda_1 = 1.0$ for $L_1$ and $L_{D-SSIM}$ [S1].
*   $\lambda_2 = 0.05$ for $L_{normal}$ and $L_{distortion}$ [S1].
*   $\lambda_3 = 0.1$ for $L_{np}$ [S1].
*   $\lambda_4 = 0.1$ for $L_{mask}$ [S1].
*   $\lambda_5 = 10.0$ for $L_{sp}$, decaying to $0.01$ over 50,000 iterations [S1].
*   $\lambda_6 = 1.0$ for $L_{amb}$ [S1].

### Steps
The training proceeds in three stages [S1]:

1.  **Stage 1: Geometry Optimization**
    *   Optimize only geometry-related Gaussian attributes (position, covariance, opacity).
    *   Non-geometric attributes are initialized but not optimized.
    *   This stage establishes a robust geometric foundation.

2.  **Stage 2: Albedo and Lighting Decomposition**
    *   Freeze all geometry-related Gaussian attributes.
    *   **Stage 2a (Iterations 0–10,000):**
        *   Omit shading from the rendering process. This allows the model to learn an initial, refined albedo without being influenced by lighting variations [S1].
        *   Optimize non-geometric Gaussian attributes (albedo, embeddings).
    *   **Stage 2b (Iterations 10,000–50,000):**
        *   Freeze all Gaussian attributes, including the newly learned albedo.
        *   Optimize only lighting parameters (sunlight direction/color, ambient irradiance) [S1].
        *   This isolates lighting optimization, preventing interference from appearance updates.

3.  **Stage 3: Joint Fine-tuning**
    *   **Iterations 50,000–100,000:**
        *   Unfreeze non-geometric Gaussian attributes and lighting parameters.
        *   Jointly optimize all non-geometric attributes and lighting parameters [S1].
        *   Geometry remains frozen.
        *   This stage fine-tunes the decomposition, allowing albedo and lighting to adjust to each other.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_training_gaussian_decomposition(images, poses, num_iterations=100000):
    """
    Implements the three-stage progressive training strategy for Gaussian Splatting decomposition.
    """
    # Initialization
    gaussians = initialize_3d_gaussians() # Positions, covariances, opacities, albedos, embeddings
    lighting = initialize_lighting()      # Sunlight direction/color, ambient irradiance
    
    # Hyperparameters
    lambda1 = 1.0
    lambda2 = 0.05
    lambda3 = 0.1
    lambda4 = 0.1
    lambda5_start = 10.0
    lambda5_end = 0.01
    lambda6 = 1.0
    
    # Stage 1: Geometry Optimization
    # Optimize only geometry-related attributes
    for i in range(10000): # Assuming equal stage lengths for simplicity, adjust as needed
        # Freeze non-geometry attributes
        gaussians.albedo.requires_grad = False
        gaussians.embeddings.requires_grad = False
        lighting.requires_grad = False
        
        # Forward pass with shading
        rendered_image = render(gaussians, lighting, poses)
        
        # Compute losses
        loss = compute_losses(rendered_image, images, gaussians, lighting, 
                              lambda1, lambda2, lambda3, lambda4, lambda5_start, lambda6, 
                              shading_enabled=True)
        
        # Backward pass
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    # Stage 2a: Albedo Learning (No Shading)
    # Freeze geometry, optimize albedo/embeddings, no shading
    for i in range(10000, 20000):
        # Freeze geometry
        gaussians.position.requires_grad = False
        gaussians.covariance.requires_grad = False
        gaussians.opacity.requires_grad = False
        # Freeze lighting
        lighting.requires_grad = False
        
        # Forward pass WITHOUT shading
        rendered_image = render(gaussians, lighting, poses, shading_enabled=False)
        
        # Compute losses (shading-related lighting losses may be skipped or modified)
        loss = compute_losses(rendered_image, images, gaussians, lighting, 
                              lambda1, lambda2, lambda3, lambda4, lambda5_start, lambda6, 
                              shading_enabled=False)
        
        # Backward pass
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

    # Stage 2b: Lighting Optimization
    # Freeze all Gaussian attributes, optimize lighting only
    for i in range(20000, 60000):
        # Freeze all Gaussian attributes
        gaussians.position.requires_grad = False
        gaussians.covariance.requires_grad = False
        gaussians.opacity.requires_grad = False
        gaussians.albedo.requires_grad = False
        gaussians.embeddings.requires_grad = False
        
        # Forward pass with shading
        rendered_image = render(gaussians, lighting, poses, shading_enabled=True)
        
        # Compute losses
        # Decay lambda5 over iterations
        progress = (i - 20000) / 40000.0
        lambda5_current = lambda5_start + (lambda5_end - lambda5_start) * progress
        
        loss = compute_losses(rendered_image, images, gaussians, lighting, 
                              lambda1, lambda2, lambda3, lambda4, lambda5_current, lambda6, 
                              shading_enabled=True)
        
        # Backward pass
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

    # Stage 3: Joint Fine-tuning
    # Optimize non-geometry attributes and lighting
    for i in range(60000, 110000): # Adjusted to match 50k iterations mentioned in source
        # Geometry remains frozen
        gaussians.position.requires_grad = False
        gaussians.covariance.requires_grad = False
        gaussians.opacity.requires_grad = False
        
        # Forward pass with shading
        rendered_image = render(gaussians, lighting, poses, shading_enabled=True)
        
        # Compute losses
        # Continue decay of lambda5 if needed, or keep at end value
        progress = (i - 60000) / 50000.0
        lambda5_current = lambda5_end # Or continue decay if source implies it
        
        loss = compute_losses(rendered_image, images, gaussians, lighting, 
                              lambda1, lambda2, lambda3, lambda4, lambda5_current, lambda6, 
                              shading_enabled=True)
        
        # Backward pass
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

def compute_losses(rendered, ground_truth, gaussians, lighting, l1_w, l2_w, l3_w, l4_w, l5_w, l6_w, shading_enabled):
    """
    Computes the total loss based on the specified weights and components.
    """
    loss_l1 = l1_w * L1Loss(rendered, ground_truth)
    loss_dssim = l1_w * DSSIMLoss(rendered, ground_truth)
    
    loss_normal = l2_w * NormalConsistencyLoss(gaussians)
    loss_distortion = l2_w * DistortionLoss(gaussians)
    
    loss_np = l3_w * NormalPriorLoss(gaussians)
    loss_mask = l4_w * SemanticMaskLoss(rendered, ground_truth_masks) # Assuming masks available
    
    loss_sp = l5_w * SunlightColorPriorLoss(lighting)
    loss_amb = l6_w * AmbientIrradianceMapLoss(lighting)
    
    total_loss = loss_l1 + loss_dssim + loss_normal + loss_distortion + loss_np + loss_mask + loss_sp + loss_amb
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** The method requires a total of 100,000 iterations (10k + 10k + 40k + 50k) [S1]. The per-iteration cost is similar to standard 3DGS but with additional loss computations for normals and lighting priors.
*   **Memory Complexity:** Uses standard 3DGS memory plus storage for per-Gaussian (size 24) and per-image (size 32) embeddings, and an MLP for appearance transformation [S1].
*   **Performance:** On synthetic SOL-NeRF datasets, it achieves competitive PSNR and SSIM scores compared to baselines like LumiGauss, ReCap, and SOL-NeRF [S1].
*   **Strengths:** Stabilizes lighting optimization, prevents overfitting to appearance, and produces high-quality intrinsic decomposition.
*   **Limitations:** Requires careful tuning of stage durations and loss weights. Performance on real-world datasets may vary due to lack of ground truth normals/texture [S1].

## Background
The method is part of the ROS-GS (Relightable Outdoor Scenes With Gaussian Splatting) framework [S1]. It addresses the challenge of decomposing relightable scenes from multi-view images by introducing a progressive training strategy that separates the optimization of geometry, albedo, and lighting. This approach builds upon earlier works in differentiable rendering and Gaussian Splatting, such as [Two-stage training procedure for Relightable 3D Gaussians](15262-two-stage-training-procedure-for-relightable-3d-gaussians.md), by adding more granular control over the decomposition process.

## Sources
[S1] ROS-GS: Relightable Outdoor Scenes With Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3709027, fetched 2026-10-26
