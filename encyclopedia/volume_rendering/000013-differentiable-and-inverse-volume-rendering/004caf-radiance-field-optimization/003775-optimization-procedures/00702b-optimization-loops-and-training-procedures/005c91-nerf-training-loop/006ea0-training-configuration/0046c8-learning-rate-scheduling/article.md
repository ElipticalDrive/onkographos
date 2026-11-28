# Learning Rate Scheduling

Also called: LR decay, LR Scheduler, LR Scheduling, Learning Rate Decay, Learning Rate Schedulers

## Overview
Learning Rate Scheduling (LRS) dynamically adjusts the optimizer's step size during the training of radiance fields to balance convergence speed and final reconstruction quality [G]. It solves the problem of optimization instability and slow convergence by decaying the learning rate from an initial high value to a lower final value, often with specific phases for warm-up or delay [S3, unofficial]. It differs from [Training Schedule](3154-training-schedule.md) by adjusting only the scalar step size rather than the active set of losses or structural representation state, and from [Adam Optimisation with Adaptive Scaling](3093-adam-optimisation-with-adaptive-scaling.md) by providing the external schedule that modulates the base step size used by the adaptive optimizer [G].

## How it works
**Inputs and Outputs:**
- **Inputs:** Current optimization step $i$ (or epoch), initial learning rate $lr_{init}$, final learning rate $lr_{final}$, maximum steps $max\_steps$, and optional delay parameters ($lr_{delay\_steps}$, $lr_{delay\_mult}$) [S3, unofficial].
- **Output:** The current learning rate $lr(i)$.

**Data Structures:**
No persistent data structures are required for the core scheduling logic; it is a stateless function of the step count [G]. However, implementations may cache the schedule or use a scheduler object that tracks the last epoch/step [G].

**Formulas and Steps:**

1. **Log-Linear Exponential Decay (Standard):**
   The schedule decays the learning rate log-linearly from $lr_{init}$ to $lr_{final}$ over $max\_steps$ [S3, unofficial].
   
   Calculate the normalized progress $t$:
   $$ t = \text{clip}\left(\frac{i}{max\_steps}, 0, 1\right) $$
   
   Calculate the base learning rate using log-linear interpolation:
   $$ lr_{base}(i) = \exp\left( \ln(lr_{init}) \cdot (1 - t) + \ln(lr_{final}) \cdot t \right) $$
   
   If a delay phase is enabled ($lr_{delay\_steps} > 0$), calculate a delay multiplier $delay\_rate$:
   $$ delay\_rate = lr_{delay\_mult} + (1 - lr_{delay\_mult}) \cdot \sin\left( \frac{\pi}{2} \cdot \text{clip}\left(\frac{i}{lr_{delay\_steps}}, 0, 1\right) \right) $$
   If $lr_{delay\_steps} \le 0$, $delay\_rate = 1.0$ [S3, unofficial].
   
   The final learning rate is:
   $$ lr(i) = delay\_rate \cdot lr_{base}(i) $$
   
   Edge case: If $i < 0$ or both $lr_{init}$ and $lr_{final}$ are 0.0, return 0.0 [S3, unofficial].

2. **Power-of-2 Batch Decay:**
   In some implementations, the learning rate is divided by a power of 2 after each batch, dependent on the batch count and epoch [S2, unofficial].
   
   $$ lr_{new} = \frac{lr_{old}}{2^{\frac{1}{batchCount \cdot (epoch + 1)}}} $$
   
   This applies separately to the main learning rate and the Spherical Harmonics (SH) learning rate [S2, unofficial].

3. **LambdaLR (PyTorch Style):**
   An exponential decay can be defined via a lambda function for use with `torch.optim.lr_scheduler.LambdaLR` [S5, unofficial]:
   $$ \lambda(epoch) = \gamma^{\frac{epoch}{step\_size}} $$
   where $\gamma$ is the gamma value and $step\_size$ is the interval [S5, unofficial].

4. **Piecewise Schedules:**
   Complex schedules can be composed of multiple sub-schedules applied sequentially at milestone steps [S1, unofficial].
   - **Layout:** A list of `Schedule` objects and a 1D array of cumulative `milestones`.
   - **Lookup:** Given a step, find the index `idx` using `searchsorted` on milestones. The base step for the sub-schedule is `step - milestones[idx-1]` (or `step` if `idx=0`) [S1, unofficial].

5. **Per-Parameter Scheduling:**
   Different parameter groups (e.g., means, features, scale, rotation) can have distinct learning rates [S1, unofficial][S11, unofficial]. For 3D Gaussians, the mean positions (Adam group 0) often use an exponential decay from `means_lr_init` to `means_lr_final` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class Schedule:
    def __init__(self, func):
        self.func = func
    def get(self, step):
        return self.func(step)

def get_learning_rate(step, max_steps, lr_init, lr_final, lr_delay_steps=0, lr_delay_mult=1.0):
    """
    Computes the learning rate using log-linear decay with optional delay.
    
    Args:
        step (int): Current optimization step.
        max_steps (int): Total number of optimization steps.
        lr_init (float): Initial learning rate.
        lr_final (float): Final learning rate.
        lr_delay_steps (int): Number of steps for the initial delay phase.
        lr_delay_mult (float): Multiplier for the learning rate during the delay phase.
        
    Returns:
        float: The computed learning rate.
    """
    # Edge case: invalid step or zero rates
    if step < 0 or (lr_init == 0.0 and lr_final == 0.0):
        return 0.0

    # Calculate normalized progress t
    t = np.clip(step / max_steps, 0, 1)
    
    # Log-linear interpolation
    log_lr_init = np.log(lr_init)
    log_lr_final = np.log(lr_final)
    log_lerp = np.exp(log_lr_init * (1 - t) + log_lr_final * t)
    
    # Calculate delay multiplier
    if lr_delay_steps > 0:
        # Sinusoidal delay ramp
        delay_t = np.clip(step / lr_delay_steps, 0, 1)
        delay_rate = lr_delay_mult + (1 - lr_delay_mult) * np.sin(0.5 * np.pi * delay_t)
    else:
        delay_rate = 1.0
        
    return delay_rate * log_lerp

def get_piecewise_schedule(step, schedules, milestones):
    """
    Retrieves the learning rate from a piecewise schedule.
    
    Args:
        step (int): Current step.
        schedules (list): List of Schedule objects, each with a get(step) method.
        milestones (list): Cumulative step counts where each sub-schedule begins.
        
    Returns:
        float: The learning rate from the active sub-schedule.
    """
    # Find the index of the schedule active at this step
    idx = np.searchsorted(milestones, step, side='right')
    
    # Get the active schedule
    schedule = schedules[idx]
    
    # Calculate the relative step within this sub-schedule
    base_idx = milestones[idx - 1] if idx >= 1 else 0
        
    return schedule.get(step - base_idx)
```

**Parameters:**
- `lr_init`: Initial learning rate. Typical values: $2 \cdot 10^{-3}$ [S13], $10^{-3}$ [S12].
- `lr_final`: Final learning rate. Typical values: $2 \cdot 10^{-5}$ [S13], $5 \cdot 10^{-5}$ [S12].
- `max_steps`: Total optimization steps. Default: $1,000,000$ [S3, unofficial].
- `lr_delay_steps`: Steps for initial delay. Default: 0 [S3, unofficial].
- `lr_delay_mult`: Delay multiplier. Default: 1.0 [S3, unofficial].
- `gamma`: Exponential decay factor for LambdaLR.
- `step_size`: Interval for LambdaLR decay.

**Edge Cases and Pitfalls:**
- **Zero Rates:** If both initial and final rates are zero, the function must return zero to avoid division by zero or invalid log operations [S3, unofficial].
- **Negative Steps:** Steps before training begins should return zero [S3, unofficial].
- **Precision:** Log-linear interpolation preserves the geometric mean of the rates, which is often preferred over linear interpolation for multiplicative decay [S3, unofficial].
- **Boundary Conditions:** Ensure `t` is clamped to $[0, 1]$ to handle steps outside the training range [S3, unofficial].

## Subtopics

### [Training Schedule](3154-training-schedule.md)
Training schedules define the temporal progression of optimization hyperparameters, loss functions, and structural updates during differentiable volume rendering training. They solve unstable convergence and slow training by decoupling coarse geometry acquisition from fine detail refinement, actively managing scene representation complexity and optimization sensitivity over time.

### [View-count-based Learning Rate](4682-view-count-based-learning-rate.md)
This method adjusts the learning rates of the optimization process based on a view-count prior, which accounts for the number of training views available. It solves the problem of slow convergence or suboptimal reconstruction in radiance field optimization when the number of input images is limited. It sets itself apart from standard schedules by dynamically scaling the learning rate according to the density of observed views, rather than relying solely on iteration count or time. Without this prior, peak signal-to-noise ratio (PSNR) can degrade by up to -1.22 in the worst case.

### [Hyperparameter Configuration for Stochastic Gradient Estimation](6748-hyperparameter-configuration-for-stochastic-gradient-estimat.md)
This article defines the hyperparameter configuration required to optimize differentiable rendering methods using first-order (OurG, OurH) and second-order (OurHVP, OurHVPA) stochastic gradient estimators. It solves the problem of selecting optimal sampling, learning rate, and trust region parameters for inverse rendering tasks such as caustic optimization, shape reconstruction, and texture mapping. What sets this configuration apart from standard NeRF training is the explicit use of Sigma annealing for distribution bandwidth control and antithetic sampling to reduce variance, alongside distinct parameter sets for first-order versus second-order optimization loops.

### [XYZ Learning Rate Scheduling](11823-xyz-learning-rate-scheduling.md)
XYZ Learning Rate Scheduling adjusts the learning rate specifically for the 3D Gaussian point locations (xyz) during radiance-field optimization to balance training speed and rendering quality. It distinguishes itself from general scalar schedules by targeting the spatial parameters of the Gaussian primitives, addressing the sensitivity of point positions to large learning rates which can degrade rendering quality.

### [Per-Parameter Learning Rate Scheduling](11832-per-parameter-learning-rate-scheduling.md)
Per-Parameter Learning Rate Scheduling adjusts the learning rates for specific parameter groups (scaling, opacity, rotation, and features) independently during each training iteration. It addresses the need to tune convergence speed and final rendering quality by allowing distinct optimization dynamics for different geometric and radiometric properties, setting it apart from uniform scheduling methods that apply a single rate to all parameters.

### [Annealing Schedules](11839-annealing-schedules.md)
Annealing Schedules provide a unified framework for parameter annealing during training, supporting constant, linear, and exponential schedules. Unlike specialized schedules that implement only one decay strategy, this framework offers multiple decay modes within a single implementation.

### [Piecewise Schedule](11853-piecewise-schedule.md)
A learning rate scheduling method that composes multiple sub-schedules applied sequentially at specified milestone steps. It allows complex training dynamics by chaining distinct optimization behaviors (e.g., warmup, decay, fine-tuning) without requiring a single monolithic formula. It differs from continuous decay schedules like [Exponential Decay Schedules](18486-exponential-decay-schedules.md) by allowing abrupt changes in behavior at discrete boundaries, and from [Annealing Schedules](11839-annealing-schedules.md) by its explicit modular composition of independent schedule objects rather than a single parametric function.

### [Exponential Decay Schedules](18486-exponential-decay-schedules.md)
Exponential decay schedules reduce the learning rate multiplicatively over training steps, providing a smooth, continuous transition from high initial exploration to fine-grained convergence. This family differs from [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md) by applying relative reductions rather than absolute ones, and from [Cosine Decay Learning Rate Schedule](14118-cosine-decay-learning-rate-schedule.md) by avoiding the symmetric "slow-fast-slow" trajectory in favor of a monotonic decrease. It is primarily used in NeRF optimization to stabilize training when the model has converged to a coarse geometry but requires precise refinement of appearance and density fields.

### Learning Rate Spike
A technique to increase the learning rate of specific model parameters (instant NGP weights and vertex coordinates) after a densification event to allow rapid accommodation for changes [S8, unofficial]. After densification at iteration $I$, the learning rate is increased by adding a spike function with duration $L$ on top of the decay function $l(i)$ [S8, unofficial]. The spike function increases the decaying learning rate back up to its maximum (initial) value, before decaying it back down in $L$ iterations [S8, unofficial].
Formula:
$$ \mathbb{1}\{i - I > 0\} (l(0) - l(I)) \exp\left(-\frac{6(i-I)}{L}\right) $$
where $i$ is the current iteration, $I$ is the iteration of densification, $l(0)$ is the initial learning rate, $l(I)$ is the learning rate at iteration $I$, and $L$ is the duration of the spike [S8, unofficial].
Parameter: $L$: The duration of the learning rate spike in iterations [S8, unofficial].

### Learning Rate Reset on Upsampling
If iteration is in `upsamp_list`, pop target voxel count `n_voxels = N_voxel_list.pop(0)` [S9, unofficial]. Compute resolution `reso_cur = N_to_reso(n_voxels, tensorf.aabb)` [S9, unofficial]. Limit samples `nSamples = min(args.nSamples, cal_n_samples(reso_cur,args.step_ratio))` [S9, unofficial]. Call `tensorf.upsample_volume_grid(reso_cur)` [S9, unofficial]. If `args.lr_upsample_reset` is true, set `lr_scale = 1` [S9, unofficial]. Otherwise, compute `lr_scale = args.lr_decay_target_ratio ** (iteration / args.n_iters)` [S9, unofficial]. Get optimizer groups `grad_vars = tensorf.get_optparam_groups(args.lr_init*lr_scale, args.lr_basis*lr_scale)` and reinitialize `optimizer = torch.optim.Adam(grad_vars, betas=(0.9, 0.99))` [S9, unofficial].

### Halving Training Iterations with Schedules
Training for 15K iterations instead of 30K, compensated by learning rate schedules for scale, rotation, and SH DC coefficients, and increased SH higher-order LR [S10]. Train for 15K or 7.5K iterations instead of 30K [S10]. Apply additional learning rate schedules to enable rapid initial growth, compensating for the reduced iteration count [S10]. SH higher-order coefficient learning rate is increased 5x to 0.000625 [S10].
Performance:
- 15K Iterations: PSNR 26.47, LPIPS 0.236, Opt Time 05:40 [S10].
- 30K Iterations: PSNR 26.51, LPIPS 0.243, Opt Time 10:24 [S10].
- 15K iterations + LR schedules: PSNR 26.47, SSIM 0.819, LPIPS 0.236, Opt Time 05:40, FPS 248, Final Avg Scale 0.0166 [S10].
- 7.5K iterations + LR schedules: PSNR 26.23, SSIM 0.817, LPIPS 0.238, Opt Time 02:58, FPS 229, Final Avg Scale 0.0153 [S10].

### SH Degree Scheduling
A training strategy to gradually increase the Spherical Harmonics (SH) degree to improve image quality, as higher degrees do not significantly contribute if used from the start [S11, unofficial]. During the coarse training course, increase the SH degree from 0 to 1 [S11, unofficial]. During fine training, gradually increase the degree from 1 to the target degree $n$ [S11, unofficial]. Each increase in degree happens after 10% of the total fine training iterations have completed [S11, unofficial]. The default SH degree used in experiments is 1 [S11, unofficial]. The target degree $n$ is chosen based on scene requirements [S11, unofficial].

### Uncertainty Tempering
A strategy to gradually enlarge the uncertainty parameter epsilon during training, providing more freedom for the radiance field to explore as training proceeds, opposite to learning rate decay [S12].
Formula:
$$ \epsilon' = \gamma \epsilon $$
where $\epsilon'$ is the updated uncertainty, and $\gamma$ is the exponential growth rate [S12].
Loss:
$$ L = L_{rgb} + \lambda_d L_{gd} + \lambda_v L_v $$
where $L$ is the total loss, $L_{rgb}$ is the RGB loss, $L_{gd}$ is ground density supervision, $L_v$ is multi-view consistency supervision, $\lambda_d = 0.2$, and $\lambda_v = 0.5$ [S12].
Parameter: $\gamma$ (gamma): Exponential growth rate for uncertainty tempering. Set to 1.0005 in experiments [S12].

### Exponential Growth Weight Scheduler
A scheduler that gradually increases the regularization strength $\lambda$ from a minimum to a maximum value during training to handle the moving target of distribution matching [S13].
Step: Combine the scheduled $\lambda(t)$ with the current learning rate (which decays via cosine annealing) to determine the actual regularization strength applied at each step [S13].
Formula:
$$ \lambda(t) = \lambda_{min} + (\lambda_{max} - \lambda_{min}) \frac{r^{t-1} - 1}{r - 1} $$
where $r$ is the growth rate [S13].
Parameters:
- $\lambda_{max}$: The maximum regularization strength, a hyperparameter to be tuned [S13].
- $\lambda_{min}$: The minimum regularization strength, default value is 0 [S13].
- $r$: The growth rate parameter, where $r \in (1, \infty]$. Default recommended value is 500 [S13].
- $t_{max}$: The total number of training steps, at which $\lambda(t)$ reaches $\lambda_{max}$ [S13].

## Operating characteristics
- **Time Complexity:** The scheduling calculation itself is $O(1)$ per step [G]. The impact on training time is indirect; effective scheduling reduces the number of iterations required for convergence [S10]. For example, halving iterations with proper scheduling reduced training time from 10:24 to 05:40 (approx. 2x speedup) with minimal quality loss [S10].
- **Memory Complexity:** $O(1)$ for stateless schedules. Piecewise schedules require storing the list of sub-schedules and milestones, which is negligible compared to model parameters [G].
- **Measured Performance:** On standard NeRF datasets, log-linear decay with warm-up/delay stabilizes training and improves final PSNR compared to constant learning rates [S3, unofficial]. In 3D Gaussian Splatting, per-parameter scheduling allows for faster convergence of geometry (xyz) while maintaining stability in features [S1, unofficial].
- **Scaling:** Schedules scale with the total number of iterations. For ultra-large scale scenes, schedules may need to be adjusted to account for longer training times or different convergence dynamics [S11, unofficial].
- **Strengths:** Prevents divergence in early training, allows fine-tuning in later stages, and can be tailored to specific parameter groups (e.g., xyz vs. SH coefficients) [G].
- **Limitations:** Requires careful tuning of initial/final rates and decay curves. Poorly chosen schedules can lead to suboptimal local minima or slow convergence [G].
- **When to Use:** Standard practice for almost all radiance field optimizations. Use log-linear or cosine decay for general MLPs. Use per-parameter schedules for Gaussian Splatting. Use piecewise schedules for complex multi-phase training (e.g., coarse-to-fine) [G].
- **When Not to Use:** Not applicable for methods that use adaptive optimizers exclusively without external scheduling (though even Adam often benefits from LR decay) [G].

## Background
Learning rate scheduling is a fundamental technique in deep learning optimization, originating from the need to stabilize gradient descent and improve convergence [G]. In the context of Neural Radiance Fields (NeRF) and 3D Gaussian Splatting, it has been adapted to handle the unique challenges of differentiable volume rendering, such as the sensitivity of geometry parameters and the need for multi-scale optimization [S1, unofficial][S3, unofficial][S8, unofficial][S10][S11, unofficial]. The specific variants listed (e.g., log-linear decay, per-parameter scheduling, learning rate spikes) have emerged from various research groups and open-source implementations to address specific convergence issues in radiance field training [S1, unofficial][S2, unofficial][S3, unofficial][S5, unofficial][S8, unofficial][S10][S11, unofficial][S12][S13].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S4] gsgen3d/gsgen, https://github.com/gsgen3d/gsgen, fetched 2026-10-27 (unofficial)
[S5] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S6] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26 (unofficial)
[S7] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S8] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S9] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29 (unofficial)
[S10] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28
[S11] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S12] ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision, https://doi.org/10.1109/cvpr52729.2023.00025, fetched 2026-10-26
[S13] SGCNeRF: Few-Shot Neural Rendering via Sparse Geometric Consistency Guidance, https://doi.org/10.1109/tcsvt.2025.3610950, fetched 2026-10-30
[S14] MapNeRF: Incorporating Map Priors into Neural Radiance Fields for Driving View Simulation, https://doi.org/10.1109/iros55552.2023.10341681, fetched 2026-10-26
