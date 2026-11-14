# Diffusion-Based Guidance for Radiance Field Optimization

Also called: Score Distillation Sampling, Stable Diffusion Training Guidance

## Overview
Diffusion-Based Guidance for Radiance Field Optimization computes a gradient signal from a frozen 2D diffusion model (typically Stable Diffusion) to optimize a radiance field representation by predicting noise, applying classifier-free guidance, and backpropagating the difference between predicted and actual noise [S1, unofficial]. It solves the problem of generating 3D geometry and texture from 2D priors without 3D ground-truth data, enabling text-to-3D and image-to-3D synthesis [S1, unofficial]. It sets itself apart from sibling methods like [VCCNF Training Loop](4736-vccnf-training-loop.md) by relying on the generative prior of a diffusion model rather than consistency losses between multi-view renderings, and from [MVDream Multi-View Diffusion Guidance](6345-mvdream-multi-view-diffusion-guidance.md) by typically operating on single-view or sparse-view supervision rather than explicit multi-view diffusion consistency [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A differentiable radiance field (NeRF or 3D Gaussian Splatting) parameterized by $\theta$; a text prompt $T$; a camera pose $C$; a timestep $t$; and a guidance scale $s$.
*   **Outputs:** A scalar loss value $\mathcal{L}$ and gradients $\nabla_\theta \mathcal{L}$ used to update the radiance field parameters.

**Data Structures:**
*   **Latents:** A tensor of shape $[B, 4, 64, 64]$ representing the encoded image in the Stable Diffusion latent space [S1, unofficial].
*   **Embeddings:** Text embeddings from a CLIP text encoder, concatenated with optional directional embeddings [S1, unofficial].
*   **Timesteps:** Integer indices $t$ ranging from $0$ to $T_{max}$, representing the noise level in the diffusion process [S1, unofficial].

**Parameters:**
*   `guidance_scale`: Classifier-free guidance scale multiplier. Default value is 100 [S1, unofficial].
*   `t_range`: List $[0.02, 0.98]$ used to compute `min_step` and `max_step` as fractions of `num_train_timesteps` [S1, unofficial].
*   `step_ratio`: Float controlling timestep selection. $1.0$ maps to `min_step` (early diffusion), $0.0$ maps to `max_step` (late diffusion) [S1, unofficial].
*   `as_latent`: Boolean flag. If false, input RGB is interpolated to 512x512 and encoded via VAE; if true, input is assumed to be latents interpolated to 64x64 and scaled [S1, unofficial].

**Numbered Steps:**
1.  **Input Encoding:**
    *   If `as_latent` is false: Convert the rendered RGB image `pred_rgb` to the model's dtype. Interpolate to 512x512. Encode via the VAE encoder to obtain latents $z$ of shape $[B, 4, 64, 64]$ [S1, unofficial].
    *   If `as_latent` is true: Interpolate to 64x64 and scale by $2x - 1$ [S1, unofficial].
2.  **Weighting Calculation:** Compute the weighting factor $w = (1 - \alpha_t)$, where $\alpha_t$ is the noise schedule value at timestep $t$. Reshape to $[B, 1, 1, 1]$ [S1, unofficial].
3.  **Noise Addition:** Sample noise $\epsilon \sim \mathcal{N}(0, I)$ with the same shape as $z$. Compute noisy latents $z_t = \text{scheduler.add\_noise}(z, \epsilon, t)$ [S1, unofficial].
4.  **Classifier-Free Guidance (CFG) Preparation:**
    *   Duplicate noisy latents: $z_{model\_input} = [z_t; z_t]$ [S1, unofficial].
    *   Duplicate timesteps: $t_{model} = [t; t]$ [S1, unofficial].
    *   Prepare text embeddings. Concatenate positive embeddings $E_{pos}$ (from prompt $T$) and negative embeddings $E_{neg}$ (empty prompt). If directional information (azimuth angles `hors`) is provided, select directional embeddings (`front` for $|h|<60^\circ$, `side` for $60^\circ \le |h| < 120^\circ$, `back` otherwise) and concatenate with $E_{neg}$ [S1, unofficial].
5.  **Noise Prediction:** Pass inputs to the UNet: $\hat{\epsilon} = \text{UNet}(z_{model\_input}, t_{model}, \text{encoder\_hidden\_states}=[E_{pos}; E_{neg}])$. The output $\hat{\epsilon}$ has shape $[2B, 4, 64, 64]$ [S1, unofficial].
6.  **Guidance Application:** Split $\hat{\epsilon}$ into conditional $\hat{\epsilon}_{cond}$ and unconditional $\hat{\epsilon}_{uncond}$ predictions. Compute the guided noise prediction:
    $$ \hat{\epsilon}_{guided} = \hat{\epsilon}_{uncond} + s \cdot (\hat{\epsilon}_{cond} - \hat{\epsilon}_{uncond}) $$
    where $s$ is `guidance_scale` [S1, unofficial].
7.  **Gradient Computation:** Compute the gradient signal:
    $$ g = w \cdot (\hat{\epsilon}_{guided} - \epsilon) $$
    Apply `torch.nan_to_num(g)` to replace NaN values with 0 to handle numerical instability [S1, unofficial].
8.  **Target and Loss:** Compute the target latent:
    $$ z_{target} = (z - g).detach() $$
    Compute the loss:
    $$ \mathcal{L} = \frac{0.5}{B} \sum_{i=1}^{B} \| z_i - z_{target, i} \|_2^2 $$
    where the sum is over the batch dimension [S1, unofficial].
9.  **Backpropagation:** Backpropagate $\mathcal{L}$ through the radiance field rendering pipeline to update $\theta$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sds_loss(radiance_field, camera_pose, text_prompt, timestep, guidance_scale, as_latent=False):
    # 1. Render image from radiance field
    pred_rgb = radiance_field.render(camera_pose)
    
    # 2. Encode to latent space
    if not as_latent:
        # Interpolate to 512x512 and encode via VAE
        latents = vae_encoder(interpolate(pred_rgb, 512))
    else:
        # Interpolate to 64x64 and scale
        latents = (interpolate(pred_rgb, 64) * 2 - 1)
        
    # 3. Compute weighting factor
    alpha_t = noise_schedule.alpha[timestep]
    w = (1 - alpha_t).view(latents.shape[0], 1, 1, 1)
    
    # 4. Add noise
    noise = torch.randn_like(latents)
    latents_noisy = scheduler.add_noise(latents, noise, timestep)
    
    # 5. Prepare CFG inputs
    latent_model_input = torch.cat([latents_noisy, latents_noisy])
    tt = torch.cat([timestep, timestep])
    
    # 6. Prepare embeddings
    pos_embed = text_encoder(text_prompt)
    neg_embed = text_encoder("") # Empty prompt
    
    # Optional: Add directional embeddings if available
    # if hors is provided:
    #     dir_embed = select_directional_embedding(hors)
    #     neg_embed = torch.cat([neg_embed, dir_embed], dim=-1)
    
    embeddings = torch.cat([pos_embed, neg_embed])
    
    # 7. UNet prediction
    noise_pred = unet(latent_model_input, tt, encoder_hidden_states=embeddings).sample
    
    # 8. Apply CFG
    noise_pred_cond, noise_pred_uncond = torch.chunk(noise_pred, 2, dim=0)
    noise_pred_guided = noise_pred_uncond + guidance_scale * (noise_pred_cond - noise_pred_uncond)
    
    # 9. Compute gradient
    grad = w * (noise_pred_guided - noise)
    grad = torch.nan_to_num(grad)
    
    # 10. Compute loss
    target = (latents - grad).detach()
    loss = 0.5 * F.mse_loss(latents.float(), target, reduction='sum') / latents.shape[0]
    
    return loss
```

## Subtopics

### [Inpainting SDS Loss](11185-inpainting-sds-loss.md)
Inpainting SDS Loss is a training objective for Neural Radiance Fields (NeRFs) that uses a pre-trained 2D diffusion inpainting model to generate 3D content that harmonizes with an existing scene. It solves the problem of incomplete or misaligned object generation common in vanilla SDS, where generated parts (e.g., a cat's head) fail to composite properly with the background. Unlike vanilla SDS, which optimizes based on global noise prediction, this method conditions the diffusion model on a masked image and mask derived from a 3D bounding box, ensuring complete geometry and proper pose relative to the scene context.

### SKED
SKED is a method for editing a Neural Radiance Field (NeRF) scene using guidance from 2D sketches provided from multiple views, in addition to an input text prompt [S1, unofficial][S2]. It addresses the problem of localized 3D editing, specifically object insertion and replacement, by ensuring that edits are confined to sketched regions while preserving the unedited parts of the original scene [S1, unofficial][S2]. It is distinguished from sibling methods like [Inpainting SDS Loss](11185-inpainting-sds-loss.md) by its explicit use of multi-view 2D sketches as spatial priors to define the edit region, rather than relying solely on text prompts or single-view masks [S1, unofficial][S2].

### [Synthetic Training with EG3D Supervision](13312-synthetic-training-with-eg3d-supervision.md)
This method trains an image-to-triplane encoder to reconstruct a 3D radiance field representation from a single input image by leveraging synthetic data generated by a frozen, pretrained EG3D generator. It solves the problem of generalizing single-image portrait synthesis by ensuring the encoder produces triplanes that are consistent not only with the input reference view but also with novel multiview renderings, thereby enforcing geometric and photometric coherence. It differs from standard supervised training by using a frozen generative model as a ground-truth oracle and alternating between reference-view and multiview supervision to prevent overfitting to the specific distribution of the input image.

### [Mask+SDS with Background Prompt](14382-mask-sds-with-background-prompt.md)
Mask+SDS with Background Prompt is a differentiable volume rendering optimization method that uses a frozen Stable Diffusion model to edit specific regions of a scene while explicitly guiding the generation of surrounding content. It solves the problem of "black holes" (unsupervised, often black or undefined regions) that occur when masking out parts of a NeRF for editing, by appending a background description to the diffusion model's text prompt. It sets itself apart from standard Mask+SDS by providing explicit semantic guidance for the occluded areas, and from Mask+SDS+CLIP by relying on the diffusion model's generative prior for the background rather than a separate CLIP-based texture fill.

### [Scene Generation and Relighting](14392-scene-generation-and-relighting.md)
Not given in the sources.

### [Diffusion Sampling and Denoising Mechanisms](14430-diffusion-sampling-and-denoising-mechanisms.md)
Not given in the sources.

### [Diffusion-Based 3D Inpainting](14834-diffusion-based-3d-inpainting.md)
Diffusion-Based 3D Inpainting is a method that utilizes a self-trained 2D inpainting diffusion model to reconstruct missing or masked regions in a 3D scene represented by a radiance field. It solves the problem of 3D scene completion or object removal by combining explicit reconstruction constraints for visible areas with implicit diffusion priors for occluded areas. It is distinguished from sibling methods like [Inpainting SDS Loss](11185-inpainting-sds-loss.md) by its specific reliance on a *self-trained* 2D inpainting model rather than a generic pre-trained one, and from [SKED](12268-sked.md) by its focus on distilling 2D inpainting results into a single 3D scene via iterative refinement.

#### Iterative Optimization with Learnable Parameters for Diffusion-Guided Radiance Fields
A method addressing quality impairment caused by differences between the diffusion prior and the training process of the diffusion model [S2]. Iterative optimization of the 3D model and the diffusion prior is performed [S2]. A learnable unconditional embedding is introduced in the classifier-free guidance formula [S2]. Additional parameters, such as LoRA parameters, are added to the network [S2].

#### Interval Score Matching
A modification to SDS loss that replaces DDPM with DDIM inversion and introduces supervisions from interval steps to avoid high error in one-step reconstruction [S2].

#### Context-Translation Training with Segmentation Masking
Pass a segmentation mask to the training process where pixels excluded in the stylization process are masked out [S3]. Calculate the loss only on the pixels outside the mask to avoid changing the appearance of masked pixels [S3]. Employ a background loss $L_{BG}$ as an additional regularization loss within the stylization process [S3]. Use $L_{BG}$ to prevent the model from introducing unwanted style artifacts in the background [S3]. $L_{BG}$ is defined as the L1 distance between the ground truth pixels of the background in the rendered image and the pixels in the background of the GT image [S3].

#### Style Adaptation via Diffusion Guidance
Use a reference image as a condition alongside the text prompt to guide generation [S4].

#### Blending Refinement Model
A model employing two Instruct-Pix2Pix processes as learned regularization to improve quality by purifying noisy mixup images and aligning them with detailed characteristics, using averaged latent codes for stability [S5]. First pass: Use the mixup image as input and the original image as a condition to purify the noisy mixup image by preserving structural integrity [S5]. Second pass: Use the resulting image from the first pass as input and the mixup image as the condition to align with the mixup image’s detailed characteristics [S5]. Introduce $n_r$ noised latent codes and update them independently. Compute the average of the latent codes and decode it back into the image [S5]. $n_r$ (number of noised latent codes for averaging) is set to 5 [S5]. Input timestep $t$ is set to 0.6 [S5]. Diffusion steps are 3 [S5].

#### Masked-NeRF + DreamFusion
A baseline method that uses per-scene text prompts to guide the generation of masked regions using score distillation sampling (SDS) loss [S6]. Gradually and uniformly decrease the maximum noise steps $t_{max}$ during fitting, until it equals the minimum noise steps $t_{min}$ at the last iteration [S6]. $t_{max}$ is the maximum noise steps, decreased to $t_{min}$ [S6]. $t_{min}$ is the minimum noise steps [S6].

## Operating characteristics
*   **Time Complexity:** The method requires one or two forward passes through the UNet per optimization step (depending on CFG implementation), plus the differentiable rendering pass. The UNet pass is the dominant cost.
*   **Memory Complexity:** Requires storing the radiance field parameters, the UNet weights (frozen), and intermediate latents. The duplication of latents and embeddings for CFG doubles the memory for the diffusion model inputs.
*   **Measured Performance:** Performance depends heavily on the `guidance_scale` and timestep selection. High guidance scales (e.g., 100) can lead to over-saturation or artifacts if not balanced with proper timestep scheduling [S1, unofficial].
*   **Scaling:** Scales with the number of training iterations and the resolution of the rendered images. Higher resolutions require larger VAE encodings or more memory.
*   **Strengths:** Enables high-quality 3D generation from text or single images without 3D data. Flexible conditioning via text, masks, and sketches.
*   **Limitations:** Prone to "Janus" problems (multi-faced objects) and floaters. Computationally expensive due to UNet evaluations. Sensitive to hyperparameters like guidance scale and timestep range.
*   **When to use:** Text-to-3D or image-to-3D generation where 3D ground truth is unavailable.
*   **When not to use:** When high-fidelity geometric accuracy is required without generative priors, or when computational resources are severely limited.

## Background
Originated with DreamFusion [S1, unofficial][S2], which introduced Score Distillation Sampling (SDS) to optimize NeRFs using a frozen Stable Diffusion model. The method was later adapted for 3D Gaussian Splatting (3DGS) [S1, unofficial][S2]. Variants like Interval Score Matching (ISM) [S2] and iterative optimization with learnable parameters [S2] have been proposed to address quality issues. It is widely used in text-to-3D and image-to-3D synthesis pipelines.

## Sources
[S1] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Shape-Shifting Splats: Realtime Context Translation for Gaussian Splatting in VR, https://doi.org/10.1109/vr67842.2026.00046, fetched 2026-11-01
[S4] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S5] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S6] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
