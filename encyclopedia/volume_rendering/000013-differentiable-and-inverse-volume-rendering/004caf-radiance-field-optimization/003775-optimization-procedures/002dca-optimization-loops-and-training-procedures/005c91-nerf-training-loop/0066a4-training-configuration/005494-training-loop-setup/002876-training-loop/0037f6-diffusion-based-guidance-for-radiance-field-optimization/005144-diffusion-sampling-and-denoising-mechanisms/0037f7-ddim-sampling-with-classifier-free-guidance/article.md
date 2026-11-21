# DDIM Sampling with Classifier-Free Guidance

Also called: Denoising Diffusion Implicit Models, DDIM Sampling, DDIM Sampling Step (with CFG & v-parameterization)

## Overview
DDIM Sampling with Classifier-Free Guidance (CFG) is a deterministic (or stochastic with $\eta > 0$) reverse diffusion algorithm that generates samples or refines existing ones by iteratively denoising latent variables [S1, S2, unofficial]. It sets itself apart from standard DDPM sampling by skipping intermediate noise steps to allow for fewer sampling iterations, and by combining unconditional and conditional model predictions via a guidance scale to enhance fidelity [S2, unofficial]. In the context of radiance field optimization, it serves as a denoising mechanism to refine NeRF renderings or guide 3D generation by leveraging 2D diffusion priors [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - $x_T$: Initial latent tensor (random noise if generating; existing latent if refining). Shape: $(B, C, H, W)$.
  - $c$: Conditional embedding (e.g., text or image features). Shape: $(B, D)$.
  - $timesteps$: Sequence of timesteps to traverse, typically reversed from $T$ to $0$ [S2, unofficial].
  - $unconditional\_guidance\_scale$: Scalar multiplier for CFG (default 1.0) [S2, unofficial].
  - $ddim\_eta$: Stochasticity parameter (default 0.0) [S2, unofficial].
  - $mask$: Optional inpainting mask tensor.
  - $x_0$: Optional original image for inpainting [S2, unofficial].
- **Outputs:**
  - $x_0$: Final denoised latent tensor.
  - Intermediates: Optional list of $x_t$ and predicted $\hat{x}_0$ at logging intervals [S2, unofficial].

**Data Structures:**
The scheduler maintains several precomputed buffers registered as CUDA tensors (float32) indexed by timestep [S2, unofficial]:
- `alphas_cumprod` ($\alpha_t$): Cumulative product of $\alpha$ values.
- `alphas_cumprod_prev` ($\alpha_{t-1}$): Previous cumulative alpha.
- `sqrt_alphas_cumprod` ($\sqrt{\alpha_t}$): Square root of cumulative alpha.
- `sqrt_one_minus_alphas_cumprod` ($\sqrt{1-\alpha_t}$): Square root of one minus cumulative alpha.
- `ddim_sigmas` ($\sigma_t$): DDIM-specific noise scale.
- `ddim_alphas` ($\alpha_t^{DDIM}$): Adjusted alphas for the DDIM schedule.
- `ddim_alphas_prev` ($\alpha_{t-1}^{DDIM}$): Previous adjusted alphas.
- `ddim_sqrt_one_minus_alphas`: $\sqrt{1-\alpha_t^{DDIM}}$.

**Numbered Steps:**

1. **Initialization:**
   - If $x_T$ is not provided, sample $x_T \sim \mathcal{N}(0, I)$ with shape $(B, C, H, 8, W, 8)$ [S1, unofficial].
   - Initialize the schedule: Compute `ddim_timesteps` using `make_ddim_timesteps` (default discretization: uniform) [S2, unofficial].
   - Calculate derived buffers (`sqrt_alphas_cumprod`, etc.) from the model's DDPM parameters [S2, unofficial].
   - Set the scheduler timesteps: `scheduler.set_timesteps(num_inference_steps)` [S1, unofficial].

2. **Conditioning Preparation:**
   - Concatenate positive and negative embeddings for CFG:
     $$ \text{embeddings} = \text{concat}([\text{embeddings}_{pos}, \text{embeddings}_{neg}]) $$
     where each is expanded to batch size $B$ [S1, unofficial].
   - If `unconditional_guidance_scale` > 1, prepare conditional and unconditional inputs for the model [S2, unofficial].

3. **Iteration Loop:**
   - Iterate through `timesteps` in reverse order (from $T$ to $0$) [S2, unofficial]. Let $i$ be the index in the reversed list, and $t$ be the current timestep value [S2, unofficial].
   - **Dynamic Guidance (Optional):** If `ucg_schedule` is provided, set `unconditional_guidance_scale` to `ucg_schedule[i]` [S2, unofficial].
   - **Inpainting Mask (Optional):** If `mask` is not None, assert $x_0$ is provided [S2, unofficial]. Compute $img_{orig} = q\_sample(x_0, t)$ [S2, unofficial]. Update current image: $img = img_{orig} \cdot mask + (1 - mask) \cdot img$ [S2, unofficial].
   - **Model Prediction:**
     - Duplicate latents for CFG: $x_{in} = \text{cat}([x_t, x_t])$ [S2, unofficial].
     - Duplicate timesteps: $t_{in} = \text{cat}([t, t])$ [S2, unofficial].
     - Pass through UNet to get predictions for unconditional and conditional branches:
       $$ [e_{t, uncond}, e_{t, cond}] = \text{chunk}(\text{model}(x_{in}, t_{in}, c_{in}), 2) $$
     - Apply Classifier-Free Guidance:
       $$ e_t = e_{t, uncond} + \text{unconditional\_guidance\_scale} \cdot (e_{t, cond} - e_{t, uncond}) $$
     - If `parameterization == "v"`, convert model output to noise prediction:
       $$ e_t = \text{model.predict\_eps\_from\_z\_and\_v}(x_t, t, \text{model\_output}) $$
       Otherwise, $e_t$ is the direct noise prediction [S2, unofficial].
   - **Score Correction (Optional):** If `score_corrector` is not None, modify $e_t$ (requires `parameterization == "eps"`) [S2, unofficial].
   - **Select Timestep Parameters:** Retrieve $a_t, a_{prev}, \sigma_t, \sqrt{1-a_t}$ from the precomputed buffers at index $i$ [S2, unofficial].
   - **Predict Clean Image ($\hat{x}_0$):**
     - If `parameterization == "eps"`:
       $$ \hat{x}_0 = \frac{x_t - \sqrt{1-a_t} \cdot e_t}{\sqrt{a_t}} $$
     - If `parameterization == "v"`:
       $$ \hat{x}_0 = \text{model.predict\_start\_from\_z\_and\_v}(x_t, t, \text{model\_output}) $$
     - If `quantize_denoised` is True, quantize $\hat{x}_0$ via the first stage model [S2, unofficial].
   - **Compute Direction to $x_{t-1}$:**
     $$ \text{dir\_xt} = \sqrt{1 - a_{prev} - \sigma_t^2} \cdot e_t $$
   - **Sample Noise:**
     $$ \text{noise} = \sigma_t \cdot \mathcal{N}(0, I) \cdot \text{temperature} $$
     Apply dropout if `noise_dropout` > 0 [S2, unofficial].
   - **Update Latent ($x_{t-1}$):**
     $$ x_{t-1} = \sqrt{a_{prev}} \cdot \hat{x}_0 + \text{dir\_xt} + \text{noise} $$
   - **Record Intermediates:** If $i \% \text{log\_every\_t} == 0$ or $i$ is the last step, append $x_{t-1}$ and $\hat{x}_0$ to intermediates [S2, unofficial].

4. **Final Output:**
   - Return the final $x_0$ (or $x_{t-1}$ after the last step).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def ddim_sample_with_cfg(model, scheduler, x_T, c, timesteps, unconditional_guidance_scale=7.5, 
                         ddim_eta=0.0, parameterization="eps", temperature=1.0, noise_dropout=0.0,
                         mask=None, x0=None, ucg_schedule=None):
    """
    Perform DDIM sampling with Classifier-Free Guidance.
    
    Args:
        model: UNet model with apply_model method.
        scheduler: Scheduler object with precomputed buffers (alphas, sigmas, etc.).
        x_T: Initial latent tensor (B, C, H, W).
        c: Conditional embedding tensor (B, D).
        timesteps: List of timesteps to traverse (reversed order).
        unconditional_guidance_scale: CFG scale.
        ddim_eta: Stochasticity parameter.
        parameterization: "eps" or "v".
        temperature: Noise scaling factor.
        noise_dropout: Probability of zeroing noise.
        mask: Optional inpainting mask.
        x0: Optional original image for inpainting.
        ucg_schedule: Optional list of guidance scales per step.
        
    Returns:
        x_0: Final denoised latent.
    """
    device = x_T.device
    b = x_T.shape[0]
    
    x = x_T.clone()
    intermediates = {'x_inter': [], 'pred_x0': []}
    
    # 1. Prepare conditioning for CFG
    c_in = torch.cat([c, c]) # [2B, D]
    
    for i, t in enumerate(timesteps):
        ts = torch.full((b,), t, device=device, dtype=torch.long)
        
        # Dynamic guidance scale
        if ucg_schedule is not None:
            current_guidance = ucg_schedule[i]
        else:
            current_guidance = unconditional_guidance_scale
            
        # Inpainting mask application
        if mask is not None:
            assert x0 is not None, "x0 must be provided for inpainting"
            # q_sample adds noise to x0 at timestep t
            sqrt_alpha = scheduler.sqrt_alphas_cumprod[t]
            sqrt_one_minus_alpha = scheduler.sqrt_one_minus_alphas_cumprod[t]
            noise_x0 = torch.randn_like(x0)
            img_orig = sqrt_alpha * x0 + sqrt_one_minus_alpha * noise_x0
            x = img_orig * mask + (1. - mask) * x
            
        # 2. Model Prediction with CFG
        x_in = torch.cat([x, x]) # Duplicate for unconditional/conditional
        t_in = torch.cat([ts, ts])
        
        # Get raw predictions for unconditional and conditional branches
        model_output = model.apply_model(x_in, t_in, c_in)
        noise_pred_uncond, noise_pred_cond = torch.chunk(model_output, 2)
        
        # Apply CFG formula
        noise_pred = noise_pred_uncond + current_guidance * (noise_pred_cond - noise_pred_uncond)
        
        # Handle v-parameterization
        if parameterization == "v":
            noise_pred = model.predict_eps_from_z_and_v(x, ts, model_output)
            
        # 3. Select timestep parameters
        a_t = scheduler.alphas[i]
        a_prev = scheduler.alphas_prev[i]
        sigma_t = scheduler.sigmas[i]
        sqrt_one_minus_at = scheduler.sqrt_one_minus_alphas[i]
        
        # 4. Predict x0
        if parameterization == "eps":
            pred_x0 = (x - sqrt_one_minus_at * noise_pred) / a_t.sqrt()
        else: # v
            pred_x0 = model.predict_start_from_z_and_v(x, ts, model_output)
            
        # Quantize if needed
        if getattr(scheduler, 'quantize_denoised', False):
            pred_x0 = model.first_stage_model.quantize(pred_x0)
            
        # 5. Compute direction
        dir_xt = (1. - a_prev - sigma_t**2).sqrt() * noise_pred
        
        # 6. Sample noise
        noise = sigma_t * torch.randn_like(x) * temperature
        if noise_dropout > 0:
            mask_noise = torch.bernoulli(torch.ones_like(noise) * (1 - noise_dropout))
            noise = noise * mask_noise
            
        # 7. Update x
        x_prev = a_prev.sqrt() * pred_x0 + dir_xt + noise
        
        # 8. Record intermediates
        if i % 10 == 0 or i == len(timesteps) - 1:
            intermediates['x_inter'].append(x_prev)
            intermediates['pred_x0'].append(pred_x0)
            
        x = x_prev
        
    return x, intermediates
```

**Parameters:**
- `num_inference_steps`: Number of sampling steps. Default: 50 [S1, unofficial].
- `unconditional_guidance_scale`: CFG scale. Default: 1.0 (disables guidance). Typical values: 7.5 for sampling, 100 for refinement [S1, S2, unofficial].
- `ddim_eta`: Controls stochasticity. Default: 0.0 [S2, unofficial].
- `temperature`: Scales injected noise. Default: 1.0 [S2, unofficial].
- `noise_dropout`: Dropout probability for noise. Default: 0.0 [S2, unofficial].
- `quantize_denoised`: Quantize $\hat{x}_0$. Default: False [S2, unofficial].
- `use_original_steps`: Use DDPM timesteps instead of DDIM timesteps. Default: False [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Masking Requires $x_0$:** Inpainting requires the original image $x_0$ to compute the noisy version via $q\_sample$ [S2, unofficial].
- **Dynamic Thresholding:** Not implemented; raises `NotImplementedError` if enabled [S2, unofficial].
- **Score Corrector:** Only supports `eps` parameterization; raises assertion error otherwise [S2, unofficial].
- **UCG Schedule Length:** Must exactly match the length of `time_range` [S2, unofficial].

## Subtopics

### NeRF Interpolation via DDIM and SLERP
A procedure to generate smooth transitions between 3D samples by interpolating in the diffusion latent space before deterministic decoding [S3]. It samples two initial latents $x^{(T)} \sim \mathcal{N}(0, I)$, interpolates them using spherical linear interpolation (SLERP), and passes them through a deterministic solver [S3]. Standard Gaussian diffusion models often yield non-smooth interpolation; however, models trained with early stopping for sparse-view reconstruction (as in SSDNeRF) preserve a smoother prior, yielding better generalization and smoother transitions than long unconditional training schedules [S3]. Used in SSDNeRF for interpolating generated 3D samples [S3].

### DDIM Stochastic Encode
Adds noise to an initial image $x_0$ according to the diffusion process at timestep $t$, using either DDPM or DDIM schedule parameters [S2, unofficial]. It selects schedule parameters based on `use_original_steps` (DDPM vs DDIM alphas), generates noise, and computes the noisy latent: $x_t = \sqrt{\alpha_t} x_0 + \sqrt{1-\alpha_t} \epsilon$ [S2, unofficial].

### DDIM Decode
Decodes a latent representation $x_{latent}$ into a sample by running the DDIM sampling process from a starting timestep $t_{start}$ down to 0 [S2, unofficial]. It selects timesteps (DDPM or DDIM based on `use_original_steps`), reverses them, and iteratively calls `p_sample_ddim` to update the latent [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot T)$, where $N$ is the number of sampling steps and $T$ is the cost of one UNet forward pass.
- **Memory Complexity:** $O(C \cdot H \cdot W)$ for latents, plus $O(T)$ for precomputed schedule buffers [S2, unofficial].
- **Measured Performance:** With `num_inference_steps=50` and `guidance_scale=7.5`, it produces high-quality images suitable for NeRF refinement [S1, unofficial].
- **Strengths:** Deterministic sampling (when $\eta=0$) allows for reproducible results and fewer steps. CFG enhances conditional fidelity [S2, unofficial].
- **Limitations:** Stochasticity ($\eta > 0$) can introduce variance. Inpainting requires $x_0$. Dynamic thresholding is not supported [S2, unofficial].

## Background
DDIM (Denoising Diffusion Implicit Models) was introduced as a deterministic sampling method for diffusion models that skips intermediate noise steps, allowing for fewer sampling steps than DDPM while maintaining sample quality [S2, unofficial]. It has been adapted for classifier-free guidance to enhance conditional generation [S1, unofficial]. In the context of volume rendering, it is used in frameworks like GaussianObject for refining 3D Gaussians and in SSDNeRF for 3D interpolation and generation [S2, unofficial][S3].

## Sources
[S1] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S2] chensjtu/GaussianObject, https://github.com/chensjtu/GaussianObject, fetched 2026-10-27 (unofficial)
[S3] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
