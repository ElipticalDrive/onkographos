# Diffusion-Based Guidance for Radiance Field Optimization

Also called: 2D SDS Loss, Diffusion-guided NeRF, DreamFusion SDS, L2D, NeRF Optimization with SDS, Score Distillation Sampling

## Overview
Diffusion-Based Guidance for Radiance Field Optimization uses the score function of a pre-trained 2D diffusion model to provide gradient signals that guide the optimization of a differentiable radiance field parametrization $z = g(\theta)$ towards the mode of a conditional probability distribution $p(z_t | y)$ [S1]. It solves the problem of generating or refining 3D scenes from limited 2D inputs (such as text prompts or single images) by leveraging the rich semantic prior of diffusion models without fine-tuning the diffusion network itself [S5, unofficial]. It sets itself apart from direct sampling methods like Diffusion Posterior Sampling (DPS) by optimizing the 3D representation parameters directly via backpropagation through the rendering function, rather than solving an inverse problem in the latent space of the diffusion model [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A differentiable radiance field parametrization $g(\theta)$ (e.g., NeRF MLP weights or 3D Gaussian parameters), a text prompt $y$ (or image condition), and a pre-trained 2D diffusion model $\phi$ (typically Stable Diffusion v1.5 or v2.1) [S1][S3][S6, unofficial].
*   **Output:** Optimized parameters $\theta$ that define a radiance field consistent with the prompt.

**Data Structures:**
*   **Radiance Field Parameters ($\theta$):** Varies by representation. For NeRF, $\theta$ includes MLP weights and biases [S1]. For 3D Gaussian Splatting (3DGS), $\theta$ includes means, covariances, opacities, and spherical harmonic coefficients [S3].
*   **Diffusion Model:** A frozen UNet-based autoencoder with a text encoder (e.g., CLIP) and a VAE [S5, unofficial].
*   **Latent Space:** Rendered images are encoded into latent space $z_0$ of shape $[B, 4, 64, 64]$ for Stable Diffusion v1.5/v2.1 [S5, unofficial].

**Numbered Steps:**

1.  **Rendering:** Render an image $I$ (or latent $z_0$) from the current radiance field $g(\theta)$ at a randomly sampled viewpoint [S1][S3]. For NeRF, this involves volume rendering along rays [S1]. For 3DGS, this involves alpha-blending splatted Gaussians [S3].
2.  **Preprocessing:**
    *   If optimizing in pixel space (2D SDS), interpolate $I$ to $512 \times 512$ [S4, unofficial][S6, unofficial].
    *   If optimizing in latent space (common for Stable Diffusion), encode $I$ using the diffusion model's VAE encoder to get $z_0$ [S5, unofficial].
    *   Prepare text embeddings $e$ for the prompt using the text encoder [S5, unofficial].
3.  **Noise Addition:** Sample a noise timestep $t$ uniformly from a range $[t_{min}, t_{max}]$ (commonly $[0.02, 0.98]$ of total timesteps) [S4, unofficial][S5, unofficial]. Sample noise $\epsilon \sim \mathcal{N}(0, I)$ [S5, unofficial]. Add noise to the latent/image:
    $$ z_t = \sqrt{\bar{\alpha}_t} z_0 + \sqrt{1 - \bar{\alpha}_t} \epsilon $$
    where $\bar{\alpha}_t$ is the cumulative product of the noise schedule [S1][S5, unofficial].
4.  **Noise Prediction:** Pass the noisy latent $z_t$, timestep $t$, and text embeddings $e$ into the frozen UNet to predict the noise $\hat{\epsilon}_\phi(z_t, y, t)$ [S5, unofficial].
    *   **Classifier-Free Guidance (CFG):** If guidance is enabled, duplicate inputs for conditional and unconditional (empty prompt) branches. Compute:
        $$ \hat{\epsilon} = \hat{\epsilon}_{uncond} + s (\hat{\epsilon}_{cond} - \hat{\epsilon}_{uncond}) $$
        where $s$ is the guidance scale (default 100) [S4, unofficial][S5, unofficial].
5.  **Gradient Computation (SDS Loss):** Compute the gradient of the radiance field parameters $\theta$ with respect to the SDS loss. The core insight is that the gradient of the diffusion loss with respect to the clean latent $z_0$ is proportional to the difference between predicted noise and actual noise [S1].
    *   The SDS objective gradient is defined as [S1]:
        $$ \nabla_\theta L_{SDS} = \mathbb{E}_{t, \epsilon} \left[ \gamma(t) (\hat{\epsilon}_\phi(z_t, y, t) - \epsilon) \frac{\partial z_0}{\partial \theta} \right] $$
    *   Here, $\gamma(t)$ is a weighting function, often chosen as $1 - \bar{\alpha}_t$ or related to the noise variance $\sigma_t$ [S1][S5, unofficial].
    *   In implementation, this is often computed as [S5, unofficial]:
        $$ \text{grad} = -w \cdot (\hat{\epsilon} - \epsilon) \cdot \text{grad\_scale} $$
        where $w = 1 - \bar{\alpha}_t$ [S5, unofficial].
    *   The term $\frac{\partial z_0}{\partial \theta}$ is obtained by backpropagating through the VAE decoder (if in latent space) and the rendering function $g(\theta)$ [S1][S5, unofficial].
6.  **Parameter Update:** Update $\theta$ using an optimizer (e.g., AdamW) with the computed gradient [S5, unofficial].
    *   **Clamping:** If optimizing latent tensors directly, clamp values to $[-1, 1]$ to prevent over-saturation [S5, unofficial].
    *   **NaN Handling:** Apply `torch.nan_to_num` to gradients to handle numerical instability [S4, unofficial].

**Parameters:**
*   **Guidance Scale ($s$):** Default 100 [S4, unofficial][S5, unofficial].
*   **Timestep Range ($t_{min}, t_{max}$):** Default $[0.02, 0.98]$ [S4, unofficial][S5, unofficial].
*   **Learning Rate:** Varies by optimizer setup; e.g., $10^{-1}$ for latent optimization [S5, unofficial].
*   **SDS Loss Weight:** Often applied as a multiplier; e.g., $5 \times 10^{-4}$ in 3DGS refinement [S3].
*   **Total Iterations:** e.g., 2000 for latent optimization [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Over-saturation and Over-smoothing:** SDS is known to cause these artifacts [S1]. Mitigation includes clamping latents, using specific weighting functions $\gamma(t)$, or combining with other losses [S1][S5, unofficial].
*   **NaN Gradients:** Numerical instability in the diffusion process or high guidance scales can produce NaNs. Explicitly clamping NaNs to 0 is required [S4, unofficial].
*   **Viewpoint Bias:** Sampling viewpoints only from training cameras can lead to poor novel view synthesis. Random viewpoints around the scene center should be used [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def diffusion_guided_optimization(radiance_field, text_prompt, diffusion_model, optimizer, num_iterations, guidance_scale=100):
    """
    Optimizes radiance field parameters using Score Distillation Sampling (SDS).
    
    Args:
        radiance_field: Differentiable 3D representation (e.g., NeRF, 3DGS) with parameters theta.
        text_prompt: String prompt for the diffusion model.
        diffusion_model: Pre-trained Stable Diffusion model (UNet, VAE, TextEncoder).
        optimizer: Optimizer for radiance_field parameters.
        num_iterations: Total number of optimization steps.
        guidance_scale: CFG scale multiplier.
    """
    # 1. Prepare text embeddings
    text_encoder = diffusion_model.text_encoder
    text_encoder.eval()
    with torch.no_grad():
        # Encode prompt
        input_ids = text_encoder.tokenizer(text_prompt, return_tensors="pt").input_ids.to(diffusion_model.device)
        text_embeddings = text_encoder(input_ids).last_hidden_state
        
        # Prepare unconditional embedding for CFG
        uncond_input_ids = text_encoder.tokenizer("", return_tensors="pt").input_ids.to(diffusion_model.device)
        uncond_embeddings = text_encoder(uncond_input_ids).last_hidden_state

    # 2. Optimization Loop
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        # 3. Render image from current radiance field
        # Assume render() returns RGB image of shape [H, W, 3]
        rendered_image = radiance_field.render() 
        
        # 4. Encode to latent space
        vae_encoder = diffusion_model.vae.encoder
        vae_encoder.eval()
        with torch.no_grad():
            # Interpolate to 512x512 if necessary, then encode
            rendered_image_resized = F.interpolate(rendered_image.permute(2, 0, 1).unsqueeze(0), size=(512, 512), mode='bilinear')
            latents = vae_encoder(rendered_image_resized).sample

        # 5. Sample noise and timestep
        t = torch.randint(0, diffusion_model.num_timesteps, (1,), device=diffusion_model.device).long()
        noise = torch.randn_like(latents)
        
        # 6. Add noise using scheduler
        # Assuming scheduler has add_noise method
        noisy_latents = diffusion_model.scheduler.add_noise(latents, noise, t)

        # 7. Predict noise with CFG
        # Duplicate latents and timesteps for conditional and unconditional
        latent_model_input = torch.cat([noisy_latents, noisy_latents])
        t_input = torch.cat([t, t])
        
        # Concatenate embeddings
        encoder_hidden_states = torch.cat([text_embeddings, uncond_embeddings])
        
        diffusion_model.unet.eval()
        with torch.no_grad():
            noise_pred = diffusion_model.unet(latent_model_input, t_input, encoder_hidden_states=encoder_hidden_states).sample
        
        # Split predictions
        noise_pred_cond, noise_pred_uncond = noise_pred.chunk(2)
        
        # Apply CFG
        noise_pred = noise_pred_uncond + guidance_scale * (noise_pred_cond - noise_pred_uncond)

        # 8. Compute SDS Gradient
        # Weight w = 1 - alpha_t (or similar weighting function)
        # Access alpha_t from scheduler or precomputed alphas
        alpha_t = diffusion_model.alphas[t].view(1, 1, 1, 1)
        w = 1 - alpha_t
        
        # Gradient of loss w.r.t latents: w * (noise_pred - noise)
        # We need gradient w.r.t radiance field params theta
        # This requires backprop through VAE decoder and rendering function
        # However, standard SDS formulation backpropagates through the UNet's output w.r.t input
        # But we do NOT backprop through UNet. We use the gradient of the noise prediction w.r.t the noisy latent.
        
        # Re-calculate noise_pred with requires_grad=True on latents
        noisy_latents.requires_grad_(True)
        encoder_hidden_states.requires_grad_(False) # Text embeddings are fixed
        
        # Forward pass through UNet with grad enabled on latents
        noise_pred_with_grad = diffusion_model.unet(noisy_latents, t, encoder_hidden_states=encoder_hidden_states).sample
        noise_pred_cond_grad, _ = noise_pred_with_grad.chunk(2) # Assuming uncond is not needed for grad if we use conditional only? 
        # Actually, standard implementation uses the conditional prediction for the gradient direction
        # Let's stick to the formula: grad = w * (noise_pred - noise)
        # But noise_pred here must be the one used for guidance.
        
        # Simplified standard implementation:
        # Compute loss as MSE between predicted noise and actual noise, weighted
        # But we need gradient w.r.t latents.
        
        # Correct SDS gradient computation:
        # L = 0.5 * || latents - (noisy_latents - w * (noise_pred - noise)) ||^2 ? 
        # No, the gradient is directly: grad_latents = w * (noise_pred - noise)
        
        # Let's compute the target denoised latent
        # z_0_hat = (z_t - sqrt(1-a_t) * eps_pred) / sqrt(a_t)
        # Gradient is often computed as: grad = w * (eps_pred - eps) * (dz_0 / dz_t) ?
        # The paper [S1] says: grad_theta = E [ gamma(t) (eps_pred - eps) * (dz_0 / d_theta) ]
        # So we need grad of (eps_pred - eps) w.r.t latents, then chain rule.
        
        # eps_pred is noise_pred_cond_grad
        eps_pred = noise_pred_cond_grad
        eps = noise
        
        # Gradient of loss w.r.t latents
        # L = 0.5 * || eps_pred - eps ||^2 ? No, that's not SDS.
        # SDS gradient is: grad_latents = w * (eps_pred - eps)
        # This is because d/dz_t ( - log p(z_t|y) ) is related to eps_pred.
        
        grad_latents = w * (eps_pred - eps)
        
        # Backprop through VAE Decoder to get grad w.r.t latents (z_0)
        # z_0 is the output of the VAE decoder? No, latents are z_0.
        # We rendered image I, encoded to z_0.
        # So we need d(z_0)/d(I) and d(I)/d(theta).
        
        # VAE Decoder: I_recon = VAE.decode(z_0)
        # We want grad w.r.t z_0.
        # But we have grad w.r.t noisy_latents (z_t).
        # Chain rule: dz_t / dz_0 = sqrt(a_t).
        # So grad_z0 = grad_zt / sqrt(a_t) ?
        
        # Actually, most implementations (e.g. DreamFusion) compute:
        # target = (latents - grad_latents).detach()
        # loss = MSE(latents, target)
        # This implicitly handles the chain rule correctly for the optimization step.
        
        target = (latents - grad_latents).detach()
        loss = 0.5 * F.mse_loss(latents.float(), target, reduction='sum') / latents.shape[0]
        
        # Backprop through VAE Decoder and Rendering Function
        # Note: We must backprop through the VAE decoder to get gradients w.r.t the rendered image,
        # and then through the rendering function to get gradients w.r.t theta.
        
        # Re-encode to ensure graph is correct? 
        # The graph is: theta -> Rendered Image -> VAE Encoder -> Latents -> Loss
        # But we used noisy_latents which depends on latents.
        
        # Let's restart the graph for clarity
        rendered_image = radiance_field.render()
        rendered_image_resized = F.interpolate(rendered_image.permute(2, 0, 1).unsqueeze(0), size=(512, 512), mode='bilinear')
        latents = vae_encoder(rendered_image_resized).sample
        latents.requires_grad_(True)
        
        noisy_latents = diffusion_model.scheduler.add_noise(latents, noise, t)
        noisy_latents.requires_grad_(True)
        
        encoder_hidden_states = torch.cat([text_embeddings, uncond_embeddings])
        noise_pred = diffusion_model.unet(noisy_latents, t, encoder_hidden_states=encoder_hidden_states).sample
        noise_pred_cond, _ = noise_pred.chunk(2)
        
        # Apply CFG to get the noise prediction used for guidance
        # Note: We don't backprop through the CFG combination if we treat it as a fixed vector?
        # No, we backprop through the conditional prediction.
        
        eps_pred = noise_pred_cond
        eps = noise
        w = 1 - alpha_t
        
        grad_latents = w * (eps_pred - eps)
        
        # Target for MSE loss
        target = (latents - grad_latents).detach()
        loss = 0.5 * F.mse_loss(latents.float(), target, reduction='sum') / latents.shape[0]
        
        # Backprop
        loss.backward()
        
        # Handle NaNs
        for p in radiance_field.parameters():
            if p.grad is not None:
                p.grad = torch.nan_to_num(p.grad)
                
        optimizer.step()
        
        # Optional: Clamp latents if optimizing latent directly (not applicable here as we optimize theta)
```

## Subtopics

### [Iterative Optimization with Learnable Parameters for Diffusion-Guided Radiance Fields](8945-iterative-optimization-with-learnable-parameters-for-diffusi.md)
This method addresses quality impairments in 3D generation caused by the discrepancy between the diffusion prior and the training process of the diffusion model. It sets itself apart from standard diffusion-guided approaches by introducing an iterative optimization loop that jointly updates the 3D model and specific learnable parameters within the diffusion guidance mechanism, rather than treating the diffusion model as a static prior.

### [Scene Generation and Relighting](14392-scene-generation-and-relighting.md)

### [Diffusion Sampling and Denoising Mechanisms](14430-diffusion-sampling-and-denoising-mechanisms.md)
This family defines the algorithms and network architectures used to invert the forward diffusion process, recovering clean latent representations of radiance fields from noisy inputs. It solves the problem of generating or reconstructing 3D scenes by iteratively denoising a latent volume or code. It differs from direct optimization methods by leveraging the learned data prior of a diffusion model to guide the reconstruction, often using classifier-free guidance to steer the output toward conditional targets.

### [Diffusion-Based 3D Inpainting](14834-diffusion-based-3d-inpainting.md)
Diffusion-Based 3D Inpainting is a method that utilizes a self-trained 2D inpainting diffusion model to complete 3D scenes or radiance fields where specific regions are masked out. It solves the problem of generating geometrically and photometrically consistent content for missing areas by combining a reconstruction loss for observed regions with a Score Distillation Sampling (SDS) loss for masked regions. It differs from sibling methods like [Iterative Optimization with Learnable Parameters for Diffusion-Guided Radiance Fields](8945-iterative-optimization-with-learnable-parameters-for-diffusi.md) by explicitly separating the supervision signal into unmasked reconstruction and masked diffusion priors, rather than relying solely on general score distillation.

### [Diffusion Model Preprocessing & Training Setup](15506-diffusion-model-preprocessing-training-setup.md)
This method defines the data preprocessing pipeline and hyperparameter configuration required to fine-tune a 2D diffusion model (specifically Stable Diffusion 1.5) to serve as a semantic prior for radiance field optimization. It solves the problem of overfitting to specific camera viewpoints and occlusion artifacts by decoupling semantic identity learning from spatial pose control, ensuring the model generalizes to novel views. It sets itself apart from methods like ReconFusion by explicitly managing the perspective gap between reference and training images through distinct cropping strategies.

### [Two-Plane Diffusion Model (TPDM)](18131-two-plane-diffusion-model-tpdm.md)
Two-Plane Diffusion Model (TPDM) is a method for solving 3D inverse problems and generating 3D voxel volumes by training two separate 2D diffusion models on perpendicular planes and alternating their usage during the denoising process to maintain 3D consistency. It avoids the curse of dimensionality by assuming a 3D distribution is the product of 2D distributions, allowing it to handle 3D volumes without using a 3D diffusion model. Unlike [Diffusion-Based Guidance for Radiance Field Optimization](8117-diffusion-based-guidance-for-radiance-field-optimization.md) which guides optimization via gradients, TPDM performs conditional sampling directly on the volume using a primary model for measurement consistency and an auxiliary model for cross-plane consistency.

### [Score Distillation Loss Modifications](18224-score-distillation-loss-modifications.md)
Score Distillation Loss Modifications are optimization objectives that adapt the gradient signal from a frozen 2D diffusion model to refine a 3D radiance field or latent representation. They solve the limitations of vanilla Score Distillation Sampling (SDS), such as oversaturation, poor composition, or lack of high-frequency detail, by altering how the diffusion model is conditioned, how noise is sampled, or how the loss is computed. This family is distinguished from its siblings by focusing specifically on the mathematical formulation of the loss function and its conditioning inputs, rather than the broader training loop architecture or sampling mechanisms.

### [Regularization and Refinement Techniques](18361-regularization-and-refinement-techniques.md)

### [Pix2Pix Denoising Network for Point Clouds](18911-pix2pix-denoising-network-for-point-clouds.md)
A generative adversarial network (GAN) based method to denoise point cloud renderings by mapping noisy splat images to clean splat images, followed by a 3D update via Differentiable Surface Splatting (DSS) [DSS](52-differentiable-surface-splatting.md). It solves the ill-posed problem of point cloud denoising where handcrafted priors struggle to recover both smooth and sharp features simultaneously. Unlike optimization-based methods that rely on local geometric priors, this approach leverages a learned image-to-image translation network to hallucinate plausible high-frequency details, which are then transferred to 3D geometry through simultaneous multi-view optimization.

### [MaterialFusion](19399-materialfusion.md)
MaterialFusion is a 3D inverse rendering pipeline that reconstructs an object’s geometry, physically-based rendering (PBR) material properties, and environmental illumination from multi-view images under unknown lighting conditions. It addresses the intrinsic challenge of disentangling albedo and material properties from input images, which typically leads to poor relighting performance in novel conditions. Unlike standard inverse rendering methods that optimize geometry and materials solely based on photometric consistency, MaterialFusion incorporates a 2D diffusion prior (StableMaterial) to guide the optimization of albedo and material textures, ensuring physically plausible and disentangled results.

### [Style and Content Adaptation](19896-style-and-content-adaptation.md)
Style and Content Adaptation modifies a pre-trained radiance field to incorporate specific visual styles or object contents by leveraging external conditions, such as reference images or text prompts, during the optimization loop. It distinguishes itself from sibling methods like [SKED](12268-sked.md) and [Single-Image Textual Inversion](19221-single-image-textual-inversion.md) by explicitly using reference images as conditional priors to guide the diffusion model's denoising process, ensuring the generated content adheres to the visual characteristics of the reference rather than relying solely on textual descriptions or single-view embeddings.

### [Latent Space and Autoencoder Methods](21348-latent-space-and-autoencoder-methods.md)
This family of methods integrates pre-trained 2D diffusion models or autoencoders with 3D radiance field representations to solve the ill-posed problem of 3D reconstruction and editing from limited 2D inputs. Unlike sibling methods that apply diffusion guidance directly to pixel space or use iterative denoising for sampling, these methods operate in the compressed latent space of a Variational Autoencoder (VAE) to enforce geometric consistency and photorealism. The core distinction lies in whether the optimization targets the diffusion noise prediction error (as in the Rendering and Diffusion Prior Pipeline) or jointly trains the autoencoder with the 3D representation to ensure 3D-consistent latent features (as in the 3D-aware Autoencoder).

## Operating characteristics
*   **Time Complexity:** Each iteration requires one forward pass through the radiance field (rendering), one forward pass through the VAE encoder, one forward pass through the diffusion UNet, and one backward pass through the UNet (for gradient computation) and the rendering function. The UNet pass is the most expensive [S5, unofficial].
*   **Memory Complexity:** Requires storing the radiance field parameters, the diffusion model weights (frozen), and intermediate activations for the backward pass through the UNet and VAE [S5, unofficial].
*   **Measured Performance:** SDS consistently produces inferior results compared to Diffusion Posterior Sampling (DPS) for environment map generation, failing to generate realistic samples [S2]. However, it is widely used for text-to-3D due to its simplicity and compatibility with any differentiable renderer [S1].
*   **Strengths:** Does not require fine-tuning the diffusion model; works with any differentiable 3D representation; leverages strong 2D priors [S5, unofficial].
*   **Limitations:** Prone to over-saturation and over-smoothing; requires careful tuning of guidance scale and timestep range; computationally expensive due to repeated UNet calls [S1][S4, unofficial].
*   **When to Use:** Text-to-3D generation, single-image 3D reconstruction, refining 3D models with semantic priors [S1][S5, unofficial].
*   **When Not to Use:** When high-fidelity geometric consistency is critical without post-processing; when computational resources are extremely limited; when the diffusion model's prior conflicts with the scene content (e.g., complex textures) [S1][S2].

## Background
Score Distillation Sampling (SDS) was introduced by Poole et al. in "DreamFusion: Text-to-3D using 2D Diffusion" (2022) [S11, unofficial]. It leverages the score function of pre-trained diffusion models to guide the optimization of differentiable image parametrizations [S1]. The method has since been adapted for various 3D representations, including NeRFs [S1], 3D Gaussian Splatting [S3], and latent representations [S5, unofficial]. It is a cornerstone of modern text-to-3D generation pipelines.

## Sources
[S1] DiSR-NeRF: Diffusion-Guided View-Consistent Super-Resolution NeRF, https://doi.org/10.1109/cvpr52733.2024.01943, fetched 2026-10-26
[S2] Diffusion Posterior Illumination for Ambiguity-Aware Inverse Rendering, https://doi.org/10.1145/3618357, fetched 2026-10-26
[S3] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
[S4] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S5] ShashwatChawla/Diffusion-guided-3D-Representations, https://github.com/ShashwatChawla/Diffusion-guided-3D-Representations, fetched 2026-10-27 (unofficial)
[S6] Magic123: One Image to High-Quality 3D Object Generation Using Both 2D and 3D Diffusion Priors, https://doi.org/10.48550/arxiv.2306.17843, fetched 2026-10-30 (unofficial)
[S7] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S8] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S9] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S10] DiffRF: Rendering-Guided 3D Radiance Field Diffusion, https://doi.org/10.1109/cvpr52729.2023.00421, fetched 2026-10-26
[S11] DreamFusion: Text-to-3D using 2D Diffusion, https://doi.org/10.48550/arxiv.2209.14988, fetched 2026-10-27 (unofficial)
[S12] Segment Anything in 3D with Radiance Fields, https://doi.org/10.1007/s11263-025-02421-7, fetched 2026-10-26
