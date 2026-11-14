# Learning Rate Scheduling

Also called: Exponential Learning Rate Decay, LR Scheduling, LR decay, LambdaLR Scheduler, Learning Rate Decay, Learning Rate Scheduler

## Overview
Learning rate scheduling adjusts the optimizer's step size during the training of radiance fields to balance convergence speed and stability. It prevents overshooting during fine-tuning phases and manages the trade-off between rapid initial geometric alignment and precise detail refinement. Unlike [Training Schedule](3154-training-schedule.md), which modifies the active set of losses and structural representation states, learning rate scheduling specifically controls the magnitude of parameter updates [G].

## How it works
The method defines a function or rule that maps the current training step (or epoch) to a scalar learning rate value. This value is applied to the optimizer's parameter groups. Common implementations include exponential decay, log-linear interpolation, and piecewise schedules.

### Inputs and Outputs
- **Inputs**: Current step index $i$, maximum steps $N$, initial learning rate $lr_{init}$, final learning rate $lr_{final}$, and schedule-specific parameters (e.g., delay steps, gamma).
- **Output**: A scalar learning rate $lr(i)$.

### Data Structures
- **Optimizer Groups**: Parameters are typically grouped (e.g., by type such as means, features, scales) to allow different learning rates for different components [S1, unofficial][S11, unofficial].
- **Schedule Config**: A configuration object or dictionary containing parameters like `lr_init`, `lr_final`, `max_steps`, `lr_delay_steps`, and `lr_delay_mult` [S3, unofficial].

### Numbered Steps

1. **Initialize Parameters**: Define $lr_{init}$, $lr_{final}$, and total steps $N$. For exponential schedules, define gamma $\gamma$ and step size $s$ [S4, unofficial]. For delay schedules, define $N_{delay}$ and multiplier $\mu$ [S3, unofficial].
2. **Calculate Normalized Progress**: Compute the normalized step $t = \text{clamp}(i / N, 0, 1)$ [S1, unofficial][S3, unofficial].
3. **Compute Base Learning Rate**:
   - **Exponential Decay**: Calculate $lr_{base} = lr_{init} \cdot \gamma^{(i/s)}$ [S4, unofficial].
   - **Log-Linear Decay**: Calculate $lr_{base} = \exp(\ln(lr_{init}) \cdot (1 - t) + \ln(lr_{final}) \cdot t)$ [S3, unofficial].
   - **Linear Decay**: Calculate $lr_{base} = lr_{init} + (lr_{final} - lr_{init}) \cdot t$ [S1, unofficial].
   - **Cosine Decay**: Apply cosine annealing from $lr_{init}$ to $lr_{final}$ over $N$ steps [S13].
4. **Apply Delay (Optional)**: If $i < N_{delay}$, apply a delay multiplier. Calculate $r = \text{clamp}(i / N_{delay}, 0, 1)$. Then $delay\_rate = \mu + (1 - \mu) \cdot \sin(0.5 \pi r)$ [S3, unofficial]. If $N_{delay} = 0$, $delay\_rate = 1.0$.
5. **Combine**: The final learning rate is $lr(i) = lr_{base} \cdot delay\_rate$ [S3, unofficial].
6. **Update Optimizer**: Set the learning rate for the relevant parameter groups in the optimizer to $lr(i)$ [S4, unofficial].

### Formulas

**Log-Linear Decay with Delay [S3, unofficial]:**
$$
t = \text{clamp}\left(\frac{i}{N}, 0, 1\right)
$$
$$
lr_{base} = \exp\left(\ln(lr_{init}) \cdot (1 - t) + \ln(lr_{final}) \cdot t\right)
$$
$$
r = \text{clamp}\left(\frac{i}{N_{delay}}, 0, 1\right)
$$
$$
delay\_rate = \mu + (1 - \mu) \cdot \sin\left(\frac{\pi}{2} r\right)
$$
$$
lr(i) = lr_{base} \cdot delay\_rate
$$

**Exponential Decay [S4, unofficial]:**
$$
lr(i) = lr_{init} \cdot \gamma^{\lfloor i / s \rfloor}
$$

**Piecewise Schedule [S12, unofficial]:**
$$
idx = \text{searchsorted}(milestones, i)
$$
$$
base\_idx = milestones[idx - 1] \text{ if } idx \ge 1 \text{ else } 0
$$
$$
lr(i) = schedules[idx].get(i - base\_idx)
$$

**Learning Rate Spike after Densification [S7, unofficial]:**
When densification occurs at iteration $I$, a spike is added to the decaying learning rate $l(i)$ for duration $L$:
$$
\text{Spike}(i) = \mathbb{1}\{i - I > 0\} \cdot (l(0) - l(I)) \cdot \exp\left(-\frac{6(i - I)}{L}\right)
$$
$$
lr_{total}(i) = l(i) + \text{Spike}(i)
$$

### Parameters
- **$lr_{init}$**: Initial learning rate. Typical values: $10^{-3}$ [S13], $10^{-2}$ [S11, unofficial].
- **$lr_{final}$**: Final learning rate. Typical values: $5 \times 10^{-5}$ [S13], $10^{-6}$ [S11, unofficial].
- **$N$**: Total number of optimization steps. Typical values: 100,000 to 30,000 [S3, unofficial][S9].
- **$N_{delay}$**: Number of steps for the initial delay phase. Default: 0 [S3, unofficial].
- **$\mu$**: Multiplier for the learning rate during the delay phase. Default: 1.0 [S3, unofficial].
- **$\gamma$**: Gamma value for exponential decay.
- **$s$**: Step size for exponential decay.
- **$L$**: Duration of the learning rate spike in iterations [S7, unofficial].

### Edge Cases and Pitfalls
- **Zero Learning Rate**: If $lr_{init}$ and $lr_{final}$ are both 0.0, return 0.0 [S3, unofficial].
- **View-Count Prior**: Without view-count based learning rate adjustments, PSNR can degrade significantly (up to -1.22) [S6].
- **Feature LR**: Setting feature learning rate to 0 does not significantly change speed or quality, indicating lower impact on improvement [S11, unofficial].
- **Scaling LR**: Optimal scaling learning rates are typically in the range $[10^{-3}, 10^{-6}]$ [S11, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def get_exponential_lr_func(lr_init, lr_final, max_steps, lr_delay_steps=0, lr_delay_mult=1.0):
    """
    Generates a learning rate function that decays log-linearly from lr_init to lr_final
    over max_steps, with an optional initial delay phase.
    """
    def lr_func(step):
        if step < 0:
            return 0.0
        if lr_init == 0.0 and lr_final == 0.0:
            return 0.0
        
        # Calculate normalized step t
        t = np.clip(step / max_steps, 0, 1)
        
        # Calculate log-linear interpolation
        log_lr_init = np.log(lr_init)
        log_lr_final = np.log(lr_final)
        log_lerp = np.exp(log_lr_init * (1 - t) + log_lr_final * t)
        
        # Calculate delay rate
        if lr_delay_steps > 0:
            delay_t = np.clip(step / lr_delay_steps, 0, 1)
            delay_rate = lr_delay_mult + (1 - lr_delay_mult) * np.sin(0.5 * np.pi * delay_t)
        else:
            delay_rate = 1.0
            
        return delay_rate * log_lerp
        
    return lr_func

def apply_lambda_lr_scheduler(optimizer, lr_lambda, last_epoch=-1):
    """
    Simulates torch.optim.lr_scheduler.LambdaLR behavior.
    """
    for param_group in optimizer.param_groups:
        # Assuming lr_lambda is a function of epoch/step
        # In practice, this might need to track the current step
        param_group['lr'] = lr_lambda(last_epoch)
```

## Subtopics

### [Training Loop with Adaptive Regularization & LR Decay](2408-training-loop-with-adaptive-regularization-lr-decay.md)
Defines the standard iterative optimization protocol for NeRF, combining ray-batched SGD with a two-phase loss schedule and exponential learning rate decay. It differs from [FreeNeRF](7471-pixelnerf.md) by using simpler auxiliary models for depth supervision and explicitly managing regularization weights that decay alongside the learning rate.

### [Training Schedule](3154-training-schedule.md)
Defines the temporal progression of hyperparameters, loss functions, and structural updates. Unlike learning rate scheduling which adjusts only step size, a training schedule actively changes the *active set* of losses and the *structural state* of the representation (e.g., enabling densification) over specific iteration intervals [G].

### [Hyperparameter Configuration for Stochastic Gradient Estimation](6748-hyperparameter-configuration-for-stochastic-gradient-estimat.md)
Defines hyperparameters for optimizing differentiable rendering using first-order and second-order stochastic gradient estimators. It sets apart from standard NeRF training by using **Sigma annealing** for distribution bandwidth control and **antithetic sampling** to reduce variance.

### [Exponential Decay Learning Rate Schedule](9145-exponential-decay-learning-rate-schedule.md)
An exponentially decaying schedule transitioning from an initial to a final value using a base raised to a normalized step exponent. It provides a non-linear, accelerating reduction in step size, preferred for stabilizing MLP weight convergence compared to linear schedules.

### [XYZ Learning Rate Scheduling](11823-xyz-learning-rate-scheduling.md)
Adjusts the learning rate specifically for 3D Gaussian point locations (xyz) to balance training speed and rendering quality. It targets spatial parameters sensitive to large learning rates, distinguishing itself from general scalar schedules.

### [Annealing Schedules](11839-annealing-schedules.md)
A unified framework supporting constant, linear, and exponential interpolation between initial and final values. It differs from exponential decay by using interpolation between bounds rather than a fixed decay rate, and from linear schedules by offering multiple decay modes.

### [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md)
A schedule transitioning from an initial to a final value using linear interpolation. It provides a deterministic, monotonic decay path, changing the rate by a constant absolute amount per step, unlike exponential decay which changes proportionally.

### View-count-based Learning Rate
Adjusts learning rates based on a view-count prior. Without this prior, PSNR can degrade up to -1.22 in the worst case [S6].

### Learning Rate Spike
A technique to increase the learning rate of specific parameters (instant NGP weights and vertex coordinates) after a densification event. It adds a spike function with duration $L$ on top of the decay function, increasing the rate back to its maximum before decaying again [S7, unofficial].

### Learning Rate Reset on Upsampling
If iteration is in the upsampling list, the learning rate scale is reset to 1. Otherwise, it is computed based on a decay target ratio. The optimizer is reinitialized with the new learning rates [S8, unofficial].

### Halving Training Iterations with Schedules
Training for 15K iterations instead of 30K, compensated by learning rate schedules for scale, rotation, and SH DC coefficients, and increased SH higher-order LR. This halves training time (10:24 → 05:40) with minimal quality loss (PSNR 26.51 → 26.47) [S9].

### SH Degree Scheduling
Gradually increases the Spherical Harmonics (SH) degree during training. It increases from 0 to 1 during coarse training, and then from 1 to the target degree $n$ during fine training, with each increase occurring after 10% of the total fine training iterations [S10, unofficial].

### Per-Parameter Learning Rate Scheduling
Adjusts learning rates for scaling, opacity, rotation, and feature parameters individually. Scaling LR has an optimal range of $[10^{-3}, 10^{-6}]$. Setting feature LR to 0 has little impact on speed or quality [S11, unofficial].

### Piecewise Schedule
A schedule composed of multiple sub-schedules applied sequentially at specified milestone steps. It uses `searchsorted` to find the active sub-schedule based on the current step [S12, unofficial].

### Cosine Decay Learning Rate Schedule
Optimizes networks using Adam with a learning rate linearly warmed up from 0 to $10^{-3}$ over the first 5,000 iterations, then cosine decayed to a minimum of $5 \times 10^{-5}$ [S13].

## Operating characteristics
- **Time Complexity**: The calculation of the learning rate is $O(1)$ per step. The overhead is negligible compared to the rendering and gradient computation steps.
- **Memory Complexity**: $O(1)$ additional memory for storing schedule parameters.
- **Performance**: Exponential and log-linear schedules are effective in preventing overshooting during the fine-tuning phase of radiance field optimization [S2, unofficial]. Cosine decay with warmup is effective for neural SDF reconstruction [S13].
- **Strengths**: Stabilizes convergence, allows for faster initial training, and enables reduction in total training iterations when combined with other schedules (e.g., halving iterations with LR schedules) [S9].
- **Limitations**: Requires careful tuning of parameters like delay steps and decay rates. Incorrect scheduling can lead to poor convergence or suboptimal final quality [S6].

## Background
Learning rate scheduling is a fundamental technique in optimization, widely used in deep learning. In the context of Neural Radiance Fields (NeRF) and 3D Gaussian Splatting, it has been adapted to handle the specific challenges of differentiable volume rendering, such as the need for stable convergence of MLP weights and the management of geometric primitives [S2, unofficial][S7, unofficial][S13]. The specific implementations cited here originate from various NeRF and Gaussian Splatting research projects and implementations [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial][S5, unofficial][S6][S7, unofficial][S8, unofficial][S9][S10, unofficial][S11, unofficial][S12, unofficial][S13].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S4] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S5] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S6] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S7] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S8] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S9] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
[S10] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S11] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S12] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S13] ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision, https://doi.org/10.1109/cvpr52729.2023.00025, fetched 2026-10-26
