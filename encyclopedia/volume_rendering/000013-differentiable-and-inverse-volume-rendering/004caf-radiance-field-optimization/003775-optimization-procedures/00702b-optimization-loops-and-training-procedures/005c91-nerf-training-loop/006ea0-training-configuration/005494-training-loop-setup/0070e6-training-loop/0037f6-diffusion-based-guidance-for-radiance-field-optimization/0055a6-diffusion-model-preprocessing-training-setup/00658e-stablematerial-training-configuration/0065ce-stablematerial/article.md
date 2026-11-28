# StableMaterial

Also called: 2D Material Denoising Diffusion Prior

## Overview
StableMaterial is a 2D diffusion model prior designed to refine multi-view data to estimate the most likely albedo and material properties from given input appearances [S1]. It addresses the challenge of recovering physically plausible material factors (albedo and ORM) from images under unknown illumination by leveraging a fine-tuned Stable Diffusion architecture [S1]. Unlike generic denoising models, it is specifically conditioned on input RGB images to output factorized albedo ($\hat{I}_d$) and ORM ($\hat{I}_{orm}$) 2D maps, serving as a strong prior for inverse rendering pipelines such as MaterialFusion [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An RGB image $x$ of an object under unknown illumination.
*   **Output:** Two 2D maps: the albedo map $\hat{I}_d$ and the ORM (Occlusion, Roughness, Metallic) map $\hat{I}_{orm}$ [S1].

**Data Structures and Layout:**
The model operates in the latent space of the pretrained Stable Diffusion VAE.
*   **Latent Channels:** The noisy latent codes $z_t$ consist of 8 channels: 4 channels for albedo ($z_{td}$) and 4 channels for ORM ($z_{torm}$) [S1].
*   **UNet Input:** The UNet input tensor has 12 channels in total:
    1.  4 channels: Clean latent encoding of the input image $x$, denoted $z_{0x} = E(x)$, where $E$ is the frozen pretrained VAE encoder [S1].
    2.  4 channels: Noisy albedo latent $z_{td}$ [S1].
    3.  4 channels: Noisy ORM latent $z_{torm}$ [S1]
*   **Conditioning:** The text conditioning of the original Stable Diffusion is replaced with a CLIP image embedding of the input image $x$ [S1].

**Training Data:**
The model is trained on triplets $(x, I_d, I_{orm})$ derived from a curated dataset of approximately 29K high-quality objects [S1]. This includes:
*   ~12K artist-designed synthetic Blender objects (BlenderVault) [S1].
*   ~8.5K filtered Objaverse objects [S1].
*   ~8K ABO objects [S1].
Total training dataset size is approximately 28K–29K synthetic objects with high-quality PBR assets [S1].

**Algorithm Steps:**
1.  **Initialization:** Load pretrained Stable Diffusion 2.1 weights. Modify the UNet to accept 12 input channels and 8 output channels. Randomly initialize the first and last layers of the UNet to account for the different channel dimensions; keep all other UNet parameters unchanged [S1].
2.  **Encoding:** For a training triplet $(x, I_d, I_{orm})$, encode the input image $x$ using the frozen VAE encoder $E$ to obtain $z_{0x}$ [S1]. Encode the ground truth albedo $I_d$ and ORM $I_{orm}$ using $E$ to obtain their clean latents, then concatenate them in the channel dimension to form the target latent $z$ [S1].
3.  **Noise Addition:** Sample a diffusion timestep $t$ and an 8-channel random noise vector $\epsilon \sim \mathcal{N}(0, I)$ [S1]. Compute the noisy latent $z_t$ by adding noise to $z$ according to the diffusion scheduler parameters $\alpha_t$ and $\sigma_t$ [S1].
4.  **Conditioning:** Compute the CLIP image embedding of $x$. Construct the UNet input by concatenating $z_{0x}$, $z_t$, and the timestep embedding $t$ [S1].
5.  **Prediction:** Pass the concatenated input through the UNet $\epsilon_\theta$ to predict the noise component $v_t$ [S1].
6.  **Loss Calculation:** Compute the diffusion loss $L_{diff}$ using the v-prediction objective [S1].
7.  **Optimization:** Update the UNet parameters $\theta$ to minimize $L_{diff}$ [S1].

**Formulas:**
The diffusion loss is defined as [S1]:
$$
L_{diff} = \mathbb{E}_{x, k_d, k_{orm}, \epsilon \sim \mathcal{N}(0,I), t} \|\epsilon_\theta(E(x), z_t, t) - v_t\|_2^2
$$
where $v_t$ is the v-prediction target [S1]:
$$
v_t = \alpha_t \epsilon - \sigma_t z
$$
Here, $\alpha_t$ and $\sigma_t$ are parameters from the diffusion scheduler [S1].

**Parameters:**
*   **Classifier-Free Guidance (CFG):** Input images, input image prompts (CLIP embeddings), or both are set to all zeros with a 5% probability each [S1].
*   **Guidance Scale:** Set to 3.0 [S1].
*   **Base Model:** Stable Diffusion 2.1 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
from diffusers import StableDiffusionPipeline

def train_stable_material(unet, vae, clip_encoder, optimizer, dataloader, num_epochs):
    """
    Trains the StableMaterial model.
    unet: Modified UNet with 12 input channels, 8 output channels.
    vae: Pretrained Stable Diffusion VAE (encoder E and decoder D).
    clip_encoder: Pretrained CLIP image encoder.
    optimizer: Optimizer for unet parameters.
    dataloader: Yields batches of (x, I_d, I_orm).
    """
    unet.train()
    vae.eval() # Frozen
    clip_encoder.eval() # Frozen
    
    for epoch in range(num_epochs):
        for x, I_d, I_orm in dataloader:
            # 1. Encode inputs
            # x: RGB image [B, 3, H, W]
            # I_d, I_orm: Albedo and ORM maps [B, 3, H, W]
            
            with torch.no_grad():
                z_0x = vae.encode(x).latent_dist.sample() # Clean latent of input image
                z_d = vae.encode(I_d).latent_dist.sample() # Clean latent of albedo
                z_orm = vae.encode(I_orm).latent_dist.sample() # Clean latent of ORM
            
            # Concatenate target latents: 4 channels albedo + 4 channels ORM
            z = torch.cat([z_d, z_orm], dim=1) # [B, 8, H_lat, W_lat]
            
            # 2. Sample noise and timestep
            t = torch.randint(0, unet.config.num_train_timesteps, (x.shape[0],), device=x.device).long()
            noise = torch.randn_like(z) # [B, 8, H_lat, W_lat]
            
            # 3. Add noise to get z_t
            # Note: Actual scheduler logic for alpha_t, sigma_t is handled by diffusers library
            # or manually computed if implementing from scratch.
            # Here we assume access to scheduler parameters alpha_t, sigma_t
            alpha_t, sigma_t = get_scheduler_params(t) # Implementation detail of scheduler
            
            # z_t = alpha_t * z + sigma_t * noise (standard forward diffusion)
            # However, the loss uses v_t = alpha_t * noise - sigma_t * z
            # We need z_t for the UNet input.
            # Using standard DDPM/DDIM notation where z_t = sqrt(alpha_bar_t) * z + sqrt(1-alpha_bar_t) * epsilon
            # The paper defines v_t = alpha_t * epsilon - sigma_t * z.
            # We compute z_t based on the specific scheduler used in Stable Diffusion.
            z_t = vae.scheduler.add_noise(z, noise, t)
            
            # 4. Prepare UNet input
            # Concatenate z_0x (4 ch) and z_t (8 ch) -> 12 ch input
            unet_input = torch.cat([z_0x, z_t], dim=1) # [B, 12, H_lat, W_lat]
            
            # 5. Classifier-Free Guidance Logic
            # Randomly zero out conditions with 5% probability each
            do_cond_img = torch.rand(1).item() > 0.05
            do_cond_clip = torch.rand(1).item() > 0.05
            
            if not do_cond_img:
                z_0x = torch.zeros_like(z_0x)
            
            # Compute CLIP embedding
            clip_emb = clip_encoder(x).image_embeds # [B, D]
            if not do_cond_clip:
                clip_emb = torch.zeros_like(clip_emb)
                
            # 6. Forward pass
            # unet predicts v_t
            pred_v = unet(unet_input, t, encoder_hidden_states=clip_emb).sample
            
            # 7. Compute Loss
            # v_t target = alpha_t * noise - sigma_t * z
            target_v = alpha_t * noise - sigma_t * z
            
            loss = nn.functional.mse_loss(pred_v, target_v)
            
            # 8. Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```

## Operating characteristics
*   **Complexity:** The model inherits the computational complexity of Stable Diffusion 2.1 UNet inference and training, scaled by the batch size and resolution of the latent space [S1].
*   **Performance:** The model demonstrates superior performance in recovering albedo and materials from input images compared to prior state-of-the-art methods, particularly in novel view synthesis under relighting [S1].
*   **Strengths:** Provides a strong 2D prior for material properties, improving relighting performance in inverse rendering pipelines [S1]. It generalizes better to novel lighting conditions across diverse scenarios [S1].
*   **Limitations:** It is a 2D prior; it does not directly output 3D geometry or handle multi-view consistency without integration into a 3D optimization loop (e.g., via SDS) [S1].
*   **When to use:** Use when a strong prior on albedo and material properties is needed for inverse rendering or relighting tasks, especially when input images are under unknown illumination [S1].

## Background
StableMaterial was introduced in the paper "MaterialFusion: Enhancing Inverse Rendering with Material Diffusion Priors" [S1]. It builds upon Stable Diffusion 2.1 [S1] and is trained on a dataset comprising BlenderVault, filtered Objaverse, and ABO objects [S1]. The method is part of the MaterialFusion pipeline, which combines this 2D diffusion prior with score distillation sampling (SDS) for 3D inverse rendering [S1].

## Sources
[S1] MaterialFusion: Enhancing Inverse Rendering with Material Diffusion Priors, https://doi.org/10.1109/3dv66043.2025.00079, fetched 2026-10-29
