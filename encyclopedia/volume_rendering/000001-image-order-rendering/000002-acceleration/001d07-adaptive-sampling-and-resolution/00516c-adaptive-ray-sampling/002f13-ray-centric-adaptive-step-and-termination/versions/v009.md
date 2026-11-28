# Ray-Centric Adaptive Step and Termination

Also called: Adaptive Ray Marching, Adaptive Step Size Control, Early Ray Termination (ERT)

## Overview
Ray-Centric Adaptive Step and Termination optimizes volume rendering by dynamically adjusting the sampling density along individual rays and terminating evaluation when opacity thresholds are met, rather than using fixed uniform steps. It solves the inefficiency of uniform sampling, which wastes queries in empty regions and oversamples homogeneous areas, while ensuring visual fidelity in high-frequency regions. Unlike sibling methods that focus on global resolution scaling or random pixel subsampling, this family adapts parameters locally per ray based on metrics such as opacity, gradient magnitude, or distance from the viewpoint.

## Shared principles
The core mechanism shared by all children is the deviation from a fixed, uniform sampling strategy. Instead of casting a constant number of rays or samples per pixel/ray, the method computes a metric (such as gradient magnitude, rendering error, or temporal difference) and uses it to modulate the sampling rate or resolution.

Mathematically, this often involves a control loop:
1.  **Metric Evaluation:** Compute a scalar value $M$ representing local complexity or error.
2.  **Adaptation:** Adjust a parameter $P$ (step size $\Delta t$, resolution $R$, or sampling probability $w$) based on $M$.
3.  **Rendering/Training:** Execute the volume integral or loss computation with the adjusted parameter.

Common adaptation laws include:
*   **Step Size Adaptation:** $\Delta t \propto \frac{1}{M}$ or $\Delta t = \min(\Delta t_{max}, \max(\Delta t_{min}, \frac{k}{M}))$ [S6, S8, S9, unofficial].
*   **Resolution Control:** $R_{new} = R_{old} + \alpha \cdot (t_{target} - t_{actual})$ or threshold-based switching [S3063, unofficial].
*   **Probability Weighting:** $w_i \propto M_i$ for importance sampling [S4696, unofficial].

All methods in this family aim to maintain visual fidelity while reducing the total number of volume queries or network evaluations [S1, unofficial][S2, unofficial].

## Subtopics

### [Adaptive Sampling](1385-adaptive-sampling.md)
Modulates step size $\Delta t$ or sample count based on local data metrics (opacity, gradient, variance).

### [Adaptive Sampling Step with Early Ray Termination](4573-adaptive-sampling-step-with-early-ray-termination.md)
Relies on geometric distance from the viewpoint and opacity accumulation thresholds for brick-level checks.

### [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md)
Focuses on latency constraints, adjusting sampling based on framerate during user interaction.

### [Loss-based Adaptive Sampling](6928-loss-based-adaptive-sampling.md)
Allocates samples to image regions with higher rendering loss, identified by partitioning the image into a grid.

### [Adaptive Sampling Rate Control](8896-adaptive-sampling-rate-control.md)
Relies on a discrete, user-selected rate rather than a computed metric, offering a deterministic trade-off.

### [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md)
Recursively splits intervals only when the change in the accumulated volume integral exceeds a threshold.

### Adaptive Sampling with Uniform Fallback
**Definition**
Samples points along rays using adaptive sampling via nerfacc, with a fallback to uniform sampling for rays that receive too few samples [S1, unofficial].

**Steps**
1.  Compute $t_{near}$ and $t_{far}$ using `ray_aabb_intersect`. Filter valid rays where $t_{far} > t_{near}$ [S1, unofficial].
2.  If no valid rays, return empty tensors for $t_s, t_e, ray\_idx$, and valid [S1, unofficial].
3.  Call `self.nerfacc_estimator.sampling` with valid `rays_o`, `rays_d`, $t_{near}$, $t_{far}$, and `self.step_size` to get `ray_idx`, $t_s$, $t_e$ [S1, unofficial].
4.  Convert `ray_idx` to `torch.long` [S1, unofficial].
5.  If `self.enable_fallback` is True, count samples per ray using `torch.bincount(ray_idx, minlength=Nv)` [S1, unofficial].
6.  Identify rays with counts < `self.fallback_when_samples` as `low_ids` [S1, unofficial].
7.  For low rays, compute uniform samples: $t_{s\_fb} = t_0[:, None] + k \cdot dt$, $t_{e\_fb} = t_{s\_fb} + dt$, where $k$ is `range(K)`, $dt = (t_1-t_0)/K$, and clamp final $t_{e\_fb}$ to $t_1$ [S1, unofficial].
8.  Construct `ray_idx_fb` for the fallback samples [S1, unofficial].
9.  Concatenate nerfacc samples with fallback samples [S1, unofficial].

## Background
The concept of adaptive ray marching originated in the 1980s as a means to optimize volume rendering performance by reducing the number of texture lookups in empty or homogeneous regions. Early implementations focused on opacity-based skipping and gradient-driven step sizing to preserve edges. With the advent of neural radiance fields (NeRF) and differentiable rendering, adaptive sampling has evolved to include loss-based strategies and interaction-aware control loops to maintain interactivity in large-scale datasets [S1, unofficial][S2, unofficial].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
