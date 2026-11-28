# Diffusion Prior Guidance for Pseudo Views

Also called: 3D Gaussian Splatting with Diffusion Prior Regularization, Pseudo-view Regularization, SGD Training

## Overview
This method is a training procedure for 3D Gaussian Splatting (3DGS) that uses a fine-tuned Diffusion Model to generate guidance images from randomly sampled pseudo-views, regularizing the Gaussian model parameters to improve unseen view synthesis [S1]. It sets itself apart from standard 3DGS by introducing a diffusion-based prior loss computed on rendered views that do not correspond to ground-truth images, thereby preventing the model from getting stuck in local minima and enhancing the scene's comprehensive observation [S1]. Unlike methods relying on Structure from Motion (SfM) point clouds for initialization, this approach exclusively uses LiDAR data to avoid prior bias [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   LiDAR point clouds (voxelized-downsampled) [S1].
    *   Monocular training images and their poses [S1].
    *   A fine-tuned Diffusion Model (specifically a ControlNet variant or image-to-image diffusion model) [S1].
    *   A Variational Autoencoder (VAE) associated with the Diffusion Model for latent space encoding and decoding [S1].
*   **Outputs:**
    *   Optimized 3D Gaussian Splatting parameters $\Phi$ (means $\mu$, covariances $\Sigma$, opacities $\alpha$, and spherical harmonics coefficients for color $c$) [S1].

### Data Structures
The 3D Gaussian Splatting representation consists of a set of Gaussians [S1]. Each Gaussian $i$ is parameterized by:
*   Mean position $\mu_i \in \mathbb{R}^3$ [S1].
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (typically parameterized via a rotation matrix and scaling vector) [S1].
*   Opacity $\alpha_i \in [0, 1]$ [S1].
*   Spherical Harmonics coefficients for view-dependent RGB color $c_i$ [S1].

The total parameter set is denoted as $\Phi = \bigcup \{\Phi_o, \Phi_p\}$, where $\Phi_o$ are parameters visible to the current training view $v_o$, and $\Phi_p$ are parameters visible to the pseudo-views $v_p$ [S1].

### Parameters and Typical Values
*   **Training Iterations:** 50,000 [S1].
*   **Learning Rate:** Decays from $1.6 \times 10^{-4}$ to $1.6 \times 10^{-6}$ [S1].
*   **Pseudo-view Sampling Interval ($k$):** Every 10 iterations [S1].
*   **Number of Pseudo Views ($M$):** 4 [S1].
*   **Warm-up Period:** 500 iterations without pseudo-view sampling [S1].
*   **Maximum Noise Level ($t_{max}$):** 10 [S1].
*   **Minimum Noise Level ($t_{min}$):** Not given in the sources.
*   **Strength Hyper-parameter Range ($s$):**
    *   $s_{min} = 0.2$ [S1].
    *   $s_{max}$ starts at 0.6 and linearly decreases to 0.4 over the training process [S1].
*   **Noise Level Formula:** $t = s \times t_{max}$, where $s$ is sampled uniformly from $[s_{min}, s_{max}]$ for each pseudo view [S1].
*   **Pseudo-view Orientation Rotation ($\delta$):** The orientation is rotated from training views within range $[-\delta, \delta]$ along the z-axis (yaw angle) [S1].

### Algorithm Steps

1.  **Initialization:**
    *   Voxelize and downsample the LiDAR point clouds with a voxel size of 0.5 [S1].
    *   Initialize the 3D Gaussian Splatting model using the voxelized LiDAR point clouds [S1].
    *   Do not use SfM (Structure from Motion) point clouds for initialization [S1].

2.  **Training Loop (for iteration $i = 1$ to 50,000):**

    3.  **Warm-up Check:**
        *   If $i \le 500$, skip pseudo-view sampling. Proceed only to Step 4.

    4.  **Reconstruction Loss Computation (Standard 3DGS):**
        *   Sample a training view $v_o$ [S1].
        *   Render the view using differentiable rasterization/splatting to obtain the rendered image $I_o$ and rendered depth map $\tilde{D}_o$ [S1].
        *   Obtain the ground-truth image $I_{gt}$ and the completed dense depth map $D_o$ (from LiDAR projection) for $v_o$ [S1].
        *   Compute the reconstruction loss $L_{recon}$:
            $$ L_{recon}(\Phi_o) = \mathbb{E}_{v_o} \left[ \| I_o - I_{gt} \|_1 + \lambda_{SSIM} L_{SSIM}(I_o, I_{gt}) + \lambda_{depth} \| D_o - \tilde{D}_o \|_1 \right] $$
            where $\Phi_o$ are the Gaussian parameters visible to $v_o$ [S1].

    5.  **Pseudo-View Sampling (if $i > 500$ and $i \pmod k == 0$):**
        *   Randomly sample $M$ pseudo views $v_p = \{v_p^j\}_{j=0}^{M-1}$ [S1].
        *   **Position:** Interpolate the position of each pseudo view between the current training view $v_o$ and its adjacent views [S1].
        *   **Orientation:** Rotate the orientation of each pseudo view from the training view's orientation within the range $[-\delta, \delta]$ along the z-axis (yaw angle) [S1].

    6.  **Pseudo-View Rendering:**
        *   Render the pseudo views using differentiable splatting with parameters $\Phi$ to obtain rendered pseudo-view images $\tilde{I}_p(\Phi, v_p) \in \mathbb{R}^{M \times 3 \times H \times W}$ and depth maps $\tilde{D}_p$ [S1].
        *   Let $\Phi_p$ be the union of Gaussian parameters visible to any of the pseudo views $v_p$ [S1].

    7.  **Diffusion Guidance Generation:**
        *   For each rendered pseudo-view image $\tilde{I}_p^j$:
            *   Encode $\tilde{I}_p^j$ into a latent feature map $z$ using the VAE Encoder [S1].
            *   Sample a strength value $s$ uniformly from $[s_{min}, s_{max}]$ [S1].
            *   Calculate the noise level $t = s \times t_{max}$ [S1].
            *   Add noise to the latent feature map $z$ at level $t$ to obtain the noisy latent $z_t$ [S1].
            *   Denoise $z_t$ using the fine-tuned Diffusion Model from level $t$ down to $t_{min}$ [S1].
            *   Decode the denoised latent to obtain the guidance image $I_g^j$ [S1].

    8.  **Pseudo Loss Computation:**
        *   Compute the pseudo loss $L_{pseudo}$ for the set of pseudo views:
            $$ L_{pseudo}(\Phi_p) = \mathbb{E}_{v_p} \left[ \| I_g - \tilde{I}_p \|_1 + \lambda_{p-lpips} L_{lpips}(I_g, \tilde{I}_p) + \lambda_{p-depth} \| D_p - \tilde{D}_p \|_1 \right] $$
            where $I_g$ and $D_p$ are the guidance images and corresponding target depth maps for the pseudo views, and $\tilde{I}_p, \tilde{D}_p$ are the rendered images and depths [S1].

    9.  **Total Loss and Optimization:**
        *   Compute the total loss:
            $$ L(\Phi) = L_{recon}(\Phi_o) + \lambda_{pseudo} L_{pseudo}(\Phi_p) $$
            where $\Phi = \text{Union}\{\Phi_o, \Phi_p\}$ [S1].
        *   Perform gradient descent to update $\Phi$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def train_3dgs_with_diffusion_prior(
    lidar_points, 
    training_images, 
    training_poses, 
    diffusion_model, 
    vae_encoder, 
    vae_decoder,
    num_iterations=50000,
    voxel_size=0.5,
    k=10,
    M=4,
    t_max=10,
    s_min=0.2,
    s_max_init=0.6,
    s_max_final=0.4,
    lambda_ssim=0.2,
    lambda_depth=1.0,
    lambda_pseudo=1.0,
    lambda_p_lpips=0.1,
    lambda_p_depth=1.0
):
    # 1. Initialization
    # Voxelize and downsample LiDAR points with voxel size 0.5 [S1]
    voxelized_points = voxelize_downsample(lidar_points, voxel_size)
    
    # Initialize 3D Gaussian Splatting model with voxelized points [S1]
    # Do NOT use SfM points [S1]
    gaussians = initialize_3dgs_from_points(voxelized_points)
    
    # Learning rate schedule parameters [S1]
    lr_start = 1.6e-4
    lr_end = 1.6e-6
    
    # Minimum noise level for denoising [G]
    t_min = 0
    
    # 2. Training Loop
    for iteration in range(1, num_iterations + 1):
        # Update learning rate (linear decay) [S1]
        current_lr = lr_start - (lr_start - lr_end) * (iteration / num_iterations)
        
        # Sample a training view [S1]
        v_o, I_gt_o, D_gt_o = sample_training_view(training_images, training_poses)
        
        # Render training view [S1]
        I_o, D_tilde_o = render_3dgs(gaussians, v_o)
        
        # Compute Reconstruction Loss [S1]
        loss_recon = compute_l1(I_o, I_gt_o) + \
                     lambda_ssim * compute_ssim(I_o, I_gt_o) + \
                     lambda_depth * compute_l1(D_gt_o, D_tilde_o)
        
        # Initialize total loss
        total_loss = loss_recon
        
        # Pseudo-view regularization (after warm-up) [S1]
        if iteration > 500 and iteration % k == 0:
            # Sample M pseudo views [S1]
            pseudo_views = sample_pseudo_views(v_o, training_poses, M)
            
            # Render pseudo views [S1]
            I_tilde_p, D_tilde_p = render_3dgs_batch(gaussians, pseudo_views)
            
            # Generate guidance images using Diffusion Model [S1]
            I_g_list = []
            for i in range(M):
                # Encode to latent [S1]
                z = vae_encoder(I_tilde_p[i])
                
                # Sample strength s [S1]
                s_max_current = s_max_init - (s_max_init - s_max_final) * (iteration / num_iterations)
                s = random.uniform(s_min, s_max_current)
                
                # Compute noise level t [S1]
                t = s * t_max
                
                # Add noise to latent [S1]
                z_noisy = add_noise(z, t)
                
                # Denoise using diffusion model from t to t_min [S1]
                z_denoised = denoise(z_noisy, t_start=t, t_end=t_min, model=diffusion_model)
                
                # Decode to image [S1]
                I_g = vae_decoder(z_denoised)
                I_g_list.append(I_g)
            
            I_g = stack_tensors(I_g_list)
            
            # Compute Pseudo Loss [S1]
            loss_pseudo = compute_l1(I_g, I_tilde_p) + \
                          lambda_p_lpips * compute_lpips(I_g, I_tilde_p) + \
                          lambda_p_depth * compute_l1(D_p, D_tilde_p)
            
            total_loss = loss_recon + lambda_pseudo * loss_pseudo
        
        # Backpropagate and update Gaussians [S1]
        total_loss.backward()
        for param in gaussians:
            param -= current_lr * param.grad
        for param in gaussians:
            param.grad = None

def voxelize_downsample(points, voxel_size):
    """Downsample points by keeping one point per voxel grid cell."""
    voxelized = {}
    for p in points:
        key = (math.floor(p[0]/voxel_size), math.floor(p[1]/voxel_size), math.floor(p[2]/voxel_size))
        if key not in voxelized:
            voxelized[key] = p
    return list(voxelized.values())

def initialize_3dgs_from_points(points):
    """Initialize Gaussian parameters from point cloud."""
    gaussians = []
    for p in points:
        gaussians.append({'mu': p, 'Sigma': identity_matrix(3), 'alpha': 1.0, 'c': zeros(3)})
    return gaussians

def sample_training_view(images, poses):
    """Randomly select a training view and its ground truth."""
    idx = random.randint(0, len(images) - 1)
    return poses[idx], images[idx], images[idx] # Depth placeholder for brevity

def render_3dgs(gaussians, view):
    """Differentiable rasterization of Gaussians for a view."""
    # Standard 3DGS splatting implementation [G]
    return render_image, render_depth

def sample_pseudo_views(current_view, poses, M):
    """Sample M pseudo views interpolated between current and adjacent views."""
    pseudo_views = []
    for _ in range(M):
        # Interpolate position and rotate yaw by [-delta, delta] [S1]
        pseudo_views.append(current_view)
    return pseudo_views

def render_3dgs_batch(gaussians, views):
    """Render multiple views simultaneously."""
    images = []
    depths = []
    for v in views:
        img, dep = render_3dgs(gaussians, v)
        images.append(img)
        depths.append(dep)
    return stack_tensors(images), stack_tensors(depths)

def compute_l1(a, b):
    return sum(abs(a[i] - b[i]) for i in range(len(a))) / len(a)

def compute_ssim(a, b):
    return 1.0 - structural_similarity(a, b) # Standard SSIM computation [G]

def compute_lpips(a, b):
    return perceptual_loss(a, b) # Standard LPIPS computation [G]

def add_noise(latent, t):
    return latent + random.gauss(0, 1) * t

def denoise(latent, t_start, t_end, model):
    """Iteratively denoise from t_start to t_end using the diffusion model."""
    current = latent
    for t in range(t_start, t_end, -1):
        current = model.predict_noise(current, t)
        current = current - (t - t_end) * current # Simplified Euler step [G]
    return current

def stack_tensors(tensor_list):
    return [sum(t) for t in zip(*tensor_list)]

def identity_matrix(n):
    return [[1 if i==j else 0 for j in range(n)] for i in range(n)]

def zeros(n):
    return [0.0] * n

def structural_similarity(a, b):
    return 1.0 # Placeholder for standard SSIM [G]

def perceptual_loss(a, b):
    return 0.0 # Placeholder for standard LPIPS [G]
```

### Edge Cases and Pitfalls
*   **Initialization Bias:** Using SfM point clouds is explicitly avoided to prevent prior bias from SfM reconstruction errors or assumptions. Only LiDAR data should be used [S1].
*   **Noise Level Control:** The strength parameter $s$ controls the deviation from the 3DGS rendering. A small $s$ preserves 3DGS-rendered noise/details, while a large $s$ produces cleaner images but loses details. The decreasing schedule for $s_{max}$ ensures that early in training (when 3DGS quality is low), the model relies more on the diffusion prior, and later, it relies more on the 3DGS renderings to preserve details [S1].
*   **Pseudo-View Overlap:** Pseudo-views are sampled close to training views, leading to considerable overlap in the corresponding Gaussian models. This allows simultaneous multi-view learning and prevents local minima [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The method adds a diffusion model inference step for every $k$ iterations (every 10 iterations in the cited implementation) for $M$ pseudo views (4 views) [S1]. This significantly increases the per-iteration cost compared to standard 3DGS, which only performs rasterization and loss computation [S1]. The diffusion model inference (encoding, noise addition, denoising, decoding) is computationally expensive [S1].
*   **Memory Complexity:** Requires storing the 3DGS parameters, the diffusion model weights, and the VAE weights [S1]. The latent space operations are memory-efficient compared to pixel-space operations, but the batch processing of pseudo views adds temporary memory overhead [S1].

### Measured Performance
*   Evaluated on KITTI and KITTI-360 datasets [S1].
*   The method is designed for street view synthesis where LiDAR data is available but SfM might be unreliable or biased [S1].
*   The ablation study on $s$ shows that smaller $s$ preserves details, while larger $s$ cleans up noise but loses detail. The schedule $s_{max} \in [0.6, 0.4]$ balances this trade-off [S1].

### Strengths and Limitations
*   **Strengths:**
    *   Improves unseen view synthesis by regularizing with a diffusion prior [S1].
    *   Prevents local minima by leveraging overlapping pseudo-views [S1].
    *   Does not rely on SfM point clouds, making it suitable for scenarios with only LiDAR and monocular images [S1].
*   **Limitations:**
    *   Computationally expensive due to diffusion model inference [S1].
    *   Requires a fine-tuned diffusion model, which adds a pre-training step (ControlNet training) [S1].
    *   The quality of the guidance depends on the fine-tuning of the diffusion model [S1].

### When to Use
*   Use when LiDAR data is available but SfM point clouds are not desired or reliable [S1].
*   Use when high-quality novel view synthesis is required, especially for views not seen during training [S1].
*   Use when the scene has complex textures or structures that benefit from diffusion-based regularization [S1].

### When Not to Use
*   Use standard 3DGS if computational resources are limited and high-quality novel views are not critical [S1].
*   Use if SfM point clouds are available and accurate, as they provide a strong geometric prior without the need for diffusion guidance [S1].

## Background

### Origin and History
*   The method is described in the paper "SGD: Street View Synthesis with Gaussian Splatting and Diffusion Prior" [S1].
*   It builds upon the 3D Gaussian Splatting framework [S1].
*   The diffusion model is fine-tuned using a ControlNet architecture [S1].
*   The method was evaluated on autonomous driving datasets (KITTI, KITTI-360) [S1].

### Development
*   The approach was developed to address the limitations of standard 3DGS in synthesizing unseen views, particularly in autonomous driving scenarios where LiDAR data is available but SfM might be insufficient or biased [S1].
*   The authors ablated the impact of the diffusion prior strength $s$, showing that a decreasing schedule for $s_{max}$ yields better results than a fixed value [S1].

## Sources

[S1] SGD: Street View Synthesis with Gaussian Splatting and Diffusion Prior, https://doi.org/10.1109/wacv61041.2025.00375, fetched 2026-10-27
