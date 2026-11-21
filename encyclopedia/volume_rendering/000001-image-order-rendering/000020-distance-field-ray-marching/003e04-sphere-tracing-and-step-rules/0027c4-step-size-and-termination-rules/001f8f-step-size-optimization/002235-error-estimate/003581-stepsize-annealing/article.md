# Stepsize Annealing

Also called: Sampling Annealing, Progressive Supersampling, Max Time-Step Scaling, Shrinking Sampling Procedure

## Overview
Stepsize Annealing is a training strategy for differentiable volume rendering and neural radiance fields that varies the sampling density along rays over the course of training epochs or iterations [S1, unofficial][S2]. It addresses the trade-off between convergence speed and geometric accuracy by starting with coarse sampling (large step sizes or few points) to accelerate early learning and progressively refining the sampling (smaller step sizes or more points) to recover fine details [S1, unofficial][S2]. It is distinct from static error-based adaptive stepping (see [Error Estimate](2768-error-estimate.md)) because the sampling schedule is determined by training progress rather than local field gradients or error metrics at inference time [S1, unofficial][S2].

## How it works
The method modifies the number of samples or the step size used during the ray marching process as a function of the training iteration $u$ or epoch $e$. There are two primary formulations found in the literature: one based on sampling rate scaling [S1, unofficial] and one based on linear point count increase [S2]. A variant for temporal stability uses max time-step scaling [S3, unofficial], and a variant for surface compactness uses shrinking sampling [S4, unofficial].

### Inputs and Outputs
*   **Inputs:** Current training iteration/epoch ($u$ or $e$), total training iterations/epochs ($U$ or $E$), hyperparameters defining the range and speed of annealing.
*   **Outputs:** The number of sampling points $N_t$ or the sampling rate $s(e)$ to be used for the current training step.

### Data Structures
No specific complex data structures are required. The method relies on scalar values for the current progress and hyperparameters.

### Formulas and Parameters

#### 1. Quadratic Sampling Rate Annealing [S1, unofficial]
This method varies the sampling rate $s$ within a range $[s_l, s_h]$.

$$ s(e) = s_l \left(1 - \left(\frac{e}{E}\right)^2\right) + \left(\frac{e}{E}\right)^2 s_h $$

*   $s(e)$: Sampling rate at epoch $e$.
*   $s_l$: Lower bound of sampling rate. Typical value: $0.1$ [S1, unofficial].
*   $s_h$: Upper bound of sampling rate. Typical value: $2.0$ [S1, unofficial].
*   $e$: Current epoch.
*   $E$: Total number of epochs.

#### 2. Linear Point Count Annealing [S2]
This method increases the number of sampling points $N_t$ linearly to address near-field artifacts.

$$ N_t = \min(N_{\max}, \lfloor u/\eta \rfloor + N_{start}) $$

*   $N_t$: Number of sampling points along a ray at iteration $u$.
*   $N_{\max}$: Maximum number of sampling points.
*   $u$: Current training iteration.
*   $N_{start}$: Number of sampling points at the start of training.
*   $\eta$: Hyperparameter controlling the increasing speed of sampling points.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_sampling_config(epoch, total_epochs, sl=0.1, sh=2.0):
    """
    Returns the sampling rate for quadratic annealing [S1, unofficial].
    """
    progress = epoch / total_epochs
    # Clamp progress to [0, 1] to avoid numerical issues at final epoch
    progress = min(1.0, max(0.0, progress))
    sampling_rate = sl * (1 - progress**2) + (progress**2) * sh
    return sampling_rate

def get_num_samples(iteration, max_iterations, n_max, n_start, eta):
    """
    Returns the number of sampling points for linear annealing [S2].
    """
    # Calculate linear increase
    raw_count = (iteration // eta) + n_start
    # Cap at maximum
    num_samples = min(n_max, raw_count)
    return num_samples

def get_max_time_step(stage, epoch_in_stage, t_full=8):
    """
    Returns the max time step for Max Time-Step Scaling [S3, unofficial].
    """
    # At the beginning of each training stage, set max time step to 2 for the initial epoch
    if epoch_in_stage == 0:
        return 2
    # Restore to full time step T
    return t_full

def get_shrinking_region_bounds(current_iteration, total_iterations, initial_bounds, final_bounds):
    """
    Returns the sampling region bounds for Shrinking Sampling Procedure [S4, unofficial].
    """
    progress = current_iteration / total_iterations
    # Linearly interpolate bounds from initial to final
    current_bounds = initial_bounds * (1 - progress) + final_bounds * progress
    return current_bounds
```

### Edge Cases and Pitfalls
*   **Near-Field Artifacts:** In forward-facing real-world scenes, floating artifacts often appear close to the camera. Linear point count annealing ($N_t$) specifically addresses this by ensuring sufficient sampling density near the camera early on, though it increases points over time to refine geometry [S2].
*   **Memory Bottlenecks:** Early training with dense sampling can cause peak memory costs. Max time-step scaling mitigates this by starting with a reduced max time step (e.g., 2) and restoring it (e.g., 8) once sampling stabilizes [S3, unofficial].
*   **Surface Compactness:** Without a shrinking sampling procedure, learned occupancy fields may result in noisy normal maps and dents in surface geometry, even if rendered images look realistic [S4, unofficial].

## Subtopics

### Max Time-Step Scaling
A strategy to keep training stable under large maximum time steps ($T$). It starts with a reduced maximum time step (e.g., 2) in the initial epoch of each training stage to avoid peak memory costs from dense early samples, then restores $T$ (e.g., 8) once sampling becomes stable [S3, unofficial]. This preserves full temporal capacity while removing the initial memory bottleneck [S3, unofficial].

### Shrinking Sampling Procedure
A training schedule that progressively reduces the sampling interval around the surface to force the learned occupancy field to converge tightly to the true geometry. It initializes the sampling region to cover the entire volume and gradually shrinks it to a minimal neighborhood around the predicted surface, using root-finding on alpha values to locate the surface within the shrinking bounds [S4, unofficial]. Without this, normal maps contain random noise and face shapes develop dents [S4, unofficial].

## Operating characteristics
*   **Training Time:** Stepsize annealing reduces training time by an average of 33.4% compared to training with a constant small step size (e.g., $s=2.0$) while achieving comparable visual results [S1, unofficial].
*   **Convergence:** It benefits from faster convergence at low sampling rates in early epochs for rough approximation and increases sampling rate towards the end to refine the optimized transfer function [S1, unofficial].
*   **Quality:** Produces visual results comparable to training with a fixed high sampling rate (e.g., $s=2.0$) [S1, unofficial].
*   **Limitations:** The method is a training-time optimization; it does not inherently improve inference-time rendering speed unless the final model is configured to use the final annealed parameters. It requires careful tuning of hyperparameters like $s_l, s_h, \eta, N_{start}, N_{max}$ [S1, unofficial][S2].

## Background
Stepsize annealing was introduced as a training acceleration method for differentiable volume rendering models [S1, unofficial]. It was proposed to address the diminishing returns of increasing sampling rates beyond a certain point (e.g., $s=1.0$) and to accelerate training by starting with a high rendering step size in early epochs [S1, unofficial]. Variants have been developed to address specific issues in neural radiance fields, such as near-field artifacts in forward-facing scenes [S2], memory bottlenecks in spike-based NeRFs [S3, unofficial], and surface compactness in generative occupancy fields [S4, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
[S2] Is Vanilla MLP in Neural Radiance Field Enough for Few-Shot View Synthesis?, https://doi.org/10.1109/cvpr52733.2024.01918, fetched 2026-10-26
[S3] Adaptive Time-step Training for Enhancing Spike-Based Neural Radiance Fields, https://arxiv.org/abs/2507.23033v2, fetched 2026-10-27 (unofficial)
[S4] Generative Occupancy Fields for 3D Surface-Aware Image Synthesis, https://doi.org/10.48550/arxiv.2111.00969, fetched 2026-10-29 (unofficial)
