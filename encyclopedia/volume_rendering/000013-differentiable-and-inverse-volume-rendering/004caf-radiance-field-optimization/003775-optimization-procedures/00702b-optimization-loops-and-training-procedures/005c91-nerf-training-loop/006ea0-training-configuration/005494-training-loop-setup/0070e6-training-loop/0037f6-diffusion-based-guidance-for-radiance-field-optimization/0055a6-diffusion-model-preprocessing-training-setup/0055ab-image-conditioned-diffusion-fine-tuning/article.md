# Image-Conditioned Diffusion Fine-Tuning

Also called: Training Stage 1

## Overview
Image-Conditioned Diffusion Fine-Tuning is the first stage of a two-step procedure that adapts a pre-trained 2D diffusion model (specifically Stable Diffusion 1.5) to learn high-level semantic content of a specific scene from adjacent reference images, without using pose information [S1]. It sets itself apart from methods like ReconFusion by decoupling semantic identity learning from spatial pose control, thereby preventing the model from overfitting to specific camera viewpoints or occlusion artifacts common in complex driving scenes [S1].

## How it works
This method fine-tunes the U-Net parameters of a diffusion model to condition generation on text prompts and latent features from adjacent reference frames.

**Inputs and Outputs:**
*   **Input:** An input image $I$ (current frame), a previous reference image $I_{pre}$, a next reference image $I_{next}$, and a text prompt $T$.
*   **Output:** A fine-tuned U-Net $\theta$ capable of denoising latents conditioned on $e_T$ and $e_{I_{ref}}$.

**Data Structures:**
*   **Latent Feature Map $z_0$:** Output of the frozen VAE encoder applied to $I$. Dimensions depend on the VAE's downsampling factor (typically $H/8 \times W/8$ for SD 1.5).
*   **Text Embedding $e_T$:** Sequence of token embeddings from the frozen CLIP Text Encoder. Shape: $[L_T, D]$, where $L_T$ is token length and $D=768$ (for clip-vit-B-32).
*   **Image Embedding $e_{I_i}$:** Sequence of patch embeddings from the frozen CLIP Image Encoder. Shape: $[L_I, D]$, where $L_I=50$ (1 CLS token + 49 patch tokens) for $224 \times 224$ input.
*   **Combined Condition $c_{img}$:** Concatenation of $e_T$ and $[e_{I_{pre}}, e_{I_{next}}]$ along the sequence length dimension. Shape: $[L_T + 2L_I, D]$.

**Parameters:**
*   **CLIP Image Encoder Input Size:** $224 \times 224 \times 3$ [S1].
*   **Batch Size:** 4 [S1].
*   **Hardware:** 4 NVIDIA V100 GPUs (32GB VRAM) [S1].
*   **Iterations:** 625,000 [S1].

**Steps:**
1.  **Reference Selection:** For a given input image $I$, select the immediately preceding frame $I_{pre}$ and the immediately following frame $I_{next}$ as the reference set $I_{ref} = \{I_{pre}, I_{next}\}$ [S1].
2.  **Latent Encoding:** Encode $I$ into latent space $z_0$ using the frozen VAE encoder $\mathcal{E}_{VAE}$:
    $$ z_0 = \mathcal{E}_{VAE}(I) $$
3.  **Text Encoding:** Tokenize $T$ and encode it using the frozen CLIP Text Encoder $\mathcal{E}_{text}$:
    $$ e_T = \mathcal{E}_{text}(T) = [e_T^0, \ldots, e_T^{L_T}], \quad e_T^i \in \mathbb{R}^D \quad (1) $$
    where $L_T$ is the token sequence length and $D$ is the embedding dimension [S1].
4.  **Reference Image Encoding:** Encode $I_{pre}$ and $I_{next}$ separately using the frozen CLIP Image Encoder $\mathcal{E}_{Image}$:
    $$ e_{I_i} = \mathcal{E}_{Image}(I_i) = [e_{I_i}^0, \ldots, e_{I_i}^{L_I}], \quad i \in \{\text{pre}, \text{next}\} \quad (2) $$
    where $L_I$ is the number of image patches (50) [S1].
5.  **Condition Fusion:** Concatenate the reference image features to the text embedding along the token length dimension to form the combined condition $c_{img}$:
    $$ c_{img} = \text{Concat}(e_T, e_{I_{pre}}, e_{I_{next}}) $$
6.  **Denoising Prediction:** Pass the noisy latent $z_t$ (at timestep $t$), the timestep $t$, and the combined condition $c_{img}$ into the trainable U-Net $\epsilon_\theta$ to predict the noise $\epsilon$:
    $$ \hat{\epsilon} = \epsilon_\theta(z_t, t, c_{img}) $$
7.  **Loss Calculation:** Optimize $\theta$ by minimizing the mean squared error between the predicted noise and the actual random noise $\epsilon$:
    $$ \mathcal{L}(\theta) = \mathbb{E}_{z_0, \epsilon \sim \mathcal{N}(\mathbf{0}, \mathbf{1}), t, T, I_{ref}} ||\epsilon - \epsilon_\theta(z_t, t, e_T, e_{I_{ref}})||_2^2 \quad (3) $$
    where $\epsilon$ is random noise, $t$ is the denoising timestep, and $z_t$ is the noisy latent at timestep $t$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from diffusers import StableDiffusionPipeline

def train_stage_1(model, vae, text_encoder, image_encoder, dataloader, optimizer, num_iterations=625000, batch_size=4):
    """
    Fine-tunes the U-Net of a Stable Diffusion model using image-conditioned diffusion loss.
    
    Args:
        model: The U-Net component of Stable Diffusion 1.5 (trainable).
        vae: Frozen VAE encoder.
        text_encoder: Frozen CLIP Text Encoder.
        image_encoder: Frozen CLIP Image Encoder (clip-vit-B-32).
        dataloader: Iterator yielding batches of (I, I_pre, I_next, T).
        optimizer: Optimizer for model parameters.
        num_iterations: Total training steps.
        batch_size: Batch size per GPU (total batch size = batch_size * num_gpus).
    """
    model.train()
    vae.eval()
    text_encoder.eval()
    image_encoder.eval()
    
    # Noise scheduler for sampling timesteps
    # Assuming standard DDPM noise schedule used in SD 1.5
    noise_scheduler = model.config.get_noise_scheduler() 
    
    for step in range(num_iterations):
        # 1. Load batch
        # I: [B, 3, H, W], I_pre: [B, 3, H, W], I_next: [B, 3, H, W], T: list of strings
        I, I_pre, I_next, T = next(dataloader)
        
        # 2. Encode Input Image to Latent z0
        with torch.no_grad():
            z0 = vae.encode(I)
            
        # 3. Encode Text Prompt
        with torch.no_grad():
            text_inputs = text_encoder.tokenizer(T, padding=True, return_tensors="pt")
            e_T = text_encoder(text_inputs.input_ids, text_inputs.attention_mask).last_hidden_state
            # e_T shape: [B, L_T, D]
            
        # 4. Encode Reference Images
        with torch.no_grad():
            # Resize references to 224x224 if necessary, as per CLIP input requirements
            I_pre_resized = F.interpolate(I_pre, size=(224, 224), mode='bilinear', align_corners=False)
            I_next_resized = F.interpolate(I_next, size=(224, 224), mode='bilinear', align_corners=False)
            
            e_I_pre = image_encoder(I_pre_resized).last_hidden_state
            e_I_next = image_encoder(I_next_resized).last_hidden_state
            # e_I_pre, e_I_next shape: [B, L_I, D] where L_I=50
            
        # 5. Concatenate Conditions
        # Concatenate along sequence dimension (dim=1)
        # e_T: [B, L_T, D], e_I_pre: [B, L_I, D], e_I_next: [B, L_I, D]
        combined_condition = torch.cat([e_T, e_I_pre, e_I_next], dim=1)
        # combined_condition shape: [B, L_T + 2*L_I, D]
        
        # 6. Sample Timestep and Noise
        t = torch.randint(0, noise_scheduler.config.num_train_timesteps, (batch_size,), device=I.device)
        noise = torch.randn_like(z0)
        
        # 7. Add Noise to Latent
        z_t = noise_scheduler.add_noise(z0, noise, t)
        
        # 8. Predict Noise
        # model expects: sample, timestep, encoder_hidden_states
        predicted_noise = model(z_t, t, encoder_hidden_states=combined_condition).sample
        
        # 9. Compute Loss
        loss = F.mse_loss(predicted_noise, noise)
        
        # 10. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        if step % 1000 == 0:
            print(f"Step {step}, Loss: {loss.item()}")
```

## Subtopics

### Depth ControlNet Integration
A second fine-tuning stage that adds a ControlNet module to the frozen Stage 1 model to incorporate spatial relationships via depth maps derived from LiDAR point clouds [S1]. It addresses the limitation of Stage 1, which lacks awareness of spatial relationships between objects, by enabling depth-controlled generation [S1]. It sets itself apart from Stage 1 by introducing a separate trainable network (ControlNet) initialized as a copy of the U-Net encoder, rather than modifying the main U-Net weights directly [S1].

**Steps:**
1.  **Depth Map Generation:** Derive depth maps $D_{I_{ref}}$ for reference images and $D_I$ for the input image by projecting 3D LiDAR point clouds onto the image plane and completing them using an off-the-shelf depth completion model [S1].
2.  **ControlNet Initialization:** Initialize the ControlNet as a trainable copy of the U-Net’s encoder block from Stage 1 [S1].
3.  **Depth Encoding:** Encode reference depth maps $D_{I_{ref}}$ using a pre-trained depth encoder $\mathcal{E}_{depth}$ (structure identical to CLIP Image Encoder, pre-trained via contrastive learning) [S1]:
    $$ e_{D_i} = \mathcal{E}_{Depth}(I_i) $$
4.  **Feature Fusion:** Fuse encoded depth features $e_{D_i}$ with CLIP image features $e_{I_i}$ via average pooling:
    $$ \tilde{e}_{I_i} = \text{AvgPool}(e_{I_i}, e_{D_i}), \quad i \in \{\text{pre}, \text{next}\} \quad (5) $$
5.  **Injection:** Concatenate the fused features $\tilde{e}_{I_{ref}}$ with text embedding $e_T$ and inject into the ControlNet via cross-attention [S1].
6.  **Training:** Train the ControlNet parameters $\tilde{\theta}$ using the loss function:
    $$ \mathcal{L}(\tilde{\theta}) = \mathbb{E}_{z_0, \epsilon \sim \mathcal{N}(\mathbf{0}, \mathbf{1}), t, T, \{I_{ref}, D_{ref}\}} ||\epsilon - \epsilon_\theta(z_t, t, c_D, e_T, \tilde{e}_{I_{ref}})||_2^2 \quad (6) $$
    where $c_D$ represents the depth condition [S1].

**Parameters:**
*   **Training Iterations:** 125,000 steps [S1].

## Operating characteristics
*   **Time Complexity:** The method requires significant computational resources due to the large number of iterations (625,000 for Stage 1, 125,000 for Stage 2) and the use of multiple encoders (VAE, CLIP Text, CLIP Image) per step [S1].
*   **Memory Complexity:** Requires 4 NVIDIA V100 GPUs (32GB VRAM each) with a batch size of 4 per GPU, indicating high memory usage for storing latent representations and model weights [S1].
*   **Strengths:** Effectively learns high-level semantic content and "what exists" in a scene without being biased by camera pose, which is crucial for complex driving scenes with occlusions [S1].
*   **Limitations:** Stage 1 alone does not capture spatial relationships or depth, requiring the subsequent ControlNet stage for geometric accuracy [S1]. The reliance on LiDAR for Stage 2 limits applicability to scenes with available depth data [S1].
*   **When to use:** When synthesizing novel views for a specific scene where semantic consistency across adjacent frames is more critical than initial geometric precision, and LiDAR data is available for the second stage.
*   **When not to use:** When pose information is the primary constraint or when computational resources are limited to a single GPU.

## Background
The method originates from the paper "SGD: Street View Synthesis with Gaussian Splatting and Diffusion Prior" [S1]. It was developed to address the challenges of autonomous driving scene synthesis, where complex objects (buildings, vehicles, pedestrians) and mutual occlusions make it difficult for diffusion models to understand 3D structure from pose alone [S1]. The approach builds on Stable Diffusion 1.5 [S1] and utilizes CLIP encoders  for conditioning. It is part of a pipeline that eventually regularizes 3D Gaussian Splatting training [S1].

## Sources
[S1] SGD: Street View Synthesis with Gaussian Splatting and Diffusion Prior, https://doi.org/10.1109/wacv61041.2025.00375, fetched 2026-10-27
