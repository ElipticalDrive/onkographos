# VoxGRAF

Also called: Sparse Voxel Grid GAN, VoxGRAF Sparse Voxel Grid Generation

## Overview
VoxGRAF is a 3D-aware Generative Adversarial Network (GAN) that synthesizes images by generating radiance fields on a sparse voxel grid using a 3D convolutional neural network (CNN) [S1]. Unlike coordinate-based MLPs, it uses a single forward pass to generate a 3D scene, enabling efficient rendering via alpha composition [S1]. It distinguishes itself by combining progressive growing with on-the-fly pruning to maintain high sparsity, and by conditioning the generator on camera pose rather than view direction to model directional dependencies [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Latent code $z \sim \mathcal{N}(0, 1)$, camera pose $\xi$, and camera intrinsics $K$ [S1].
*   **Outputs:** A rendered RGB image $I$ and optionally an alpha mask [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D tensor representing the radiance field. The grid resolution $R_G$ grows from $32^3$ to $128^3$ [S1].
*   **Sparse Representation:** Voxels are pruned based on transmittance and density thresholds [S1].
*   **Feature Storage:** Each voxel stores color $\mathbf{c}$ and density $\sigma$ [S1].

**Numbered Steps:**

1.  **Generator Forward Pass:** The generator $G_\theta$ takes $z$, $\xi$, and $K$ as input and produces a 3D voxel grid containing colors and densities [S1]. The generator consists of 3D modulated convolutions and 3D upsampling, replacing 2D operations from StyleGAN2 [S1].
2.  **Ray Sampling:** For each pixel in the target image, a camera ray $\mathbf{r}$ is defined. $N$ sampling points $\mathbf{x}_i$ are generated along $\mathbf{r}$ [S1].
3.  **Interpolation:** For each sample point $\mathbf{x}_i$, the color $\mathbf{c}_{ri}$ and density $\sigma_{ri}$ are retrieved from the voxel grid via trilinear interpolation of the nearest eight vertices [S1].
4.  **Alpha Composition:** The color $\mathbf{c}_r$ and alpha mask $a_r$ for the ray are computed using alpha composition [S1]:
    $$
    \mathbf{c}_r = \sum_{i=1}^N T_{ri} \alpha_{ri} \mathbf{c}_{ri}
    $$
    $$
    a_r = \sum_{i=1}^N T_{ri} \alpha_{ri}
    $$
    where transmittance $T_{ri}$ and alpha $\alpha_{ri}$ are defined as:
    $$
    T_{ri} = \prod_{j=1}^{i-1} (1 - \alpha_{rj})
    $$
    $$
    \alpha_{ri} = 1 - \exp(-\sigma_{ri} \delta_{ri})
    $$
    Here, $\delta_{ri} = \|\mathbf{x}_{i+1}^r - \mathbf{x}_i^r\|_2$ is the distance between neighboring sample points [S1].
5.  **Refinement (Optional):** The rendered image may be refined by a shallow 2D CNN with 2 hidden layers of dimension 16 and kernel size 3, operating on 3 channels at full resolution [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxgraf_render(z, pose, K, grid, N_samples):
    """
    Renders an image from a generated voxel grid.
    
    Args:
        z: Latent code.
        pose: Camera pose.
        K: Camera intrinsics.
        grid: Voxel grid (sparse or dense) containing colors and densities.
        N_samples: Number of sampling points per ray.
        
    Returns:
        Rendered image.
    """
    # 1. Generate Voxel Grid
    # G_theta takes z, pose, K and outputs a 3D voxel grid
    # The grid contains color (c) and density (sigma) for each voxel
    voxel_grid = generator_forward(z, pose, K)
    
    # 2. Define Rays
    # For each pixel in the output image, define a ray r
    rays = define_rays_from_camera(K, pose, image_resolution)
    
    # 3. Sample Points
    # For each ray, generate N sampling points x_i
    sample_points = sample_along_rays(rays, N_samples)
    
    # 4. Interpolate Features
    # Retrieve color and density at each sample point via trilinear interpolation
    colors = []
    densities = []
    for point in sample_points:
        c, sigma = trilinear_interpolate(voxel_grid, point)
        colors.append(c)
        densities.append(sigma)
        
    # 5. Alpha Composition
    # Compute transmittance and alpha for each sample
    transmittances = []
    alphas = []
    current_T = 1.0
    for i in range(N_samples):
        sigma = densities[i]
        # Calculate distance between current and next sample point
        # delta_i = ||x_{i+1} - x_i||_2
        delta = distance(sample_points[i], sample_points[i+1] if i < N_samples-1 else sample_points[i])
        
        alpha = 1.0 - exp(-sigma * delta)
        alphas.append(alpha)
        transmittances.append(current_T)
        
        # Update transmittance for next sample
        current_T *= (1.0 - alpha)
        
    # 6. Composite Color
    # c_r = sum(T_i * alpha_i * c_i)
    rendered_image = zeros_like(image_resolution)
    for i in range(N_samples):
        rendered_image += transmittances[i] * alphas[i] * colors[i]
        
    # 7. Optional Refinement
    # Apply shallow 2D CNN if configured
    if use_refinement:
        rendered_image = refine_2d_cnn(rendered_image)
        
    return rendered_image

def voxgraf_train_step(batch_z, batch_poses, batch_images, batch_Ks, grid, optimizer_G, optimizer_D):
    """
    One step of GAN training.
    """
    # Generator Loss
    # Condition pose: 50% use ground truth pose, 50% random
    poses_cond = []
    for i, gt_pose in enumerate(batch_poses):
        if random() < 0.5:
            poses_cond.append(gt_pose)
        else:
            poses_cond.append(random_pose())
            
    # Generate images
    generated_images = [voxgraf_render(z, pose, K, grid, N_samples) 
                        for z, pose, K in zip(batch_z, poses_cond, batch_Ks)]
                        
    # Compute GAN loss (non-saturating)
    # V(theta, phi) = E[f(-D(G(z, xi, xi', K), xi'))] + ...
    # f(t) = -log(1 + exp(-t))
    disc_gen = discriminator(generated_images, poses_cond)
    gen_loss = -mean(log_sigmoid(disc_gen)) # Non-saturating variant
    
    # Add regularization losses (Depth Variance, TV, Coverage)
    reg_loss = compute_regularization(grid, generated_images)
    total_gen_loss = gen_loss + reg_loss
    
    # Update Generator
    optimizer_G.zero_grad()
    total_gen_loss.backward()
    optimizer_G.step()
    
    # Discriminator Loss
    disc_real = discriminator(batch_images, batch_poses)
    disc_fake = discriminator(generated_images, poses_cond)
    
    # R1 Regularization
    # lambda * ||grad D(I, xi)||^2
    r1_reg = compute_r1_regularization(discriminator, batch_images, batch_poses)
    
    # GAN Loss for Discriminator
    # E[f(D(I, xi))] - lambda * ||grad D(I, xi)||^2
    disc_loss = mean(log_sigmoid(disc_real)) - mean(log_sigmoid(disc_fake)) - lambda_r1 * r1_reg
    
    # Update Discriminator
    optimizer_D.zero_grad()
    disc_loss.backward()
    optimizer_D.step()
    
    return total_gen_loss, disc_loss
```

**Parameters:**
*   **Grid Resolution ($R_G$):** 32, 64, or 128 [S1].
*   **Image Resolution ($R_I$):** Grows from 32 to 128 [S1].
*   **Pruning Thresholds:** $\tau_T$ (transmittance) and $\tau_\sigma$ (density) [S1].
*   **Depth Variance Loss Threshold ($\tau$):** Hyperparameter defining surface thickness to avoid empty foreground [S1].
*   **R1 Regularization Strength ($\lambda$):** Controls R1-regularizer strength [S1].
*   **Learning Rates:** Generator: 0.0025, Discriminator: 0.002 [S1].
*   **Batch Size:** 64 [S1].

**Edge Cases and Pitfalls:**
*   **Empty Foreground:** Without depth variance loss, the foreground may become empty. Thresholding the loss with $\tau$ is crucial [S1].
*   **Pose Correlation:** Strong data correlations (e.g., eyes looking at camera) require pose conditioning. Randomizing pose during training (50% ground truth, 50% random) increases flexibility [S1].
*   **Sparse vs Dense Convolutions:** Sparse convolutions reduce memory but increase runtime due to coordinate management overhead. Dense convolutions with zeroed pruned voxels increase memory but reduce runtime. VoxGRAF prioritizes faster training, using dense convolutions with zeroed pruned voxels [S1].

## Operating characteristics
*   **Time Complexity:** Generation time is significantly reduced with sparser representations. On FFHQ ($R_G=128, R_I=128$), generation time is 198 ms with depth variance loss (95% sparsity) and 229 ms without (74% sparsity) [S1].
*   **Memory Complexity:** Memory consumption is 0.9 GB with depth variance loss (95% sparsity) and 1.4 GB without (74% sparsity) [S1].
*   **Rendering Time:** Rendering time is 4 ms with depth variance loss and 7 ms without [S1].
*   **Strengths:** Fast 3D-aware image synthesis, efficient rendering via alpha composition, high sparsity with depth variance loss [S1].
*   **Limitations:** Sparse convolutions increase runtime due to coordinate management overhead. Pose conditioning alone may not be sufficient for strong data correlations, requiring optional 2D refinement [S1].
*   **When to Use:** When fast 3D-aware image synthesis is required, and efficient rendering is a priority. Suitable for datasets with single objects per image [S1].
*   **When Not to Use:** When memory is extremely constrained and sparse convolutions' overhead is unacceptable, or when strong view-dependent effects are not correlated with pose [S1].

## Background
VoxGRAF was introduced in the paper "VoxGRAF: Fast 3D-Aware Image Synthesis with Sparse Voxel Grids" [S1]. It builds on the principles of 3D-aware GANs and sparse voxel grids, challenging the paradigm of coordinate-based MLPs by using a 3D CNN [S1]. The method combines progressive growing with pruning to scale voxel grids to high resolutions [S1].

## Sources
[S1] VoxGRAF: Fast 3D-Aware Image Synthesis with Sparse Voxel Grids, https://doi.org/10.52202/068431-2464, fetched 2026-10-26
