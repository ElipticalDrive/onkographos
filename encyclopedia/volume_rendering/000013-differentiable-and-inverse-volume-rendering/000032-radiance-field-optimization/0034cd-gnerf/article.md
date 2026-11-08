# GNeRF

Also called: GAN-based Neural Radiance Field without Posed Camera

## Overview
GNeRF is a two-phase, end-to-end trainable framework that jointly optimizes coarse camera poses and neural radiance fields using adversarial training, then refines them with photometric loss, enabling novel view synthesis from randomly initialized poses in complex scenes [S1]. Unlike methods that require short trajectories or rough initialization (e.g., iNeRF, NeRF++), GNeRF handles complex outside-in scenarios and scenes with low texture or repeated patterns where traditional Structure-from-Motion (SfM) fails [S1]. It builds on [Radiance-field optimization](44-radiance-field-optimization.md) by introducing a GAN-based generator-discriminator pair and an inversion network to decouple pose estimation from direct photometric gradient propagation, which is prone to local minima [S1].

## How it works
GNeRF operates in two alternating phases: Phase A (Pose-free NeRF Estimation via Adversarial Training) and Phase B (Regularized Learning Strategy via Photometric Loss). The optimization follows an iterative sequence: `A, AB...AB, B` [S1].

### Inputs and Outputs
*   **Inputs:** A set of uncalibrated RGB images $I = \{I_1, \dots, I_n\}$ of a single scene. No camera poses or intrinsics are provided initially.
*   **Outputs:** A trained neural radiance field $F_\Theta$ and estimated camera poses $\Phi = \{\phi_1, \dots, \phi_n\}$ for the input images.

### Data Structures and Parameters
*   **Camera Pose $\phi$:** Represented as a latent variable.
    *   Translation $t \in \mathbb{R}^3$: A 3D embedding vector in Euclidean space [S1].
    *   Rotation $r \in \mathbb{R}^6$: A continuous 6D embedding vector derived from a rotation matrix $R \in \mathbb{R}^{3 \times 3}$ by dropping its last column. This avoids the discontinuities of quaternions and Euler angles [S1]. The original rotation matrix can be recovered via a Gram-Schmidt-like process [S1].
*   **NeRF Model $F_\Theta$:** An MLP mapping $(x, d) \to (c, \sigma)$, where $x \in \mathbb{R}^3$ is 3D location, $d \in [-\pi, \pi]^2$ is viewing direction, $c \in \mathbb{R}^3$ is color, and $\sigma$ is density [S1].
    *   Architecture: Original NeRF architecture with hierarchical sampling. Coarse and importance sampling both use 64 points. MLP dimension is increased from 256 to 360 to keep parameter count comparable to original NeRF while aligning "coarse" and "fine" pose spaces [S1].
*   **Generator $G$:** Synthesizes images from random poses $\phi$ and $F_\Theta$. Uses the same MLPs for coarse and fine sampling to ensure pose space alignment [S1].
*   **Discriminator $D$:** Distinguishes real image patches from generated ones. Follows GRAF architecture with instance normalization and spectral normalization [S1].
*   **Inversion Network $E$:** Maps image patches to camera poses. Built using a Vision Transformer [S1]. Last layer modified to output $\phi$ [S1].
*   **Camera Model:** SIMPLE PINHOLE with shared intrinsics. Typical values: focal length $f = 1111.111$, principal point $cx = 400, cy = 400$ [S1].
*   **Loss Weight $\lambda$:** Weight for the L2 penalty in the hybrid loss. Typical value not specified in the sources [S1].

### Numbered Steps

**Initialization:**
1.  Set all camera extrinsics to an identity matrix [S1].
2.  Define prior pose distributions for random sampling in Phase A:
    *   Synthetic-NeRF: Cameras uniformly distributed at the upper hemisphere towards the origin. Rotation computed from position and lookat point [S1].
    *   DTU: Cameras uniformly distributed at the upper hemisphere with azimuth range $[0, 150]$. Lookat point distributed as $N(0, 0.012)$ [S1].

**Phase A: Pose-free NeRF Estimation (Adversarial Training)**
3.  **Dynamic Patch Sampling:** Sample image patches of fixed size $16 \times 16$ with dynamic scale and random offset from the image domain [S1].
4.  **Generator Training:** For a random pose $\phi \sim P(\phi)$, generate image $\hat{I} = G(\phi; F_\Theta)$. Decompose $\hat{I}$ into patches. Minimize adversarial loss $L_A$ against real patches $P_d(I)$ sampled from training images [S1].
    $$ \min_\Theta \max_\eta L_A(\Theta, \eta) = \mathbb{E}_{I \sim P_d}[\log(D(I; \eta))] + \mathbb{E}_{\hat{I} \sim P_g}[\log(1 - D(\hat{I}; \eta))] $$
    where $P_g(I|\Theta) = \int_\phi G(\phi; F_\Theta) P(\phi) d\phi$ [S1].
5.  **Inversion Network Training:** Sample static patches of fixed size $64 \times 64$ uniformly across the whole image domain from generated images $G(\phi; F_\Theta)$ [S1]. Train $E$ to map these patches to $\phi$ using self-supervised loss:
    $$ L_E(\theta_E) = \mathbb{E}_{\phi \sim P(\phi)} \| E(G(\phi; F_\Theta); \theta_E) - \phi \|_2^2 $$
    [S1].
6.  **Intrinsic Scaling:** Scale camera intrinsics at the beginning to maximize receptive field, then progressively increase to original value to concentrate on fine details [S1].
7.  Repeat steps 3-6 until convergence.
8.  **Initial Pose Estimation:** Apply $E$ to real training images $I_i$ to get initial poses $\phi_i^0 = E(I_i)$ [S1].

**Phase B: Regularized Learning Strategy (Refinement)**
9.  **Static Patch Sampling:** Sample patches of size $64 \times 64$ uniformly across the image domain from real images $I_i$ [S1].
10. **Hybrid Loss Optimization:** Optimize $F_\Theta$ and $\Phi$ jointly by minimizing the hybrid loss $L_R$:
    $$ L_R(\Theta, \Phi) = L_N(\Theta, \Phi) + \lambda \sum_{i=1}^n \| E(I_i; \theta_E) - \phi_i \|_2^2 $$
    where $L_N(\Theta, \Phi) = \frac{1}{n} \sum_{i=1}^n \| I_i - \hat{I}_i \|_2^2$ is the photometric loss and $\hat{I}_i = G(\phi_i; F_\Theta)$ [S1].
11. **Iterative Interleaving:** Alternate between Phase A and Phase B in the pattern `A, AB...AB, B`. The `AB...AB` sequence helps overcome local minima by using the inversion network's pose predictions to regularize the gradient descent [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def GNeRF_Train(images, num_iterations):
    # Initialization
    theta_NeRF = initialize_NeRF()
    phi = [identity_matrix() for _ in images]
    theta_E = initialize_ViT()
    
    # Phase A: Adversarial Training
    for iter_A in range(num_iterations):
        # Dynamic Patch Sampling for GAN
        real_patches = sample_dynamic_patches(images, size=16)
        
        # Sample random poses
        phi_rand = sample_prior_pose()
        
        # Generate images and patches
        gen_images = render_NeRF(phi_rand, theta_NeRF)
        gen_patches = extract_patches(gen_images, size=16)
        
        # Update Discriminator and Generator
        update_Discriminator(real_patches, gen_patches, theta_D)
        update_Generator(gen_patches, theta_NeRF)
        
        # Inversion Network Training (Static Patches from Generated Images)
        static_gen_patches = extract_patches(gen_images, size=64, uniform=True)
        loss_E = mean_squared_error(theta_E(static_gen_patches), phi_rand)
        update_InversionNetwork(loss_E, theta_E)
        
        # Intrinsic Scaling Schedule
        scale_intrinsics(iter_A)

    # Initial Pose Estimation for Real Images
    real_patches_static = extract_patches(images, size=64, uniform=True)
    phi = theta_E(real_patches_static)

    # Phase B: Regularized Refinement with Interleaving
    for iter_main in range(num_iterations):
        if iter_main % 2 == 0:
            # Phase A Step (Regularization)
            real_patches_dyn = sample_dynamic_patches(images, size=16)
            phi_rand = sample_prior_pose()
            gen_images = render_NeRF(phi_rand, theta_NeRF)
            gen_patches = extract_patches(gen_images, size=16)
            update_Generator(gen_patches, theta_NeRF)
            static_gen_patches = extract_patches(gen_images, size=64, uniform=True)
            loss_E = mean_squared_error(theta_E(static_gen_patches), phi_rand)
            update_InversionNetwork(loss_E, theta_E)
        else:
            # Phase B Step (Photometric + L2 Penalty)
            # Static patches for pose consistency check
            real_patches_static = extract_patches(images, size=64, uniform=True)
            pose_pred = theta_E(real_patches_static)
            
            # Render current poses
            rendered_images = render_NeRF(phi, theta_NeRF)
            
            # Photometric Loss
            loss_N = mean_squared_error(images, rendered_images)
            
            # Hybrid Loss
            loss_L2 = sum_squared_error(pose_pred, phi)
            loss_R = loss_N + lambda_weight * loss_L2
            
            update_NeRF_and_Poses(loss_R, theta_NeRF, phi)

    return theta_NeRF, phi
```

### Edge Cases and Pitfalls
*   **Local Minima:** Directly propagating photometric loss to camera poses is sensitive to low texture and varying lighting, leading to local minima [S1]. The iterative `AB...AB` scheme mitigates this [S1].
*   **Pose Representation:** Quaternions and Euler angles are discontinuous and difficult for neural networks to learn; the 6D embedding is used instead [S1].
*   **Inversion Network Limitations:** The inversion network accepts limited-size patches ($64 \times 64$) for efficiency, restricting pose accuracy on scenes with sufficient keypoints compared to COLMAP [S1].
*   **Component Dependency:** Removing the adversarial loss or inversion network prevents learning correct geometry and poses. Removing photometric loss restricts the model to coarse radiance fields [S1].

## Operating characteristics
*   **Time Complexity:** Training takes approximately 30 hours per scene on a single Nvidia RTX TITAN GPU with an Intel i7-8700K CPU [S1].
*   **Memory Complexity:** Uses dynamic patch sampling ($16 \times 16$) for GAN training to reduce memory footprint compared to full-image rendering [S1].
*   **Performance:** On Synthetic-NeRF and DTU datasets, GNeRF achieves PSNR/SSIM comparable to COLMAP-based NeRF with joint refinement (C+r) and outperforms it in challenging scenes with repeated patterns or low texture [S1]. It outperforms IDR on some DTU scenes but has lower translation accuracy than COLMAP on scenes with rich keypoints [S1].
*   **Strengths:** Robust to challenging scenes (low texture, repeated patterns) where SfM fails; does not require masks or noisy initialization [S1].
*   **Limitations:** Pose accuracy is lower than COLMAP on standard scenes with reliable keypoints due to the inversion network's limited receptive field [S1].

## Background
GNeRF was introduced in the paper "GNeRF: GAN-based Neural Radiance Field without Posed Camera" [S1]. It extends the NeRF framework to jointly optimize 3D representation and camera poses in complex scenes with large displacements and unknown poses, leveraging Generative Adversarial Networks for pose-free estimation [S1].

## Sources
[S1] GNeRF: GAN-based Neural Radiance Field without Posed Camera, https://doi.org/10.1109/iccv48922.2021.00629, fetched 2026-10-26
