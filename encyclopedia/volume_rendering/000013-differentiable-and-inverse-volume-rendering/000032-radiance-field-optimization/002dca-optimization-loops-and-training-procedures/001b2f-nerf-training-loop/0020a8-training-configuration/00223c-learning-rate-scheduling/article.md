# Learning Rate Scheduling

Also called: LR Scheduling, LR decay, LambdaLR Scheduler, Learning Rate Decay, Learning Rate Scheduler

## Overview
Learning rate scheduling adjusts the optimizer's step size during training to improve convergence stability and final reconstruction quality in radiance-field optimization [G]. It differs from static learning rates by dynamically scaling parameters based on training progress or structural changes, such as densification events [S6, unofficial]. This method builds on [Training Configuration](2332-training-configuration.md) by defining the temporal evolution of the learning rate parameter [G].

## How it works
The method implements time-based exponential decay, power-of-2 batch decay, lambda decay, linear interpolation, and event-driven spikes to modulate the learning rates of model parameters (e.g., means, weights, vertex coordinates) [S1, S2, S3, S6, unofficial].

### Inputs and Outputs
- **Inputs**: Current step/iteration $i$, maximum steps $T_{max}$, initial learning rate $lr_{init}$, final learning rate $lr_{final}$, decay gamma $\gamma$, step size $S$, densification iteration $I$, spike duration $L$, batch count, epoch [S1, S2, S3, S6, unofficial].
- **Outputs**: Scaled learning rate $lr(i)$ for the current step [G].

### Data Structures
- **Configuration**: Dictionary or struct containing `lr_init`, `lr_final`, `gamma`, `step_size`, `start_step`, `end_step`, `start_value`, `end_value` [S3, unofficial].
- **Optimizer State**: The learning rate is typically applied via a scheduler wrapper (e.g., `torch.optim.lr_scheduler.LambdaLR`) that updates the optimizer's parameter groups [G].

### Formulas
1. **Exponential Decay (Normalized Step)**: For a normalized step $t = \text{clamp}(i / T_{max}, 0, 1)$, the learning rate for means is $lr_{means}(t) = \exp(\log(lr_{init}) \cdot (1 - t) + \log(lr_{final}) \cdot t)$ [S1, unofficial].
2. **Power-of-2 Batch/Epoch Decay**: After processing each batch, divide the learning rate by $2^{1 / (\text{batchCount} \cdot (\text{epoch} + 1))}$ [S2, unofficial].
3. **Lambda Decay**: Define a lambda function $lr\_lambda(epoch) = \gamma^{(epoch / S)}$ and apply it via a scheduler [S3, unofficial].
4. **Linear Interpolation**: Parse `start_step`, `start_value`, `end_value`, `end_step` from config. Compute $value = start\_value + (end\_value - start\_value) \cdot \max(\min(1.0, (current\_step - start\_step) / (end\_step - start\_step)), 0.0)$ [S3, unofficial].
5. **Learning Rate Spike**: After densification at iteration $I$, add a spike function with duration $L$ on top of the decay function $l(i)$ for instant NGP weights and vertex coordinates: $\mathbb{1}\{i - I > 0\} (l(0) - l(I)) \exp(-\frac{6(i-I)}{L})$ [S6, unofficial]. The total learning rate becomes $l_{total}(i) = l(i) + \text{Spike}(i)$ [G].

### Parameters
- **$lr_{init}$**: Initial learning rate [S1, unofficial].
- **$lr_{final}$**: Target final learning rate [S1, unofficial].
- **$\gamma$**: Gamma value for learning rate decay [S3, unofficial].
- **$S$**: Step size for learning rate decay [S3, unofficial].
- **$L$**: Duration of the learning rate spike in iterations [S6, unofficial].
- **`start_step`, `end_step`, `start_value`, `end_value`**: Boundaries and values for linear interpolation [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_lr_exponential_decay(step, max_steps, lr_init, lr_final):
    t = max(0.0, min(1.0, step / max_steps))
    return math.exp(math.log(lr_init) * (1.0 - t) + math.log(lr_final) * t)

def compute_lr_power_of_2_decay(lr_old, batch_count, epoch):
    return lr_old / (2.0 ** (1.0 / (batch_count * (epoch + 1.0))))

def compute_lr_lambda_decay(lr_init, gamma, epoch, step_size):
    return lr_init * (gamma ** (epoch / step_size))

def compute_lr_linear(step, start_step, end_step, start_value, end_value):
    if end_step == start_step:
        return start_value
    progress = max(0.0, min(1.0, (step - start_step) / (end_step - start_step)))
    return start_value + (end_value - start_value) * progress

def compute_lr_with_spike(step, densification_step, initial_lr, current_decay_lr, spike_duration):
    if step <= densification_step:
        return current_decay_lr
    spike = (initial_lr - current_decay_lr) * math.exp(-6.0 * (step - densification_step) / spike_duration)
    return current_decay_lr + spike
```

### Edge Cases and Pitfalls
- **Division by Zero**: Ensure `step_size` and `end_step - start_step` are non-zero [G].
- **Negative Steps**: Clamp `step` to $\ge 0$ [G].
- **Precision**: Use log-space interpolation for exponential decay to avoid numerical instability with very small learning rates [G].
- **Spike Timing**: The spike function is only active for $i > I$ [S6, unofficial].

## Subtopics

### View-count-based Learning Rate
Adjusts learning rates based on a view-count prior to prioritize under-reconstructed regions [S5]. Without this prior, PSNR can degrade up to -1.22 in the worst case [S5].

### Learning Rate Spike
A technique to increase the learning rate of specific model parameters (instant NGP weights and vertex coordinates) after a densification event to allow rapid accommodation for changes [S6, unofficial]. It adds a spike function with duration $L$ on top of the decay function [S6, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per step for computing the learning rate [G].
- **Memory Complexity**: $O(1)$ additional memory for scheduler state [G].
- **Strengths**: Improves convergence stability; prevents oscillation in later stages; handles structural changes (densification) effectively [G].
- **Limitations**: Requires careful tuning of decay parameters; spike duration $L$ must be chosen appropriately to avoid instability [G].
- **When to Use**: Standard for NeRF and Gaussian Splatting training to ensure smooth convergence and high final quality [G].

## Background
Learning rate scheduling is a standard optimization technique in deep learning [G]. In the context of radiance fields, it was adapted to handle the unique convergence properties of neural implicit representations and 3D Gaussian splatting [G]. The exponential decay and spike mechanisms are specifically tailored to the iterative densification and refinement processes in these methods [S1, S6, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S4] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S5] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S6] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
