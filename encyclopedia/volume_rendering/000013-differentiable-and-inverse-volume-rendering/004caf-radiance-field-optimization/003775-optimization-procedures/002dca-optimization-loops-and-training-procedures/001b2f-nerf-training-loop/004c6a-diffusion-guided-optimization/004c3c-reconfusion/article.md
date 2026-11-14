# ReconFusion

Also called: ReconFusion: 3D Reconstruction with Diffusion Priors

## Overview
ReconFusion is a 3D reconstruction method that recovers high-quality scenes from a limited number of input images (few-view setting) by regularizing a NeRF-based pipeline with a diffusion prior trained for novel view synthesis [S1]. Unlike methods that rely solely on geometric constraints or text-guided score distillation, ReconFusion explicitly trains a diffusion model to predict plausible novel views conditioned on input images and poses, then uses this model to supervise the NeRF optimization at unobserved viewpoints [S1]. It sets itself apart from siblings like DreamFusion by using a multiview-conditioned diffusion prior rather than a text-conditioned one, and from SparseFusion by sampling a new image at each iteration rather than using multistep sampling from a noisy encoding of the current rendering [S1].

## How it works
ReconFusion consists of two main components: a diffusion model for novel view synthesis and a NeRF optimization loop. The method optimizes a NeRF representation $\psi$ (using Zip-NeRF as the backbone [S1]) to minimize two losses: a reconstruction loss on observed views and a sample loss on novel views predicted by the diffusion model.

### Inputs and Outputs
**Inputs:**
- A set of $N$ input images $x_{obs} = \{x_i\}_{i=1}^N$ with corresponding camera poses $\pi_{obs} = \{\pi_i\}_{i=1}^N$ [S1].
- A pretrained Latent Diffusion Model (LDM) [42] encoder $E$, denoising U-Net $\epsilon_\theta$, and decoder $D$ [S1].
- A pretrained CLIP image encoder [S1].
- A PixelNeRF-style encoder $R_\phi$ [S1].

**Outputs:**
- Optimized NeRF parameters $\psi$ [S1].
- Rendered images from novel viewpoints [S1].

### Data Structures
1.  **NeRF ($\psi$):** Standard Zip-NeRF representation [S1].
2.  **PixelNeRF Encoder ($R_\phi$):** A small U-Net that takes input images of resolution $512 \times 512$ and outputs a feature map $f$ of resolution $64 \times 64$ with 128 channels [S1].
3.  **Diffusion Model:**
    -   **VAE:** Encodes images to latents of dimension $64 \times 64 \times 8$ from $512 \times 512 \times 3$ input images [S1].
    -   **U-Net ($\epsilon_\theta$):** Takes noisy latents, feature maps, and CLIP embeddings as input [S1].
    -   **CLIP Embeddings:** Text/image embeddings used for cross-attention in the U-Net [S1].

### Numbered Steps

#### 1. Diffusion Model Training (Pre-requisite)
Before reconstruction, the diffusion model is finetuned for novel view synthesis [S1].
1.  **Conditioning:** For a target novel view $\pi$, select the 3 nearest camera positions from the observed inputs $\pi_{obs}$ [S1].
2.  **Feature Rendering:** Use the PixelNeRF encoder $R_\phi$ to render a feature map $f$ for the target view $\pi$ using the input images and poses:
    $$ f = R_\phi(x_{obs}, \pi_{obs}, \pi) $$
    [S1]
3.  **Latent Encoding:** Encode the input images and the target view image (from dataset) into latents using the VAE encoder $E$ [S1].
4.  **Diffusion Process:** Add noise to the target latent $z$ to create $z_t$ at timestep $t$ [S1].
5.  **Cross-Attention:** Compute CLIP embeddings of the input images and inject them into the U-Net via cross-attention [S1].
6.  **Loss:** Train the U-Net $\epsilon_\theta$ to predict the noise or clean latent. The model is finetuned on a mixture of RealEstate10K, CO3D, MVImgNet, and Objaverse datasets [S1].
7.  **Classifier-Free Guidance:** During training, set input images to all zeros with probability 10% to enable classifier-free guidance [S1].

#### 2. NeRF Optimization Loop
For each training iteration of the NeRF:
1.  **Sample Observed View:** Randomly select an observed pose $\pi_{obs}$ and image $x_{obs}$.
2.  **Render Observed View:** Render image $x_{render}$ from the current NeRF $\psi$ at $\pi_{obs}$ [S1].
3.  **Compute Reconstruction Loss:** Calculate the Charbonnier loss between $x_{render}$ and $x_{obs}$:
    $$ L_{recon}(\psi) = \ell(x_{render}, x_{obs}) $$
    [S1]
4.  **Sample Novel View Pose:** Select a novel pose $\pi$ from a distribution based on the input poses (e.g., nearest 3 inputs) [S1].
5.  **Render Novel View Feature Map:** Use the PixelNeRF encoder $R_\phi$ to generate the feature map $f$ for $\pi$ using the input images [S1].
6.  **Render Novel View Image:** Render image $x$ from the current NeRF $\psi$ at $\pi$ [S1].
7.  **Encode to Latent:** Encode $x$ into latent $z$ using VAE encoder $E$ [S1].
8.  **Add Noise:** Sample a timestep $t$ and noise $\epsilon$. Create noisy latent $z_t = \alpha_t z + \sigma_t \epsilon$. The sources specify $t_{max} = 1.0$ and $t_{min}$ is linearly annealed from 1.0 to 0.0 during training [S1].
9.  **Diffusion Sampling:**
    -   Input $z_t$, feature map $f$, and CLIP embeddings to the U-Net $\epsilon_\theta$ [S1].
    -   Perform $k=10$ DDIM sampling steps to generate a denoised latent $\hat{z}$ [S1].
    -   Decode $\hat{z}$ to image $\hat{x}$ using VAE decoder $D$ [S1].
    -   Apply classifier-free guidance with scale 3.0 [S1].
10. **Compute Sample Loss:** Calculate the loss between the rendered image $x$ and the diffusion-generated sample $\hat{x}$:
    $$ L_{sample}(\psi) = \| x - \hat{x} \|_1 + L_{p}(x, \hat{x}) $$
    where $L_p$ is the perceptual distance (LPIPS) [S1].
11. **Total Loss:** Combine losses with a weighting factor $w(t)$ that decays linearly from 1 to 0.1 over training:
    $$ L_{total} = L_{recon} + w(t) L_{sample} $$
    [S1]
12. **Update:** Update NeRF parameters $\psi$ using gradient descent [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# Assume these are initialized and loaded from checkpoints
# nerf: Zip-NeRF model with parameters psi
# pixelnerf_encoder: R_phi, outputs feature maps
# vae_encoder: E, encodes images to latents
# vae_decoder: D, decodes latents to images
# diffusion_unet: epsilon_theta, denoising U-Net
# clip_encoder: CLIP image encoder
# optimizer: Optimizer for psi

def get_nearest_poses(target_pose, observed_poses, k=3):
    """Select k nearest camera positions from observed inputs."""
    # Compute distances between target_pose and observed_poses
    # Return indices of k nearest
    pass

def compute_clip_embeddings(images):
    """Compute CLIP embeddings for input images."""
    return clip_encoder(images)

def render_nerf_image(pose, psi):
    """Render an image from NeRF at given pose."""
    # Standard volume rendering
    pass

def render_pixelnerf_feature_map(input_images, observed_poses, target_pose):
    """Render feature map using PixelNeRF encoder."""
    return pixelnerf_encoder(input_images, observed_poses, target_pose)

def add_noise(latent, t):
    """Add Gaussian noise to latent at timestep t."""
    # alpha_t, sigma_t derived from t
    noise = torch.randn_like(latent)
    return alpha_t * latent + sigma_t * noise

def diffusion_sample(noisy_latent, feature_map, clip_emb, t, guidance_scale=3.0, steps=10):
    """Perform DDIM sampling to denoise latent."""
    current_latent = noisy_latent
    for i in range(steps):
        # Predict noise using U-Net
        # Apply classifier-free guidance if needed
        # Update current_latent using DDIM step
        pass
    return current_latent

def perceptual_loss(img1, img2):
    """Compute LPIPS perceptual distance."""
    # Use pretrained LPIPS network
    pass

def train_reconfusion(input_images, observed_poses, num_iterations=1000):
    """
    Train NeRF using ReconFusion.
    input_images: List of input images (N images)
    observed_poses: List of observed poses (N poses)
    """
    # Precompute CLIP embeddings for input images
    clip_emb = compute_clip_embeddings(input_images)
    
    # Annealing schedule for t_min
    t_min_start = 1.0
    t_min_end = 0.0
    
    for step in range(num_iterations):
        # 1. Sample observed view
        idx = random.randint(0, len(input_images) - 1)
        obs_img = input_images[idx]
        obs_pose = observed_poses[idx]
        
        # 2. Render observed view
        render_img_obs = render_nerf_image(obs_pose, neRF.parameters())
        
        # 3. Compute Reconstruction Loss (Charbonnier)
        recon_loss = charbonnier_loss(render_img_obs, obs_img)
        
        # 4. Sample novel view pose
        novel_pose = sample_novel_pose(observed_poses)
        
        # 5. Render feature map for novel view
        feat_map = render_pixelnerf_feature_map(input_images, observed_poses, novel_pose)
        
        # 6. Render novel view image
        render_img_novel = render_nerf_image(novel_pose, neRF.parameters())
        
        # 7. Encode to latent
        latent_novel = vae_encoder(render_img_novel)
        
        # 8. Add noise
        # t_max is fixed at 1.0
        # t_min is annealed
        t_min = t_min_start - (t_min_start - t_min_end) * (step / num_iterations)
        t = random.uniform(t_min, 1.0)
        noisy_latent = add_noise(latent_novel, t)
        
        # 9. Diffusion Sampling
        # Use classifier-free guidance scale 3.0
        # k=10 steps
        denoised_latent = diffusion_sample(
            noisy_latent, 
            feat_map, 
            clip_emb, 
            t, 
            guidance_scale=3.0, 
            steps=10
        )
        
        # Decode to image
        sample_img = vae_decoder(denoised_latent)
        
        # 10. Compute Sample Loss
        l1_loss = torch.mean(torch.abs(render_img_novel - sample_img))
        lpips_loss = perceptual_loss(render_img_novel, sample_img)
        sample_loss = l1_loss + lpips_loss
        
        # 11. Weighting decay
        # Linear decay from 1 to 0.1
        w_t = 1.0 - 0.9 * (step / num_iterations)
        
        # 12. Total Loss
        total_loss = recon_loss + w_t * sample_loss
        
        # 13. Update
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
```

### Parameters
-   **Diffusion Model:** Latent Diffusion Model [42] with $64 \times 64 \times 8$ latent space, $512 \times 512 \times 3$ input resolution [S1].
-   **PixelNeRF Encoder:** Outputs $64 \times 64$ feature map with 128 channels [S1].
-   **Classifier-Free Guidance Scale:** 3.0 [S1].
-   **Sampling Steps ($k$):** 10 DDIM steps [S1].
-   **Timesteps:** $t_{max} = 1.0$ fixed; $t_{min}$ annealed from 1.0 to 0.0 [S1].
-   **Loss Weighting ($w(t)$):** Linearly decayed from 1 to 0.1 over training [S1].
-   **Batch Size:** 256 (for diffusion model training) [S1].
-   **Learning Rate:** $10^{-4}$ (for joint training of PixelNeRF and U-Net) [S1].
-   **NeRF Training Iterations:** 1000 [S1].
-   **Nearest Neighbors:** 3 nearest camera positions used for conditioning [S1].
-   **Reconstruction Loss:** Charbonnier loss [S1].
-   **Sample Loss:** $L_1$ + LPIPS [S1].

### Edge Cases and Pitfalls
-   **Few-View Scenarios:** With 3 or 9 views, the reconstruction problem is severely underconstrained. The diffusion prior prevents catastrophic failures and improves robustness [S1].
-   **Dense Capture:** With 54 views, the method helps reduce "fog" and "floater" artifacts common in NeRF while preserving well-sampled regions [S1].
-   **Novel View Selection:** Views should not be placed inside objects or behind walls. The method uses a distribution based on known input poses to select reasonable novel views [S1].

## Operating characteristics
-   **Time Complexity:** The method requires training a diffusion model (250k iterations) and then optimizing the NeRF (1000 iterations). Each NeRF iteration involves rendering, diffusion sampling (10 steps), and loss computation [S1].
-   **Memory Complexity:** Depends on the NeRF size and the diffusion model (U-Net, VAE, CLIP). The latent space is $64 \times 64 \times 8$ [S1].
-   **Performance:** Outperforms existing baselines on forward-facing and 360-degree scenes with few input views (3, 6, 9 views) [S1].
-   **Strengths:** Effective regularizer for few-view NeRFs; reduces artifacts in dense captures; supports any number of input images via PixelNeRF conditioning [S1].
-   **Limitations:** Requires training a diffusion model on multiview data; relies on the quality of the diffusion prior [S1].
-   **When to Use:** When only a few photos are available for 3D reconstruction [S1].
-   **When Not to Use:** When dense capture is available and artifacts are minimal, the overhead may not be justified [S1].

## Background
ReconFusion was introduced in 2023/2024 by researchers aiming to reduce the need for dense capture in 3D reconstruction [S1]. It builds on the idea of using diffusion models as priors for 3D generation, similar to DreamFusion [36] and SparseFusion [67], but specifically conditions the diffusion model on input images and poses for novel view synthesis [S1]. The method uses Zip-NeRF as the backbone [S1].

## Sources
[S1] ReconFusion: 3D Reconstruction with Diffusion Priors, https://doi.org/10.1109/cvpr52733.2024.02036, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
