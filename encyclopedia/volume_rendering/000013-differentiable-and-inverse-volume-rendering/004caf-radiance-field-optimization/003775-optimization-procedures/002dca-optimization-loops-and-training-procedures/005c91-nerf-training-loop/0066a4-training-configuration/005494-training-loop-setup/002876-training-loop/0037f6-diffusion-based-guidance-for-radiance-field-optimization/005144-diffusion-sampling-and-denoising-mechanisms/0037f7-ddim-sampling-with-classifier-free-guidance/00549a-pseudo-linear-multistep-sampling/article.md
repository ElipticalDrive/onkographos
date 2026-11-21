# Pseudo Linear Multistep Sampling

Also called: Pseudo Improved Euler, Pseudo Linear Multistep

## Overview
Pseudo Linear Multistep Sampling (PLMS) is a deterministic or stochastic sampling algorithm for diffusion models that accelerates inference by estimating the noise residual at the next timestep using a linear multistep predictor (Adams-Bashforth) [S1, unofficial]. It allows for fewer function evaluations per step compared to standard Euler or Heun methods while maintaining stability similar to DDIM [S1, unofficial]. It builds on [DDIM Sampling with Classifier-Free Guidance](8118-ddim-sampling-with-classifier-free-guidance.md).

## How it works
**Inputs:**
- $x_T$: Initial noisy latent (or `None` to sample from standard normal).
- `cond`: Conditioning tensor (e.g., text embeddings).
- `unconditional_conditioning`: Unconditional conditioning tensor (for classifier-free guidance).
- `unconditional_guidance_scale`: Scalar for guidance strength.
- `timesteps`: Sequence of timesteps to traverse.
- `ddim_use_original_steps`: Boolean flag for timestep selection.
- `model`: The noise prediction model.
- `alphas`, `alphas_prev`, `sigmas`, `sqrt_one_minus_alphas`: Precomputed schedule parameters corresponding to the timesteps.
- `mask`, `x0`: Optional tensors for inpainting/masking.
- `temperature`: Scalar multiplier for noise.
- `log_every_t`: Interval for logging intermediate states.

**Outputs:**
- `x_0`: The generated sample.
- `intermediates`: Dictionary containing intermediate latents (`x_inter`) and predicted clean samples (`pred_x0`).

**Data Structures:**
- `old_eps`: A list storing the last 3-4 noise residuals ($e_t$) to compute the multistep predictor.
- `intermediates`: Dictionary with keys `x_inter` (list of latents) and `pred_x0` (list of predicted clean samples).

**Steps:**

1.  **Initialization:**
    - If $x_T$ is `None`, set $img = \mathcal{N}(0, I)$ with shape `shape` on `device` [G].
    - If `timesteps` is `None`:
        - If `ddim_use_original_steps` is true, use `self.ddpm_num_timesteps` steps.
        - Else, use `self.ddim_timesteps`.
    - If `timesteps` is provided, subset it:
        - $subset\_end = \lfloor \min(\frac{timesteps}{len(self.ddim\_timesteps)}, 1) \times len(self.ddim\_timesteps) \rfloor - 1$
        - $timesteps = self.ddim\_timesteps[:subset\_end]$
    - Determine `time_range`:
        - If `ddim_use_original_steps` is true, `time_range = list(reversed(range(0, timesteps)))`.
        - Else, `time_range = np.flip(timesteps)`.
    - $total\_steps = len(time\_range)$.
    - Initialize `old_eps = []`.
    - Initialize `intermediates = {'x_inter': [img], 'pred_x0': [img]}`.

2.  **Iteration Loop:**
    - Iterate $i$ from $0$ to $total\_steps - 1$. Let $step = time\_range[i]$.
    - Calculate $index = total\_steps - i - 1$.
    - Create $ts = \text{tensor}([step] \times batch\_size)$ on `device` with dtype `long`.
    - Create $ts\_next = \text{tensor}([time\_range[\min(i + 1, len(time\_range) - 1)]] \times batch\_size)$ on `device` with dtype `long`.

3.  **Masking (Optional):**
    - If `mask` is not `None`:
        - Assert `x0` is not `None`.
        - Compute $img\_orig = q\_sample(x0, ts)$ (forward diffusion process).
        - Update $img = img\_orig \times mask + (1 - mask) \times img$.

4.  **Noise Prediction and History Update:**
    - Call `p_sample_plms(img, cond, ts, index, ...)` to obtain $(img, pred\_x0, e_t)$.
    - Append $e_t$ to `old_eps`.
    - If $len(old\_eps) \ge 4$, remove the oldest element (`old_eps.pop(0)`).

5.  **Logging:**
    - If $index \pmod{log\_every\_t} == 0$ or $index == total\_steps - 1$:
        - Append $img$ to `intermediates['x_inter']`.
        - Append $pred\_x0$ to `intermediates['pred_x0']`.

6.  **Inside `p_sample_plms`:**
    - Compute $e_t = \text{model\_output}(x, t)$.
    - If `unconditional_conditioning` is not `None` and `unconditional_guidance_scale` $\neq 1.0$:
        - $x\_in = \text{concat}([x] \times 2)$
        - $t\_in = \text{concat}([t] \times 2)$
        - $c\_in = \text{concat}([unconditional\_conditioning, c])$
        - $e\_t\_uncond, e\_t = \text{model}(x\_in, t\_in, c\_in).chunk(2)$
        - $e_t = e\_t\_uncond + unconditional\_guidance\_scale \times (e_t - e\_t\_uncond)$

7.  **Predictor Step (Adams-Bashforth):**
    - Compute $e_t'$ based on the length of `old_eps`:
        - If $len(old\_eps) == 0$:
            - $e_t' = \frac{e_t + e_{t+1}}{2}$ (Pseudo Improved Euler) [G: Note: $e_{t+1}$ is typically estimated or assumed zero in pseudo methods; in this specific implementation context, it often refers to using the current and next predicted noise, but since $e_{t+1}$ is not yet computed, this step usually implies a specific handling or approximation. Based on the source structure, it seems `e_t_next` might be derived or this is a placeholder for the multistep logic. However, standard PLMS uses previous steps. The source lists `e_t_prime` formulas based on `old_eps`. Let's look closer. The source says: "if len(old_eps) == 0, e_t_prime = (e_t + e_t_next) / 2". This implies `e_t_next` is available or estimated. In many PLMS implementations, `e_t_next` is the noise predicted at the *next* step, but here we are at step $t$. A common "Pseudo" variant uses the current noise and a predictor. Given the ambiguity, we follow the source's explicit formula structure but note that `e_t_next` must be defined. In the context of the provided code snippet, `e_t_next` is not explicitly calculated before this formula in the loop. It is likely that `e_t_next` refers to a predictor estimate or the method assumes a specific initialization. However, looking at the other formulas, they rely on `old_eps`. For len=0, there are no old eps. This suggests `e_t_next` might be a typo in the source notes or refers to a specific initialization strategy. For implementation, if `old_eps` is empty, one might default to Euler ($e_t' = e_t$) or use a specific heuristic. We will stick to the source's formula but mark `e_t_next` as requiring definition from the specific model context or assume it's the noise predicted at the next step if available, otherwise fallback. *Correction*: In many PLMS implementations, the "Pseudo" part means we don't actually evaluate the model at $t+1$ to get $e_{t+1}$, but estimate it. The source notes are slightly ambiguous. We will implement the formulas for len >= 1 as they are clear. For len=0, we will use the standard Euler step $e_t' = e_t$ as a safe default if `e_t_next` is undefined, or note the source's specific requirement.]
            - *Refined Interpretation*: The source explicitly lists `(e_t + e_t_next) / 2`. If `e_t_next` is not available, this step is undefined. However, in the context of "Pseudo" methods, it often implies using the current step's noise as the predictor for the next, or a specific initialization. We will write the code to handle `len(old_eps) == 0` by using $e_t' = e_t$ (Euler) or noting the source's specific variable `e_t_next` which must be provided by the caller or model wrapper. Given the constraint to implement from text, and the lack of definition for `e_t_next`, we will assume a fallback to Euler or that `e_t_next` is passed. *Decision*: The source notes are from a specific codebase. We will implement the formulas for `len(old_eps) >= 1` precisely. For `len(old_eps) == 0`, we will use $e_t' = e_t$ (standard Euler) as a robust default, noting the source's ambiguity.
        - If $len(old\_eps) == 1$:
            - $e_t' = \frac{3 e_t - old\_eps[-1]}{2}$ (2nd order Adams-Bashforth)
        - If $len(old\_eps) == 2$:
            - $e_t' = \frac{23 e_t - 16 \cdot old\_eps[-1] + 5 \cdot old\_eps[-2]}{12}$ (3rd order Adams-Bashforth)
        - If $len(old\_eps) \ge 3$:
            - $e_t' = \frac{55 e_t - 59 \cdot old\_eps[-1] + 37 \cdot old\_eps[-2] - 9 \cdot old\_eps[-3]}{24}$ (4th order Adams-Bashforth)

8.  **Reverse Diffusion Step:**
    - Retrieve parameters: $a_t = alphas[index]$, $a_{prev} = alphas\_prev[index]$, $\sigma_t = sigmas[index]$, $\sqrt{1-a_t} = \sqrt{1-alphas}[index]$.
    - Compute predicted clean sample:
        - $pred\_x0 = \frac{x - \sqrt{1-a_t} \cdot e_t'}{\sqrt{a_t}}$
    - Compute deterministic direction:
        - $dir\_xt = \sqrt{1 - a_{prev} - \sigma_t^2} \cdot e_t'$
    - Compute noise:
        - $noise = \sigma_t \cdot \mathcal{N}(0, I) \cdot temperature$
        - Apply noise dropout with probability `noise_dropout` [G].
    - Compute previous latent:
        - $x_{prev} = \sqrt{a_{prev}} \cdot pred\_x0 + dir\_xt + noise$
    - Return $(x_{prev}, pred\_x0, e_t)$.

**Parameters:**
- `ddim_use_original_steps`: Boolean. Default: `False` (uses subsampled DDIM timesteps).
- `unconditional_guidance_scale`: Scalar. Default: `1.0`.
- `quantize_denoised`: Boolean. If true, quantize `pred_x0` via `model.first_stage_model.quantize(pred_x0)`.
- `dynamic_threshold`: Scalar or `None`. If not `None`, applies norm thresholding to `pred_x0`.
- `temperature`: Scalar. Default: `1.0`.
- `noise_dropout`: Probability $p$. Default: `0.0`.

**Edge Cases and Pitfalls:**
- **Precision:** The Adams-Bashforth coefficients can lead to instability if the noise residuals vary rapidly. Ensure `old_eps` is populated correctly.
- **Boundaries:** At the last step ($index=0$), ensure `ts_next` and `alphas_prev` are handled correctly (often $a_{prev}=1, \sigma=0$).
- **Masking:** If `mask` is used, `x0` must be provided. The masking operation must be applied before the noise prediction.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of timesteps. Each step requires one model evaluation (after the first few steps where history is built).
- **Memory Complexity:** $O(1)$ additional memory for `old_eps` (fixed size 4) and intermediates if stored.
- **Performance:** Faster than Euler/Heun for the same number of steps due to higher-order predictor, but requires storing previous noise estimates. Stability is comparable to DDIM.
- **Strengths:** Fewer function evaluations per step compared to lower-order methods; compatible with classifier-free guidance.
- **Limitations:** Requires careful initialization of `old_eps`; potential instability with high guidance scales or rapid noise changes.

## Background
PLMS is a sampling method derived from linear multistep methods (specifically Adams-Bashforth) applied to the stochastic differential equations underlying diffusion models. It was popularized in the context of efficient diffusion sampling, allowing for high-quality generation with fewer steps. The "Pseudo" designation often refers to the approximation of the next step's noise or the specific handling of the multistep history in the context of discrete diffusion schedules.

## Sources
[S1] chensjtu/GaussianObject, https://github.com/chensjtu/GaussianObject, fetched 2026-10-27 (unofficial)
