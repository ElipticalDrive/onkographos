# Ray-Centric Adaptive Step and Termination

Also called: Adaptive Step Size, Adaptive Ray Marching, Adaptive Sampling and Termination

## Overview
Ray-Centric Adaptive Step and Termination is a volume rendering optimization that dynamically adjusts the sampling step size $\Delta t$ and ray termination conditions along individual rays based on local scene complexity or accumulated opacity [S1, S2, S3, unofficial][S6]. Unlike uniform sampling, which casts a constant number of samples per ray regardless of local data density, this method concentrates samples in regions of high detail (e.g., near surfaces or high gradients) and reduces them in homogeneous or empty regions [S1, S4, unofficial][S6]. It sets itself apart from sibling methods like [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) by operating directly on the ray march loop rather than separating coarse and fine networks, and from [Resolution Adaptation](3063-resolution-adaptation.md) by modulating spatial resolution rather than step size.

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

All methods in this family aim to maintain visual fidelity while reducing the total number of volume queries or network evaluations [S1, unofficial][S2].

## Subtopics

### [Adaptive Sampling](1385-adaptive-sampling.md)
Adaptive sampling is a volume rendering optimization that varies the sampling density along rays or across the image plane to reduce computational cost while maintaining visual fidelity. Unlike uniform sampling, which casts a constant number of samples per ray regardless of local data complexity, adaptive sampling concentrates samples in regions of high detail (e.g., near surfaces or high gradients) and reduces them in homogeneous or empty regions. This method builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by explicitly modulating the step size $\Delta t$ or sample count based on local metrics, often using a two-pass or hierarchical approach to balance parallel execution efficiency with accuracy.

### [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md)
Adaptive Sampling for Interaction is a technique designed to maintain real-time interactivity during user input (such as camera movement or transfer function adjustment) by temporarily lowering the volume sampling rate, then restoring high fidelity once the interaction ceases. It differs from spatially adaptive methods (like [Adaptive Sampling](1385-adaptive-sampling.md)) by modulating the global sampling density based on temporal user activity rather than local scene complexity, and from [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md) by focusing on the trade-off between frame latency and visual smoothness during dynamic exploration.

### [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md)
Adaptive Preintegration Termination Criteria is a method for determining when to stop subdividing a ray segment during volume rendering with preintegration. It solves the problem of balancing rendering accuracy against computational cost by recursively splitting intervals only when the change in the accumulated volume integral exceeds a threshold. It differs from fixed-step preintegration by dynamically adjusting the sampling density based on the local complexity of the scalar field and transfer function, ensuring that isosurfaces and high-frequency details are captured without unnecessary queries.

### Adaptive Sampling with Uniform Fallback
Samples points along rays using adaptive sampling via nerfacc, with a fallback to uniform sampling for rays that receive too few samples [S1, unofficial]. The process begins by computing $t_{near}$ and $t_{far}$ using `ray_aabb_intersect`. Valid rays are filtered where $t_{far} > t_{near}$. If no valid rays exist, empty tensors for $t_s, t_e, ray\_idx,$ and $valid$ are returned. For valid rays, `self.nerfacc_estimator.sampling` is called with `rays_o`, `rays_d`, $t_{near}$, $t_{far}$, and `self.step_size` to obtain $ray\_idx, t_s, t_e$. The $ray\_idx$ is converted to `torch.long`. If `self.enable_fallback` is True, samples per ray are counted using `torch.bincount(ray_idx, minlength=Nv)`. Rays with counts less than `self.fallback_when_samples` are identified as `low_ids`. For these low-count rays, uniform samples are computed: $t_{s\_fb} = t_0[:, None] + k \cdot dt$, $t_{e\_fb} = t_{s\_fb} + dt$, where $k$ is `range(K)`, $dt = (t_1-t_0)/K$, and the final $t_{e\_fb}$ is clamped to $t_1$. A corresponding `ray_idx_fb` is constructed. The final output concatenates the nerfacc samples with the fallback samples [S1, unofficial].

### Loss-based Adaptive Sampling
A ray sampling strategy that allocates more samples to image regions with higher rendering loss, identified by partitioning the image into an 8x8 grid and computing mean loss per region [S2]. Each input image is partitioned into 64 regions ($R_j, j=1..64$). A uniform distribution of pixels $s_i$ is randomly sampled from input source views. The region losses $H_i[j]$ are normalized into a probability distribution $f_i[j]$. Pixels for the next iteration are resampled by allocating $n_i \cdot f_i[j]$ samples to each region $R_j$, where $n_i$ is the total sample count per image. The loss for region $j$ is defined as $H_i[j] = \frac{1}{|r_j|} \sum_{(u,v) \in r_j} e_i^g[u, v] + e_i^p[u, v]$, where $r_j = s_i \cap R_j$ represents pixels uniformly sampled from region $R_j$, $e_i^g$ is gradient loss, and $e_i^p$ is pixel loss. The probability distribution is $f_i[j] = \frac{\mathcal{H}_i[j]}{\sum_{m=1}^{64} \mathcal{H}_i[m]}$ [S2].

## Background
The concept of adaptive sampling in volume rendering originated from the need to optimize the computational cost of ray marching through dense scalar fields. Early work focused on gradient-based step size adaptation to preserve high-frequency details while skipping empty space [S6]. The introduction of neural radiance fields (NeRF) and implicit representations revived interest in adaptive sampling, particularly for handling the high-dimensional function evaluations required by neural networks [S1, unofficial][S2]. Modern implementations often combine gradient magnitude, opacity accumulation, and loss-based metrics to determine sampling density, balancing the trade-off between rendering speed and visual quality [S1, S2, S3, unofficial].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
