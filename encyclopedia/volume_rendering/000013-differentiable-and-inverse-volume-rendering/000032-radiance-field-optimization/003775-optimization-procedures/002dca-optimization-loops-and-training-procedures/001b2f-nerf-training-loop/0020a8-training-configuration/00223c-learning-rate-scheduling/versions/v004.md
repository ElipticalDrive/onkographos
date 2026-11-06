# Learning Rate Scheduling

Also called: LR Scheduling, LR decay, LambdaLR Scheduler, Learning Rate Decay, Learning Rate Scheduler

## Overview
Learning rate scheduling adjusts the optimizer's step size during training to balance convergence speed and final accuracy. In radiance-field optimization, it prevents overshooting in later stages and adapts to the changing difficulty of fitting high-frequency details. It differs from [Training Loop Setup](1522-training-loop-setup.md) by focusing specifically on the parameter update magnitude, and from [Training Schedule](3154-training-schedule.md) by often being coupled with structural changes like densification.

## How it works
### Inputs and Outputs
**Inputs:**
- Current training step $i$ (or epoch $e$).
- Maximum training steps $T_{max}$ (or total epochs).
- Initial learning rate $lr_{init}$ and final learning rate $lr_{final}$ for specific parameter groups (e.g., means, SH coefficients).
- Batch count per epoch or total batch count $B$.
- Configuration parameters: gamma $\gamma$, step size $S$, spike duration $L$, densification iteration $I$.

**Outputs:**
- Updated learning rate $lr_{new}$ for each optimizer group.

### Data Structures
- **Optimizer Groups:** Typically divided by parameter type.
    - Group 0: Means (positions).
    - Group 1: Rotations.
    - Group 2: Scales.
    - Group 3: Spherical Harmonics (SH) coefficients.
- **Config Struct:**
    - `learningRate`: Base LR for non-SH groups.
    - `shLearningRate`: Base LR for SH groups.
    - `lr_scheduler_gamma`: Decay factor $\gamma$.
    - `lr_scheduler_step_size`: Step interval $S$.
    - `start_step`, `end_step`, `start_value`, `end_value`: For linear interpolation schedules.

### Steps
1. **Calculate Normalized Progress:**
   Compute the normalized step $t$ for exponential schedules:
   $$ t = \text{clamp}\left(\frac{i}{T_{max}}, 0, 1\right) $$
   [S1, unofficial]

2. **Exponential Decay for Means (Adam Group 0):**
   Interpolate between initial and final LR using logarithmic interpolation:
   $$ lr_{means}(t) = \exp\left( \log(lr_{init}) \cdot (1 - t) + \log(lr_{final}) \cdot t \right) $$
   [S1, unofficial]

3. **Power-of-2 Decay for General and SH LRs:**
   After processing each batch, update the learning rate by dividing by a power of 2 dependent on the batch count and epoch:
   $$ lr_{new} = \frac{lr_{old}}{2^{\frac{1}{B \cdot (e + 1)}}} $$
   This applies to both `configs->learningRate` (for means/rotations/scales) and `configs->shLearningRate`.
   [S2, unofficial]

4. **LambdaLR Scheduler (Alternative/Additional):**
   Define a lambda function for epoch-based decay:
   $$ lr_{lambda}(e) = \gamma^{\left(\frac{e}{S}\right)} $$
   [S3, unofficial]
   Create the scheduler:
   ```python
   lr_scheduler = torch.optim.lr_scheduler.LambdaLR(optimizer, lr_lambda, last_epoch=start_epoch - 1)
   ```
   [S3, unofficial]

5. **Linear Interpolation (If Configured):**
   If the schedule is defined by start/end values and steps:
   - Unpack `start_step`, `start_value`, `end_step`, `end_value`.
   - If the value is not a scalar, parse from config. If it is a list of length 3, prepend 0 to make it length 4.
   - Calculate interpolated value:
     $$ value = start\_value + (end\_value - start\_value) \cdot \max\left(\min\left(1.0, \frac{i - start\_step}{end\_step - start\_step}\right), 0.0\right) $$
   [S4, unofficial]

6. **Learning Rate Spike (After Densification):**
   If densification occurs at iteration $I$, add a spike function to the decaying learning rate $l(i)$ for specific parameters (instant NGP weights and vertex coordinates):
   $$ \Delta lr(i) = \mathbb{1}\{i - I > 0\} \cdot (l(0) - l(I)) \cdot \exp\left(-\frac{6(i - I)}{L}\right) $$
   The new learning rate is $l(i) + \Delta lr(i)$. This increases the LR back to its initial value before decaying over $L$ iterations.
   [S6, unofficial]

### Parameters
- `lr_init`: Initial learning rate. Typical values vary by implementation (e.g., $1.6 \times 10^{-4}$ for means in some Gaussian Splatting variants).
- `lr_final`: Final learning rate. Often $1/10$ to $1/100$ of `lr_init`.
- `gamma` ($\gamma$): Decay factor for LambdaLR. Typical value: 0.66.
- `step_size` ($S$): Step interval for LambdaLR. Typical value: 300.
- `L`: Duration of the learning rate spike in iterations.
- `batchCount` ($B$): Number of batches per epoch.

### Edge Cases and Pitfalls
- **Division by Zero:** Ensure $T_{max} > 0$ and $B > 0$ to avoid division by zero in normalization and power-of-2 decay.
- **Negative Exponents:** Clamp $t$ to $[0, 1]$ to ensure valid exponential interpolation.
- **Spike Timing:** The spike function is only active for $i > I$. Ensure $I$ is correctly identified during densification events.
- **Precision:** Use logarithmic interpolation for exponential decay to avoid numerical instability when $lr_{init}$ and $lr_{final}$ differ by orders of magnitude.
- **View-Count Prior:** Without view-count-based adjustments, PSNR can degrade up to -1.22 in the worst case.
  [S5]

## Subtopics

### View-count-based Learning Rate
Adjusts learning rates based on a view-count prior to prioritize under-reconstructed regions. It prevents PSNR degradation (up to -1.22) that occurs without this prior. Choose this when dealing with sparse or unevenly sampled input views. [S5]

### Learning Rate Spike
A technique to temporarily increase the learning rate of specific parameters (instant NGP weights, vertex coordinates) after a densification event. It allows rapid accommodation for structural changes by adding an exponential decay spike on top of the base decay function. Choose this when using dynamic densification strategies like those in 3D Gaussian Splatting or Radiance Meshes. [S6, unofficial]

## Operating characteristics
- **Time Complexity:** $O(1)$ per step for standard schedules. $O(1)$ for spike calculation.
- **Memory Complexity:** $O(1)$ additional memory for storing current LR and config parameters.
- **Performance:** Exponential decay provides smooth convergence. Power-of-2 decay offers a simple, effective alternative. Spikes can improve final quality after densification but require careful tuning of $L$.
- **Strengths:** Simple to implement, effective for stabilizing training, adaptable to different parameter groups.
- **Limitations:** Requires careful tuning of hyperparameters ($\gamma$, $S$, $L$). May not be optimal for all radiance field representations.
- **When to Use:** Standard practice for most radiance field optimizations. Use spikes when densification is employed. Use view-count-based scheduling for sparse inputs.
- **When Not to Use:** When using fixed learning rates for simplicity or when the training is very short.

## Background
Learning rate scheduling is a fundamental concept in optimization, widely used in deep learning. In the context of volume rendering and radiance fields, it has been adapted to handle the unique challenges of fitting high-frequency details and dynamic structures. The power-of-2 decay and exponential schedules are common in implementations like Clear-plenoxels [S2, unofficial] and Msplat [S1, unofficial]. The LambdaLR scheduler is a standard PyTorch utility used in various neural rendering pipelines [S3, unofficial]. The learning rate spike technique was introduced in Radiance Meshes to handle densification [S6, unofficial]. View-count-based scheduling was proposed in Direct Voxel Grid Optimization to address sparse input issues [S5].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S3] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S4] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S5] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S6] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
