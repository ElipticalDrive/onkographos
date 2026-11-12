# Learning Rate Scheduling

Also called: Exponential Learning Rate Decay, LR Scheduling, LR decay, LambdaLR Scheduler, Learning Rate Decay, Learning Rate Scheduler

## Overview
Learning rate scheduling adjusts the optimization step size during radiance field training to balance convergence speed and reconstruction quality [S2, S3, S1, unofficial]. It applies a decay function to the learning rate(s) of the optimizer(s) based on the current training step, epoch, or specific training events [S2, unofficial]. What sets this method apart from its siblings is the variety of decay strategies employed, ranging from simple exponential power-of-2 decay [S2, unofficial] to log-linear interpolation with delay phases [S3, unofficial] and event-driven spikes for densification [S7, unofficial], allowing fine-grained control over parameter groups (e.g., positions vs. features) [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs:** Current step/iteration index ($i$ or $t$), maximum steps ($max\_steps$ or $maxSteps$), initial learning rate ($lr\_init$ or $lr\_0$), final learning rate ($lr\_final$ or $lr\_end$), and schedule-specific parameters (e.g., $gamma$, $step\_size$, $L$, $I$) [S3, S4, S5, S7, unofficial].
*   **Outputs:** The scaled learning rate value(s) for the current step.

### Data Structures
No specific global data structures are required beyond the optimizer's parameter groups. Some implementations maintain lists for dynamic events, such as `upsamp_list` and `N_voxel_list` for upsampling triggers [S8, unofficial].

### Numbered Steps and Formulas

#### 1. Log-Linear Decay with Delay [S3, unofficial]
This schedule decays log-linearly from `lr_init` to `lr_final` over `max_steps`, with an optional initial delay phase [S3, unofficial].
1.  **Check for early exit:** If `step < 0` or both `lr_init` and `lr_final` are 0.0, return 0.0 [S3, unofficial].
2.  **Calculate normalized progress:** Compute $t = \text{clip}(step / max\_steps, 0, 1)$ [S3, unofficial].
3.  **Calculate log-linear interpolation:**
    $$ log\_lerp = \exp( \log(lr\_init) \cdot (1 - t) + \log(lr\_final) \cdot t ) $$ [S3, unofficial].
4.  **Calculate delay multiplier:**
    *   If `lr_delay_steps > 0`:
        $$ delay\_rate = lr\_delay\_mult + (1 - lr\_delay\_mult) \cdot \sin\left(0.5 \cdot \pi \cdot \text{clip}\left(\frac{step}{lr\_delay\_steps}, 0, 1\right)\right) $$ [S3, unofficial].
    *   Else:
        $$ delay\_rate = 1.0 $$ [S3, unofficial].
5.  **Return result:**
    $$ lr = delay\_rate \cdot log\_lerp $$ [S3, unofficial].

**Parameters:** `lr_delay_steps` (default 0), `lr_delay_mult` (default 1.0), `max_steps` (default 1,000,000) [S3, unofficial].

#### 2. Exponential Decay via Power-of-2 [S2, unofficial]
Divides the learning rate by a power of 2 after each batch, dependent on batch count and epoch [S2, unofficial].
1.  **Calculate decay factor:**
    $$ factor = 2^{1 / (batchCount \cdot (epoch + 1))} $$ [S2, unofficial].
2.  **Update learning rate:**
    $$ LR_{new} = \frac{LR_{old}}{factor} $$ [S2, unofficial].
    This is applied separately to `configs->learningRate` and `configs->shLearningRate` [S2, unofficial].

#### 3. Linear Interpolation with Clamping [S5, unofficial]
A piecewise linear schedule defined by start/end steps and values [S5, unofficial].
1.  **Unpack parameters:** `start_step`, `start_value`, `end_value`, `end_step` [S5, unofficial].
2.  **Calculate progress:**
    $$ t = \max\left(\min\left(1.0, \frac{current\_step - start\_step}{end\_step - start\_step}\right), 0.0\right) $$ [S5, unofficial].
3.  **Interpolate:**
    $$ value = start\_value + (end\_value - start\_value) \cdot t $$ [S5, unofficial].
4.  **Handle list inputs:** If the value is not a scalar, parse it from config. If it is a list of length 3, prepend 0 to make it length 4 [S5, unofficial].

#### 4. LambdaLR Scheduler [S4, unofficial]
Defines a lambda function for the scheduler [S4, unofficial].
1.  **Define lambda function:**
    $$ lr\_lambda(epoch) = gamma^{(epoch / step\_size)} $$ [S4, unofficial].
2.  **Create Scheduler:**
    $$ lr\_scheduler = \text{torch.optim.lr\_scheduler.LambdaLR}(optimizer, lr\_lambda, last\_epoch=start\_epoch - 1) $$ [S4, unofficial].

**Parameters:** `lr_scheduler_gamma`, `lr_scheduler_step_size` [S4, unofficial].

#### 5. Gaussian Means Exponential Decay [S1, unofficial]
Schedules the learning rate for means (Adam group 0) exponentially [S1, unofficial].
1.  **Calculate normalized step:**
    $$ t = \text{clamp}\left(\frac{step}{maxSteps}, 0, 1\right) $$ [S1, unofficial].
2.  **Calculate LR:**
    $$ lr\_means(t) = \exp( \log(lr\_init) \cdot (1 - t) + \log(lr\_final) \cdot t ) $$ [S1, unofficial].

#### 6. Learning Rate Spike [S7, unofficial]
Increases the learning rate of specific parameters (instant NGP weights and vertex coordinates) after a densification event [S7, unofficial].
1.  **Identify event:** Let $I$ be the iteration of densification [S7, unofficial].
2.  **Calculate spike contribution:**
    $$ spike(i) = \mathbb{1}\{i - I > 0\} \cdot (l(0) - l(I)) \cdot \exp\left(-\frac{6(i - I)}{L}\right) $$ [S7, unofficial].
    Where $i$ is the current iteration, $l(0)$ is the initial learning rate, $l(I)$ is the learning rate at iteration $I$, and $L$ is the duration of the spike [S7, unofficial].
3.  **Update LR:**
    $$ lr_{new}(i) = lr_{decay}(i) + spike(i) $$ [S7, unofficial].
    This increases the decaying learning rate back up to its maximum (initial) value, before decaying it back down in $L$ iterations [S7, unofficial].

**Parameter:** $L$ (duration of the spike in iterations) [S7, unofficial].

#### 7. Learning Rate Reset on Upsampling [S8, unofficial]
Resets or scales the learning rate when the volume grid is upsampled [S8, unofficial].
1.  **Check for upsampling:** If current iteration is in `upsamp_list`:
    *   Pop target voxel count: $n\_voxels = N\_voxel\_list.pop(0)$ [S8, unofficial].
    *   Compute resolution: $reso\_cur = N\_to\_reso(n\_voxels, tensorf.aabb)$ [S8, unofficial].
    *   Limit samples: $nSamples = \min(args.nSamples, cal\_n\_samples(reso\_cur, args.step\_ratio))$ [S8, unofficial].
    *   Call `tensorf.upsample_volume_grid(reso_cur)` [S8, unofficial].
2.  **Adjust Learning Rate:**
    *   If `args.lr_upsample_reset` is true:
        $$ lr\_scale = 1 $$ [S8, unofficial].
    *   Else:
        $$ lr\_scale = args.lr\_decay\_target\_ratio^{(iteration / args.n\_iters)} $$ [S8, unofficial].
3.  **Reinitialize Optimizer:**
    *   Get optimizer groups: $grad\_vars = tensorf.get\_optparam\_groups(args.lr\_init \cdot lr\_scale, args.lr\_basis \cdot lr\_scale)$ [S8, unofficial].
    *   Reinitialize: $optimizer = \text{torch.optim.Adam}(grad\_vars, betas=(0.9, 0.99))$ [S8, unofficial].

### Pseudocode
The following Python-like pseudocode implements the log-linear decay with delay, as it is widely used in modern Gaussian Splatting implementations [S3, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def get_learning_rate(
    step: int,
    max_steps: int = 1000000,
    lr_init: float = 1.6e-4,
    lr_final: float = 1.6e-6,
    lr_delay_steps: int = 0,
    lr_delay_mult: float = 1.0
) -> float:
    """
    Calculates the learning rate using log-linear decay with an optional delay phase.
    Based on [S3, unofficial].
    """
    # Edge case: invalid step or zero rates
    if step < 0 or (lr_init == 0.0 and lr_final == 0.0):
        return 0.0

    # 1. Calculate normalized progress t
    t = np.clip(step / max_steps, 0, 1)

    # 2. Calculate log-linear interpolation
    log_lr_init = np.log(lr_init)
    log_lr_final = np.log(lr_final)
    log_lerp = np.exp(log_lr_init * (1 - t) + log_lr_final * t)

    # 3. Calculate delay multiplier
    if lr_delay_steps > 0:
        progress_in_delay = np.clip(step / lr_delay_steps, 0, 1)
        delay_rate = lr_delay_mult + (1 - lr_delay_mult) * np.sin(0.5 * np.pi * progress_in_delay)
    else:
        delay_rate = 1.0

    # 4. Return final learning rate
    return delay_rate * log_lerp
```

### Parameters with Typical Values
*   `lr_init`: $1.6 \times 10^{-4}$ [S3, unofficial].
*   `lr_final`: $1.6 \times 10^{-6}$ [S3, unofficial].
*   `max_steps`: 1,000,000 [S3, unofficial].
*   `lr_delay_steps`: 0 [S3, unofficial].
*   `lr_delay_mult`: 1.0 [S3, unofficial].

### Edge Cases and Pitfalls
*   **Zero Learning Rate:** If `lr_init` and `lr_final` are both 0.0, return 0.0 to avoid NaNs [S3, unofficial].
*   **Division by Zero in Delay:** If `lr_delay_steps` is 0, avoid division; use the `else` branch [S3, unofficial].
*   **Precision:** Log-linear decay involves exponentials and logarithms; ensure inputs are positive [S3, unofficial].
*   **Densification Timing:** For LR spikes, duration $L$ must be chosen carefully to balance adaptation and stability [S7, unofficial].
*   **Upsampling Reset:** If `lr_upsample_reset` is false, the learning rate continues to decay, which might be too low for the new grid; if true, it resets to initial, which might be too high [S8, unofficial].

## Subtopics

### [Hyperparameter Configuration for Stochastic Gradient Estimation](6748-hyperparameter-configuration-for-stochastic-gradient-estimat.md)
This variant configures learning rates for first-order (OurG, OurH) and second-order (OurHVP, OurHVPA) stochastic gradient estimators. It solves the problem of selecting optimal sampling, learning rate, and trust region parameters for inverse rendering tasks such as caustic optimization, shape reconstruction, and texture mapping. What sets this configuration apart from standard NeRF training is the explicit use of **Sigma annealing** for distribution bandwidth control and **antithetic sampling** to reduce variance, alongside distinct parameter sets for first-order versus second-order optimization loops.

### View-count-based Learning Rate
This variant adjusts learning rates based on a view-count prior [S6]. It solves the problem of overfitting to frequently viewed regions by down-weighting their learning rates [S6]. What sets this apart is the dynamic adjustment of LR based on observation frequency rather than time [S6]. A pitfall is that without the view-count prior, PSNR can degrade up to -1.22 in the worst case [S6].

### Learning Rate Spike
This variant increases the learning rate of specific model parameters (instant NGP weights and vertex coordinates) after a densification event to allow rapid accommodation for changes [S7, unofficial]. It solves the problem of slow convergence after structural changes in the representation [S7, unofficial]. What sets this apart is the use of an exponential spike function added to the decaying learning rate [S7, unofficial].

### Learning Rate Reset on Upsampling
This variant resets or scales the learning rate when the volume grid is upsampled [S8, unofficial]. It solves the problem of the learning rate being too low for the newly introduced high-frequency details [S8, unofficial]. What sets this apart is the explicit reinitialization of the optimizer with scaled learning rates upon grid expansion [S8, unofficial].

### Halving Training Iterations with Schedules
This variant compensates for reduced training iterations (e.g., 15K instead of 30K) by applying additional learning rate schedules for scale, rotation, and SH DC coefficients, and increasing the SH higher-order coefficient learning rate 5× to 0.000625 [S9]. It solves the problem of quality degradation at reduced iteration counts [S9]. What sets this apart is the assignment of higher LR values early during training while decaying them progressively to standard values [S9]. Training for 15K iterations with these schedules yields PSNR 26.47, LPIPS 0.236, and Opt Time 05:40, halving the training time compared to 30K iterations (10:24) with only a slight PSNR decrease (26.51 → 26.47) [S9].

### SH Degree Scheduling
This variant gradually increases the Spherical Harmonics (SH) degree during training to improve image quality, as higher degrees do not significantly contribute if used from the start [S10, unofficial]. It solves the problem of inefficient early training on high-degree SH coefficients [S10, unofficial]. What sets this apart is the staged increase: from 0 to 1 during coarse training, then gradually from 1 to the target degree $n$ during fine training, with each increase occurring after 10% of the total fine training iterations [S10, unofficial]. The default SH degree is 1, and the target degree $n$ is chosen based on scene requirements [S10, unofficial].

### Per-Parameter Learning Rate Scheduling
This variant adjusts learning rates for scaling, opacity, rotation, and feature parameters in each training iteration [S11, unofficial]. It solves the problem of suboptimal convergence for specific parameter types [S11, unofficial]. What sets this apart is the independent tuning of each parameter's LR, with scaling LR optimal values mainly located in $[10^{-3}, 10^{-6}]$ [S11, unofficial]. Setting feature LR to 0 does not significantly change speed or quality, indicating less impact on improvement [S11, unofficial].

### Piecewise Schedule
This variant composes multiple sub-schedules applied sequentially at specified milestone steps [S12, unofficial]. It solves the problem of needing complex, multi-phase training schedules [S12, unofficial]. What sets this apart is the use of a list of Schedule objects and a 1D array of cumulative milestone steps to route the current step to the correct sub-schedule [S12, unofficial]. The implementation uses `jnp.searchsorted` on milestones to find the active schedule and returns `schedule.get(step - base_idx)` [S12, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per step for standard schedules [S3, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory for schedule parameters [S3, unofficial].
*   **Scaling:** Schedules are generally independent of batch size, though step counts may scale with dataset size [S3, unofficial].
*   **Strengths:** Adapts to the changing loss landscape; prevents overshooting; handles structural changes like densification and upsampling [S7, S8, unofficial].
*   **Limitations:** Requires careful tuning of schedule parameters (e.g., $L$, $gamma$, $max\_steps$) [S7, unofficial][S9].
*   **When to use:** Standard practice for most radiance field optimizations [S3, unofficial][S9].
*   **When not to use:** If training is very short or the representation is static and simple, a constant learning rate might suffice [G].

## Background
Learning rate scheduling was adopted in volume rendering and NeRFs to handle the slow convergence and instability of training neural radiance fields. The log-linear decay with delay is implemented in InstantSplat [S3, unofficial]. The exponential decay schedule is used in Neural Harmonic Textures [S9]. The learning rate spike technique is introduced in Radiance Meshes [S7, unofficial]. The view-count-based LR is discussed in Direct Voxel Grid Optimization [S6].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S4] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S5] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S6] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S7] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S8] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S9] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S10] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S11] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S12] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
