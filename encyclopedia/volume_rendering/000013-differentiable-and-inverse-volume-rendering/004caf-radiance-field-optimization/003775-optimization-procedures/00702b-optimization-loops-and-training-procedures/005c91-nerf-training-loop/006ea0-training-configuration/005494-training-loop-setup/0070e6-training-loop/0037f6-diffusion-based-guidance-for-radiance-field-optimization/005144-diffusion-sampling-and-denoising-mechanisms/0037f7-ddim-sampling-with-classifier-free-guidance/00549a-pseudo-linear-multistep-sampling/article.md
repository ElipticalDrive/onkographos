# Pseudo Linear Multistep Sampling

Also called: Pseudo Improved Euler, Pseudo Linear Multistep

## Overview
Pseudo Linear Multistep Sampling (PLMS) is a sampling method for diffusion models that uses a linear multistep predictor (Adams-Bashforth) to estimate the noise residual at the next timestep, allowing for fewer function evaluations per step compared to standard Euler or Heun methods, while maintaining stability similar to DDIM [S1, unofficial]. It builds on [DDIM Sampling with Classifier-Free Guidance](8118-ddim-sampling-with-classifier-free-guidance.md) by integrating classifier-free guidance into the noise prediction step [S1, unofficial].

## How it works
**Inputs:**
- $x_T$: Initial noisy latent (or `None` to sample from standard normal) [S1, unofficial].
- `cond`: Conditioning tensor [S1, unofficial].
- `unconditional_conditioning`: Unconditional conditioning tensor for classifier-free guidance [S1, unofficial].
- `unconditional_guidance_scale`: Scalar for guidance strength [S1, unofficial].
- `timesteps`: Sequence of timesteps to traverse [S1, unofficial].
- `ddim_use_original_steps`: Boolean flag determining whether to use the original DDPM timesteps (`self.ddpm_num_timesteps`) or the subsampled DDIM timesteps (`self.ddim_timesteps`) [S1, unofficial].
- `model`: The noise prediction model [S1, unofficial].
- `alphas`, `alphas_prev`, `sigmas`, `sqrt_one_minus_alphas`: Precomputed schedule parameters corresponding to the timesteps [S1, unofficial].
- `mask`, `x0`: Optional tensors for inpainting/masking [S1, unofficial].
- `temperature`: Scalar multiplier for the noise term [S1, unofficial].
- `log_every_t`: Interval for logging intermediate states [S1, unofficial].

**Outputs:**
- `x_0`: The generated sample [S1, unofficial].
- `intermediates`: Dictionary containing intermediate latents (`x_inter`) and predicted clean samples (`pred_x0`) [S1, unofficial].

**Data Structures:**
- `old_eps`: A list storing the last 3–4 noise residuals ($e_t$) to compute the multistep predictor [S1, unofficial].
- `intermediates`: Dictionary with keys `x_inter` (list of latents) and `pred_x0` (list of predicted clean samples) [S1, unofficial].

**Steps:**

1. **Initialization:**
   - If $x_T$ is `None`, set $img = \mathcal{N}(0, I)$ with shape `shape` on `device` [G].
   - If `timesteps` is `None`:
     - If `ddim_use_original_steps` is true, use `self.ddpm_num_timesteps` steps.
     - Else, use `self.ddim_timesteps` [S1, unofficial].
   - If `timesteps` is provided, subset it:
     - $subset\_end = \lfloor \min(\frac{timesteps}{len(self.ddim\_timesteps)}, 1) \times len(self.ddim\_timesteps) \rfloor - 1$ [S1, unofficial].
     - $timesteps = self.ddim\_timesteps[:subset\_end]$ [S1, unofficial].
   - Determine `time_range`:
     - If `ddim_use_original_steps` is true, `time_range = list(reversed(range(0, timesteps)))` [S1, unofficial].
     - Else, `time_range = np.flip(timesteps)` [S1, unofficial].
   - $total\_steps = len(time\_range)$ [S1, unofficial].
   - Initialize `old_eps = []` [S1, unofficial].
   - Initialize `intermediates = {'x_inter': [img], 'pred_x0': [img]}` [S1, unofficial].

2. **Iteration Loop:**
   - Iterate $i$ from $0$ to $total\_steps - 1$. Let $step = time\_range[i]$ [S1, unofficial].
   - Calculate $index = total\_steps - i - 1$ [S1, unofficial].
   - Create $ts = \text{tensor}([step] \times batch\_size)$ on `device` with dtype `long` [S1, unofficial].
   - Create $ts\_next = \text{tensor}([time\_range[\min(i + 1, len(time\_range) - 1)]] \times batch\_size)$ on `device` with dtype `long` [S1, unofficial].

3. **Masking (Optional):**
   - If `mask` is not `None`:
     - Assert `x0` is not `None` [S1, unofficial].
     - Compute $img\_orig = q\_sample(x0, ts)$ (forward diffusion process) [S1, unofficial].
     - Update $img = img\_orig \times mask + (1 - mask) \times img$ [S1, unofficial].

4. **Noise Prediction and History Update:**
   - Call `p_sample_plms(img, cond, ts, index, ...)` to obtain $(img, pred\_x0, e_t)$ [S1, unofficial].
   - Append $e_t$ to `old_eps` [S1, unofficial].
   - If $len(old\_eps) \ge 4$, remove the oldest element (`old_eps.pop(0)`) [S1, unofficial].

5. **Logging:**
   - If $index \pmod{log\_every\_t} == 0$ or $index == total\_steps - 1$:
     - Append $img$ to `intermediates['x_inter']` [S1, unofficial].
     - Append $pred\_x0$ to `intermediates['pred_x0']` [S1, unofficial].

6. **Inside `p_sample_plms`:**
   - Compute $e_t = \text{model\_output}(x, t)$ [S1, unofficial].
   - If `unconditional_conditioning` is not `None` and `unconditional_guidance_scale` $\neq 1.0$:
     - $x\_in = \text{concat}([x] \times 2)$
     - $t\_in = \text{concat}([t] \times 2)$
     - $c\_in = \text{concat}([unconditional\_conditioning, c])$
     - $e\_t\_uncond, e\_t = \text{model}(x\_in, t\_in, c\_in).chunk(2)$
     - $e_t = e\_t\_uncond + unconditional\_guidance\_scale \times (e_t - e\_t\_uncond)$ [S1, unofficial].

7. **Predictor Step (Adams-Bashforth):**
   - Compute $e_t'$ based on the length of `old_eps`:
     - If $len(old\_eps) == 0$:
       - $e_t' = \frac{e_t + e_{t+1}}{2}$ (Pseudo Improved Euler) [S1, unofficial].
     - If $len(old\_eps) == 1$:
       - $e_t' = \frac{3 e_t - old\_eps[-1]}{2}$ (2nd order Adams-Bashforth) [S1, unofficial].
     - If $len(old\_eps) == 2$:
       - $e_t' = \frac{23 e_t - 16 \cdot old\_eps[-1] + 5 \cdot old\_eps[-2]}{12}$ (3rd order Adams-Bashforth) [S1, unofficial].
     - If $len(old\_eps) \ge 3$:
       - $e_t' = \frac{55 e_t - 59 \cdot old\_eps[-1] + 37 \cdot old\_eps[-2] - 9 \cdot old\_eps[-3]}{24}$ (4th order Adams-Bashforth) [S1, unofficial].

8. **Reverse Diffusion Step:**
   - Retrieve parameters: $a_t = alphas[index]$, $a_{prev} = alphas\_prev[index]$, $\sigma_t = sigmas[index]$, $\sqrt{1-a_t} = \sqrt{1-alphas}[index]$ [S1, unofficial].
   - Compute predicted clean sample:
     - $pred\_x0 = \frac{x - \sqrt{1-a_t} \cdot e_t'}{\sqrt{a_t}}$ [S1, unofficial].
   - Compute deterministic direction:
     - $dir\_xt = \sqrt{1 - a_{prev} - \sigma_t^2} \cdot e_t'$ [S1, unofficial].
   - Compute noise:
     - $noise = \sigma_t \cdot \mathcal{N}(0, I) \cdot temperature$ [S1, unofficial].
     - Apply noise dropout with probability `noise_dropout` [G].
   - Compute previous latent:
     - $x_{prev} = \sqrt{a_{prev}} \cdot pred\_x0 + dir\_xt + noise$ [S1, unofficial].
   - Return $(x_{prev}, pred\_x0, e_t)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plms_sample(img, cond, unconditional_conditioning, unconditional_guidance_scale,
                timesteps, ddim_use_original_steps, alphas, alphas_prev, sigmas,
                sqrt_one_minus_alphas, mask, x0, temperature, log_every_t,
                model, device, batch_size):
    # 1. Initialization
    if img is None:
        img = torch.randn(shape, device=device)
    if timesteps is None:
        if ddim_use_original_steps:
            timesteps = self.ddpm_num_timesteps
        else:
            timesteps = self.ddim_timesteps
    else:
        subset_end = int(min(timesteps / len(self.ddim_timesteps), 1) * len(self.ddim_timesteps)) - 1
        timesteps = self.ddim_timesteps[:subset_end]

    if ddim_use_original_steps:
        time_range = list(reversed(range(0, timesteps)))
    else:
        time_range = np.flip(timesteps)
    total_steps = len(time_range)
    old_eps = []
    intermediates = {'x_inter': [img], 'pred_x0': [img]}

    # 2. Iteration Loop
    for i, step in enumerate(time_range):
        index = total_steps - i - 1
        ts = torch.full((batch_size,), step, device=device, dtype=torch.long)
        ts_next = torch.full((batch_size,), time_range[min(i + 1, len(time_range) - 1)], device=device, dtype=torch.long)

        # 3. Masking
        if mask is not None:
            assert x0 is not None
            img_orig = q_sample(x0, ts)
            img = img_orig * mask + (1. - mask) * img

        # 4. Noise Prediction & History Update
        img, pred_x0, e_t = p_sample_plms(img, cond, ts, index, unconditional_conditioning,
                                          unconditional_guidance_scale, model)
        old_eps.append(e_t)
        if len(old_eps) >= 4:
            old_eps.pop(0)

        # 5. Logging
        if index % log_every_t == 0 or index == total_steps - 1:
            intermediates['x_inter'].append(img)
            intermediates['pred_x0'].append(pred_x0)

    return img, intermediates

def p_sample_plms(x, cond, t, index, unconditional_conditioning, unconditional_guidance_scale, model):
    # 6. Inside p_sample_plms
    e_t = get_model_output(x, t)
    if unconditional_conditioning is not None and unconditional_guidance_scale != 1.0:
        x_in = torch.cat([x] * 2)
        t_in = torch.cat([t] * 2)
        c_in = torch.cat([unconditional_conditioning, cond])
        e_t_uncond, e_t = model(x_in, t_in, c_in).chunk(2)
        e_t = e_t_uncond + unconditional_guidance_scale * (e_t - e_t_uncond)

    # 7. Predictor Step
    if len(old_eps) == 0:
        e_t_next = e_t
        e_t_prime = (e_t + e_t_next) / 2
    elif len(old_eps) == 1:
        e_t_prime = (3 * e_t - old_eps[-1]) / 2
    elif len(old_eps) == 2:
        e_t_prime = (23 * e_t - 16 * old_eps[-1] + 5 * old_eps[-2]) / 12
    else:
        e_t_prime = (55 * e_t - 59 * old_eps[-1] + 37 * old_eps[-2] - 9 * old_eps[-3]) / 24

    # 8. Reverse Diffusion Step
    a_t = alphas[index]
    a_prev = alphas_prev[index]
    sigma_t = sigmas[index]
    sqrt_one_minus_at = sqrt_one_minus_alphas[index]

    pred_x0 = (x - sqrt_one_minus_at * e_t_prime) / a_t.sqrt()
    dir_xt = (1. - a_prev - sigma_t**2).sqrt() * e_t_prime
    noise = sigma_t * torch.randn_like(x) * temperature
    x_prev = a_prev.sqrt() * pred_x0 + dir_xt + noise

    return x_prev, pred_x0, e_t
```

**Parameters:**
- `ddim_use_original_steps`: Boolean. Default: `False` (uses subsampled DDIM timesteps) [S1, unofficial].
- `unconditional_guidance_scale`: Scalar. Default: `1.0` (no guidance) [S1, unofficial].
- `quantize_denoised`: Boolean. If true, quantize `pred_x0` via `model.first_stage_model.quantize(pred_x0)` [S1, unofficial].
- `dynamic_threshold`: Scalar or `None`. If not `None`, applies norm thresholding to `pred_x0` [S1, unofficial].
- `temperature`: Scalar. Default: `1.0` [S1, unofficial].
- `noise_dropout`: Probability $p$. Default: `0.0` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The Adams-Bashforth coefficients can lead to instability if the noise residuals vary rapidly. Ensure `old_eps` is populated correctly [S1, unofficial].
- **Boundaries:** At the last step ($index=0$), ensure `ts_next` and `alphas_prev` are handled correctly (often $a_{prev}=1, \sigma=0$) [G].
- **Masking:** If `mask` is used, `x0` must be provided. The masking operation must be applied before the noise prediction [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths and Limitations:** Not given in the sources.
- **When to use/not:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] chensjtu/GaussianObject, https://github.com/chensjtu/GaussianObject, fetched 2026-10-27 (unofficial)
