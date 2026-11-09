# Diffusion-Based Guidance for Radiance Field Optimization

Also called: SDS, Score Distillation Sampling, Stable Diffusion Training Guidance

## Overview
Diffusion-Based Guidance for Radiance Field Optimization, commonly known as Score Distillation Sampling (SDS), computes a gradient signal from a frozen text-to-image diffusion model (typically Stable Diffusion) to optimize a radiance field or 3D Gaussian representation [S1, unofficial]. It solves the problem of generating 3D content from 2D priors by backpropagating the difference between the diffusion model's predicted noise and the actual noise added to the rendered image's latent representation [S1, unofficial]. It differs from its siblings by relying on a single-view 2D diffusion prior with classifier-free guidance, whereas [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md) uses multi-view consistency and [VCCNF Training Loop](4736-vccnf-training-loop.md) uses a different conditional flow matching approach.

## How it works
**Inputs and Outputs:**
*   **Input:** Rendered RGB image (`pred_rgb`) from the radiance field; text prompt embeddings (`encoder_hidden_states`); optional azimuth angles (`hors`) for directional conditioning; timestep `t`.
*   **Output:** Gradient signal (`grad`) and loss value (`loss`) used to update the radiance field parameters.

**Data Structures:**
*   **Latents:** Tensor of shape `[B, 4, 64, 64]` (batch size, channels, height, width) representing the encoded image in the Stable Diffusion latent space [S1, unofficial].
*   **Noisy Latents:** Tensor of same shape as `latents`, with noise added according to the diffusion scheduler [S1, unofficial].
*   **Embeddings:** Concatenated tensor of positive (conditional) and negative (unconditional) text embeddings [S1, unofficial].

**Parameters:**
*   `guidance_scale`: Classifier-free guidance scale multiplier. Default value in code: 100 [S1, unofficial].
*   `t_range`: List `[0.02, 0.98]` used to compute `min_step` and `max_step` as fractions of `num_train_timesteps` [S1, unofficial].
*   `step_ratio`: Float controlling timestep selection. `1.0` maps to `min_step` (early diffusion), `0.0` maps to `max_step` (late diffusion) [S1, unofficial].
*   `as_latent`: Boolean flag. If `false`, input is interpolated to 512x512 and encoded via VAE. If `true`, input is interpolated to 64x64 and scaled by `2 * x - 1` [S1, unofficial].

**Steps:**
1.  **Preprocessing:** Convert `pred_rgb` to `self.dtype`. If `as_latent` is `false`, interpolate to 512x512 and encode via VAE to get `latents` of shape `[B, 4, 64, 64]`. If `as_latent` is `true`, interpolate to 64x64 and scale by `2 * x - 1` [S1, unofficial].
2.  **Weighting:** Compute weighting factor $w = (1 - \alpha_t)$, where $\alpha_t$ is the noise schedule value at timestep $t$. Reshape to `[batch_size, 1, 1, 1]` [S1, unofficial].
3.  **Noise Addition:** Sample noise $\epsilon \sim \mathcal{N}(0, I)$ with shape matching `latents`. Compute noisy latents: $z_t = \text{scheduler.add\_noise}(z_0, \epsilon, t)$ [S1, unofficial].
4.  **CFG Preparation:** Duplicate noisy latents and timesteps for classifier-free guidance: $z_{t, \text{input}} = [z_t; z_t]$ and $t_{\text{input}} = [t; t]$ [S1, unofficial].
5.  **Embedding Preparation:** Concatenate positive and negative text embeddings. If azimuth angles (`hors`) are provided, select directional embeddings: `front` if $|h| < 60^\circ$, `side` if $60^\circ \le |h| < 120^\circ$, `back` otherwise. Concatenate these with the negative embeddings [S1, unofficial].
6.  **UNet Prediction:** Pass inputs to the frozen UNet: $\hat{\epsilon} = \text{UNet}(z_{t, \text{input}}, t_{\text{input}}, \text{encoder\_hidden\_states}=\text{embeddings}).\text{sample}$ [S1, unofficial].
7.  **Classifier-Free Guidance:** Split $\hat{\epsilon}$ into conditional ($\hat{\epsilon}_c$) and unconditional ($\hat{\epsilon}_u$) predictions. Compute guided prediction: $\hat{\epsilon}_{\text{guided}} = \hat{\epsilon}_u + \text{guidance\_scale} \times (\hat{\epsilon}_c - \hat{\epsilon}_u)$ [S1, unofficial].
8.  **Gradient Computation:** Compute the raw gradient: $g = w \times (\hat{\epsilon}_{\text{guided}} - \epsilon)$. Clamp NaN values to 0: $g = \text{torch.nan\_to\_num}(g)$ [S1, unofficial].
9.  **Target and Loss:** Compute target: $z_{\text{target}} = (z_0 - g).\text{detach}()$. Compute loss: $\mathcal{L} = \frac{0.5}{B} \sum \| z_0 - z_{\text{target}} \|^2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sds_loss(pred_rgb, text_embeddings, hors, t, guidance_scale, as_latent, vae, unet, scheduler, dtype):
    # 1. Preprocessing
    if not as_latent:
        # Interpolate to 512x512 and encode via VAE
        img_512 = interpolate(pred_rgb, size=(512, 512))
        latents = vae.encode(img_512).latent_dist.sample()
    else:
        # Interpolate to 64x64 and scale
        img_64 = interpolate(pred_rgb, size=(64, 64))
        latents = 2 * img_64 - 1

    # 2. Weighting
    alpha_t = scheduler.alphas[t]
    w = (1 - alpha_t).view(latents.shape[0], 1, 1, 1)

    # 3. Noise Addition
    noise = torch.randn_like(latents)
    latents_noisy = scheduler.add_noise(latents, noise, t)

    # 4. CFG Preparation
    latent_model_input = torch.cat([latents_noisy] * 2)
    tt = torch.cat([t] * 2)

    # 5. Embedding Preparation
    # Assume pos_emb and neg_emb are extracted from text_embeddings
    pos_emb, neg_emb = text_embeddings[0], text_embeddings[1]
    
    # Handle directional embeddings if hors is provided
    if hors is not None:
        # Select directional embeddings based on azimuth angle h
        # front: |h| < 60, side: 60 <= |h| < 120, back: otherwise
        # This logic assumes a pre-defined mapping or tensor of directional embeddings
        dir_embs = select_directional_embeddings(hors) 
        # Concatenate with negative embeddings for the unconditional branch
        # Note: The source implies concatenating with negative embeddings, 
        # but standard CFG usually keeps neg_emb separate. 
        # Based on [S1, unofficial]: "concatenate with negative embeddings"
        # We assume neg_emb is expanded or concatenated appropriately for the UNet input
        # For simplicity, we assume the UNet handles the concatenation or the 
        # 'embeddings' variable passed below is the final concatenated tensor.
        # Re-reading [S1, unofficial]: "Concatenate positive and negative embeddings... 
        # select directional embeddings... and concatenate with negative embeddings."
        # This suggests the unconditional input might be modified or the structure is specific.
        # Standard CFG: [pos_emb; neg_emb]. 
        # Here: [pos_emb; (neg_emb + dir_emb)]? Or [pos_emb; neg_emb] and dir_emb is used differently?
        # The source says: "concatenate with negative embeddings". 
        # Let's assume the unconditional embedding is modified by direction.
        uncond_emb = concatenate(neg_emb, dir_embs)
        embeddings = concatenate(pos_emb, uncond_emb)
    else:
        embeddings = concatenate(pos_emb, neg_emb)

    # 6. UNet Prediction
    noise_pred = unet(latent_model_input, tt, encoder_hidden_states=embeddings).sample

    # 7. Classifier-Free Guidance
    noise_pred_cond, noise_pred_uncond = noise_pred.chunk(2)
    noise_pred = noise_pred_uncond + guidance_scale * (noise_pred_cond - noise_pred_uncond)

    # 8. Gradient Computation
    grad = w * (noise_pred - noise)
    grad = torch.nan_to_num(grad)

    # 9. Target and Loss
    target = (latents - grad).detach()
    loss = 0.5 * F.mse_loss(latents.float(), target, reduction='sum') / latents.shape[0]
    
    return loss, grad
```

**Edge Cases and Pitfalls:**
*   **NaN Values:** Gradient computation can produce NaN values due to numerical instability in the diffusion process or guidance scaling. The code explicitly clamps NaNs to 0 using `torch.nan_to_num(grad)` [S1, unofficial].
*   **Timestep Selection:** The choice of `t` (via `step_ratio`) affects the balance between structure and detail. Early timesteps (`step_ratio` near 1.0) provide structural guidance, while late timesteps (`step_ratio` near 0.0) provide detail [S1, unofficial].

## Subtopics

### Iterative Optimization with Learnable Parameters for Diffusion-Guided Radiance Fields
A method addressing quality impairment caused by differences between the diffusion prior and the training process of the diffusion model [S2]. It performs iterative optimization of the 3D model and the diffusion prior [S2]. Specifically, it introduces a learnable unconditional embedding in the classifier-free guidance formula and adds additional parameters, such as LoRA parameters, to the network [S2].

### LucidDreamer
A scene generation method based on a reference image that extends outward [S2]. It utilizes Stable Diffusion for image inpainting to generate unseen regions on sampled views [S2]. Monocular depth estimation and alignment are incorporated to establish a 3D point cloud from these views, which is then used as the initial value for 3D Gaussian Splatting optimization [S2].

### Interval Score Matching
A modification to SDS loss that replaces DDPM with DDIM inversion and introduces supervisions from interval steps to avoid high error in one-step reconstruction [S2].

### Context-Translation Training with Segmentation Masking
A technique where a segmentation mask is passed to the training process to exclude certain parts of the object from stylization [S3]. The loss is calculated only on pixels outside the mask to avoid changing the appearance of masked pixels [S3]. An additional background loss $L_{BG}$ is employed as regularization to prevent unwanted style artifacts in the background [S3]. $L_{BG}$ is defined as the L1 distance between the ground truth pixels of the background in the rendered image and the pixels in the background of the GT image [S3].

### DDPM Denoise Function Network
An 8-layer 3D convolution network used as the denoise function in some diffusion-based 3D generation methods [S4, unofficial]. Layer widths are {32, 64, 128, 256, 256, 128, 64, 32} [S4, unofficial]. The first four layers encode voxel resolution from 32 to 4 using stride 2, and the last four layers are 3D convolution transpose layers recovering resolution from 4 to 32 with stride 2 [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** High, due to the need to run the diffusion model (UNet) and VAE encoder/decoder for every optimization step. The UNet inference is the dominant cost [S1, unofficial].
*   **Memory Complexity:** Moderate to High. Requires storing the radiance field parameters, the diffusion model weights (frozen), and intermediate latent tensors. The VAE encoding/decoding adds memory overhead [S1, unofficial].
*   **Measured Performance:** Typically generates high-quality 3D assets from text prompts, but suffers from issues like the "Janus problem" (multiple faces) and floaters without additional constraints [S2].
*   **Scaling:** Scales with the complexity of the diffusion model and the resolution of the rendered images. Larger guidance scales improve coherence but may reduce diversity [S1, unofficial].
*   **Strengths:** Leverages powerful 2D diffusion priors; relatively simple to implement with existing diffusion libraries [S1, unofficial].
*   **Limitations:** Computationally expensive; prone to artifacts like Janus and floaters; sensitive to timestep selection and guidance scale [S1, unofficial][S2].
*   **When to Use:** When high-fidelity 2D diffusion models are available and computational resources are sufficient for iterative optimization.
*   **When Not to Use:** When real-time performance is required or when multi-view consistency is critical without additional modifications (e.g., using MVDream instead) [S2].

## Background
Score Distillation Sampling (SDS) was introduced by Poole et al. in "DreamFusion: Text-to-3D using 2D Diffusion" (2022) [S1, unofficial]. It enables the optimization of 3D representations (like NeRFs or 3D Gaussians) using gradients from a pre-trained 2D text-to-image diffusion model. The method has become a standard technique in text-to-3D generation, with numerous variants and improvements proposed in subsequent works [S2].

## Sources
[S1] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
[S4] Neural Volumetric Mesh Generator, https://arxiv.org/abs/2210.03158v1, fetched 2026-10-27 (unofficial)
