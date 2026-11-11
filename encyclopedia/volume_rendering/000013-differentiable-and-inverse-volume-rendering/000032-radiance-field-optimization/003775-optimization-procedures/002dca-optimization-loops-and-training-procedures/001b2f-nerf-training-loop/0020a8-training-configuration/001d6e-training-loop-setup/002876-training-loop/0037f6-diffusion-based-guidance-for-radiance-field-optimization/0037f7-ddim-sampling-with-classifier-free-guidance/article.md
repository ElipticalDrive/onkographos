# DDIM Sampling with Classifier-Free Guidance

Also called: DDIMSampler, refine

## Overview
Iteratively denoises latent variables using the DDIM scheduler and classifier-free guidance (CFG) to generate images or refine existing ones [S1, unofficial]. It sets itself apart from sibling methods like Inpainting SDS Loss by operating on the full latent space with unconditional/conditional embedding concatenation for global guidance, rather than masking regions for local consistency [S1, 11185-inpainting-sds-loss.md, unofficial].

## How it works
**Inputs:**
- `latents`: Initial noise latents of shape `(batch_size, unet.in_channels, height // 8, width // 8)` [S1, unofficial]. If not provided, initialized as `torch.randn` [S1, unofficial].
- `embeddings`: Dictionary containing `pos` (conditional) and `neg` (unconditional) embeddings [S1, unofficial].
- `num_inference_steps`: Number of sampling steps [S1, unofficial].
- `guidance_scale`: CFG multiplier [S1, unofficial].
- `strength`: (Optional) For refinement, determines the starting timestep [S1, unofficial].

**Outputs:**
- Refined or generated latents [S1, unofficial].

**Data Structures:**
- `latents`: Tensor of shape `(B, C, H', W')` where $H' = H/8, W' = W/8$ [S1, unofficial].
- `embeddings`: Tensor of shape `(B, E, D)` where $E$ is sequence length and $D$ is embedding dimension [S1, unofficial].

**Steps:**
1. **Initialize Latents:** If `latents` is not provided, generate random noise:
   $$ \text{latents} \sim \mathcal{N}(0, 1) $$
   Shape: `(1, self.unet.in_channels, height // 8, width // 8)` [S1, unofficial].
2. **Set Timesteps:** Configure the DDIM scheduler with `num_inference_steps` [S1, unofficial].
3. **Prepare Embeddings:** Concatenate positive and negative embeddings along the batch dimension for CFG:
   $$ \text{embeddings} = \text{concat}([\text{pos}.expand(B, -1, -1), \text{neg}.expand(B, -1, -1)]) $$
   Resulting shape: `(2*B, E, D)` [S1, unofficial].
4. **Iterate Timesteps:** For each timestep $t$ in the scheduler's sequence:
   a. **Duplicate Latents:** Concatenate latents to match the doubled batch size for CFG:
      $$ \text{latent\_model\_input} = \text{concat}([\text{latents}, \text{latents}]) $$
      Shape: `(2*B, C, H', W')` [S1, unofficial].
   b. **Predict Noise:** Pass `latent_model_input` and `embeddings` through the UNet to get noise predictions:
      $$ [\hat{\epsilon}_{\text{cond}}, \hat{\epsilon}_{\text{uncond}}] = \text{UNet}(\text{latent\_model\_input}, \text{embeddings}) $$
   c. **Apply CFG:** Compute the guided noise prediction:
      $$ \hat{\epsilon} = \hat{\epsilon}_{\text{uncond}} + \text{guidance\_scale} \times (\hat{\epsilon}_{\text{cond}} - \hat{\epsilon}_{\text{uncond}}) $$
   d. **Step Scheduler:** Update latents using the DDIM scheduler step:
      $$ \text{latents} = \text{scheduler.step}(\hat{\epsilon}, t, \text{latents}).\text{prev\_sample} $$
5. **Refinement (Optional):** If refining, skip early timesteps based on `strength`:
   $$ \text{init\_step} = \lfloor \text{steps} \times \text{strength} \rfloor $$
   Noise is added at `init_step` to preserve structure while adding detail [S1, unofficial].

**Parameters:**
- `num_inference_steps`: Default 50 [S1, unofficial].
- `guidance_scale`: Default 7.5 for sampling, 100 for refinement/training [S1, unofficial].

**Pitfalls:**
- Precision issues may arise if `guidance_scale` is too high, leading to over-saturation or artifacts [S1, unofficial].
- Boundary conditions: Ensure `latents` shape matches UNet input requirements (divisible by 8 in spatial dimensions) [S1, unofficial].
- Degenerate input: If `embeddings` are missing or mismatched, CFG fails [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def ddim_sampling_with_cfg(latents, embeddings, unet, scheduler, num_inference_steps, guidance_scale, strength=0.0):
    """
    Perform DDIM sampling with Classifier-Free Guidance.
    
    Args:
        latents: Initial noise latents or None.
        embeddings: Dict with 'pos' and 'neg' embeddings.
        unet: UNet model for noise prediction.
        scheduler: DDIM scheduler instance.
        num_inference_steps: Number of sampling steps.
        guidance_scale: CFG scale multiplier.
        strength: Refinement strength (0.0 to 1.0).
        
    Returns:
        Refined latents.
    """
    # Step 1: Initialize latents if not provided
    if latents is None:
        # Assuming batch_size=1, device='cuda', height/width from context
        # In practice, height/width must be known to determine latent shape
        # Placeholder for height/width extraction from embeddings or context
        # [G] Assuming standard SD latent dimensions: height=512, width=512
        height, width = 512, 512
        latents = torch.randn((1, unet.in_channels, height // 8, width // 8), device=unet.device)
        
    batch_size = latents.shape[0]
    
    # Step 2: Set scheduler timesteps
    scheduler.set_timesteps(num_inference_steps)
    
    # Step 3: Concatenate embeddings for CFG
    pos_emb = embeddings['pos'].expand(batch_size, -1, -1)
    neg_emb = embeddings['neg'].expand(batch_size, -1, -1)
    embeddings_cat = torch.cat([pos_emb, neg_emb], dim=0)
    
    # Determine starting step for refinement
    if strength > 0.0:
        init_step = int(num_inference_steps * strength)
        # Skip early timesteps
        timesteps = scheduler.timesteps[init_step:]
    else:
        timesteps = scheduler.timesteps
        
    # Step 4: Loop over timesteps
    for t in timesteps:
        # a. Duplicate latents for CFG
        latent_model_input = torch.cat([latents, latents], dim=0)
        
        # b. Predict noise via UNet
        # [G] Assuming UNet takes (input, timestep, encoder_hidden_states)
        noise_pred = unet(latent_model_input, t, embeddings_cat).sample
        
        # c. Apply CFG
        noise_pred_cond, noise_pred_uncond = noise_pred.chunk(2)
        noise_pred = noise_pred_uncond + guidance_scale * (noise_pred_cond - noise_pred_uncond)
        
        # d. Step scheduler
        latents = scheduler.step(noise_pred, t, latents).prev_sample
        
    return latents
```

## Subtopics

### NeRF Interpolation via DDIM and SLERP
A procedure to generate smooth transitions between 3D samples by interpolating in the diffusion latent space before deterministic decoding [S2]. It samples two initial diffusion latent values $x^{(T)} \sim \mathcal{N}(0, I)$, interpolates them using spherical linear interpolation (SLERP), and passes the result through a deterministic solver [S2]. Standard Gaussian diffusion models often produce non-smooth interpolation results, and models trained with long schedules for unconditional generation yield distinct but discontinuous interpolated samples [S2]. Early stopping for sparse-view reconstruction preserves a smoother prior, yielding better generalization and smoother transitions than long unconditional training schedules [S2]. Used in SSDNeRF to interpolate between generated 3D samples [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to `num_inference_steps` and UNet inference time per step [S1, unofficial].
- **Memory Complexity:** Proportional to latent size and UNet memory footprint [S1, unofficial].
- **Performance:** Default 50 steps provide a balance between quality and speed [S1, unofficial].
- **Strengths:** High-quality image generation and refinement; effective guidance via CFG [S1, unofficial].
- **Limitations:** Computationally expensive due to iterative denoising; requires careful tuning of `guidance_scale` and `strength` [S1, unofficial].
- **When to Use:** For high-fidelity image generation or refinement where global structure is important [S1, unofficial].
- **When Not to Use:** For real-time applications or when local consistency is prioritized over global structure (use Inpainting SDS Loss instead) [11185-inpainting-sds-loss.md].

## Background
Originated from the DreamGaussian project, which utilizes DDIM sampling with CFG for 3D-aware image generation and refinement [S1, unofficial]. The method builds on the DDIM sampler from Stable Diffusion pipelines [S1, unofficial]. SSDNeRF extends this concept to 3D NeRF interpolation, highlighting the importance of early stopping for smooth transitions [S2].

## Sources
[S1] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S2] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
