# Ray-Centric Adaptive Step and Termination

Also called: Adaptive Ray Marching, Adaptive Step Size Control

## Overview
Ray-Centric Adaptive Step and Termination is a family of volume rendering acceleration techniques that dynamically adjust the sampling step size $\Delta t$ and ray termination conditions based on local data metrics or geometric properties, rather than using a fixed uniform grid. It solves the inefficiency of uniform sampling, which wastes queries in homogeneous regions and fails to capture high-frequency details without excessive cost [S6, S7, S8]. Unlike sibling methods that focus on spatial subdivision or global resolution scaling, this family modulates the ray's internal progression directly, often building on [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md) by implementing specific adaptation laws for step size and opacity thresholds.

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
Adaptive sampling is a volume rendering optimization that varies the sampling density along rays or across the image plane to reduce computational cost while maintaining visual fidelity. It solves the inefficiency of uniform sampling, which wastes queries in homogeneous or empty regions and may alias in high-frequency areas. Unlike its siblings, [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md) focuses on latency constraints and [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md) focuses on pre-integration table lookups, this method modulates the step size $\Delta t$ or sample count based on local data metrics (opacity, gradient, variance).

### [Adaptive Sampling Step with Early Ray Termination](4573-adaptive-sampling-step-with-early-ray-termination.md)
This method optimizes volume rendering performance by reducing sampling density for distant data and terminating ray evaluation once total opacity is reached. It addresses the inefficiency of uniform sampling in large, extended volumes where distant regions require less precision. Unlike its siblings, which often focus on error metrics or interaction latency, this approach relies on geometric distance from the viewpoint and opacity accumulation thresholds. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by implementing specific distance-based step adaptation and brick-level opacity checks.

### [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md)
Adaptive Sampling for Interaction is a technique to maintain smooth user interaction with large volume datasets by dynamically adjusting the rendering step size based on framerate, and performing a high-quality pass after interaction stops. It lowers the sampling rate during user interaction to maintain performance, then re-renders at a higher rate once the action is completed. Unlike static resolution scaling, it is time-bound and context-aware, specifically designed to decouple performance from user input latency. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md).

### [Loss-based Adaptive Sampling](6928-loss-based-adaptive-sampling.md)
Loss-based Adaptive Sampling is a ray sampling strategy that allocates more samples to image regions with higher rendering loss, identified by partitioning the image into a grid and computing mean loss per region. It differs from random ray sampling, which captures global structures, by prioritizing local structures and areas of inadequate convergence.

### [Adaptive Sampling Rate Control](8896-adaptive-sampling-rate-control.md)
Adaptive Sampling Rate Control modulates the number of samples or slices used to render volumetric geometry to balance image quality and performance. It solves the problem of undersampling artifacts during zoom or anisotropic data distortion by dynamically adjusting the sampling density based on view direction and data characteristics. Unlike fixed-rate methods, it allows for non-uniform sampling along data dimensions to compensate for anisotropy.

### [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md)
Adaptive Preintegration Termination Criteria is a method for determining when to stop subdividing a ray segment during volume rendering with preintegration. It solves the problem of balancing rendering accuracy against computational cost by recursively splitting intervals only when the change in the accumulated volume integral exceeds a threshold. It differs from fixed-step preintegration by dynamically adjusting the sampling density based on the local complexity of the scalar field and transfer function, ensuring that isosurfaces and high-frequency details are captured without unnecessary queries.

### Adaptive Sampling with Uniform Fallback
**Definition**
Samples points along rays using adaptive sampling via nerfacc, with a fallback to uniform sampling for rays that receive too few samples [S1, unofficial].

**Steps**
1.  Compute $t_{near}$ and $t_{far}$ using `ray_aabb_intersect`. Filter valid rays where $t_{far} > t_{near}$ [S1, unofficial].
2.  If no valid rays, return empty tensors for $t_s, t_e, ray\_idx$, and `valid` [S1, unofficial].
3.  Call `self.nerfacc_estimator.sampling` with valid `rays_o`, `rays_d`, $t_{near}$, $t_{far}$, and `self.step_size` to get `ray_idx`, $t_s$, $t_e$ [S1, unofficial].
4.  Convert `ray_idx` to `torch.long` [S1, unofficial].
5.  If `self.enable_fallback` is True, count samples per ray using `torch.bincount(ray_idx, minlength=Nv)` [S1, unofficial].
6.  Identify rays with counts < `self.fallback_when_samples` as `low_ids` [S1, unofficial].
7.  For low rays, compute uniform samples: $t_{s\_fb} = t_0[:, None] + k \cdot dt$, $t_{e\_fb} = t_{s\_fb} + dt$, where $k$ is `range(K)`, $dt = (t_1-t_0)/K$, and clamp final $t_{e\_fb}$ to $t_1$ [S1, unofficial].
8.  Construct `ray_idx_fb` for the fallback samples [S1, unofficial].
9.  Concatenate nerfacc samples with fallback samples (implementation detail cut off in text, but implies combining $t_s, t_e, ray\_idx$ with $t_{s\_fb}, t_{e\_fb}, ray\_idx\_fb$) [S1, unofficial].

## Background
The concept of adaptive sampling in volume rendering originated with the need to handle the vast dynamic range of scalar fields in medical and scientific visualization. Early methods focused on opacity-based early termination to skip empty space [S6]. Gradient-based adaptive stepping was introduced to preserve high-frequency details like isosurfaces without aliasing [S6, S7]. The integration of these techniques with neural rendering (NeRF) has led to loss-based and interaction-aware variants [S1, S2].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
