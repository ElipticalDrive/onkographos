# Ray-Centric Adaptive Step and Termination

Also called: Adaptive Ray Marching, Adaptive Step Size Control

## Overview
Ray-Centric Adaptive Step and Termination is a family of volume rendering optimization techniques that dynamically adjust the sampling density along individual rays based on local data properties or accumulated state, rather than using a fixed uniform step size. It solves the problem of computational waste in empty regions and insufficient detail in complex areas by concentrating samples where they are most needed. It sets itself apart from its siblings, such as [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md) (which separates coarse/fine networks) and [Resolution Adaptation](3063-resolution-adaptation.md) (which changes image resolution), by operating directly on the ray parameterization ($\Delta t$) and termination conditions within the volume integral, often requiring specific hardware-aware quantization to maintain GPU performance [S7][S8].

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
Adaptive sampling is a volume rendering optimization that varies the sampling density along rays to reduce computational cost while preserving visual quality. Unlike uniform sampling, which treats all regions equally, adaptive sampling concentrates samples in high-detail or high-opacity areas and skips empty space. It sets itself apart from its parent method, [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md), by explicitly addressing GPU hardware constraints (warp divergence and texture cache coherence) through quantization and block-level metrics, rather than relying solely on continuous per-pixel step adjustments.

### [Adaptive Sampling for Interaction](5643-adaptive-sampling-for-interaction.md)
Adaptive Sampling for Interaction is a performance optimization strategy that dynamically reduces the volume sampling rate during user interaction (such as camera movement or transfer function adjustment) to maintain interactive frame rates, then restores the high sampling rate once the interaction ceases. It differs from static resolution scaling by being context-aware and time-bound, specifically decoupling performance from user input latency. Unlike [Adaptive Sampling](1385-adaptive-sampling.md), which adapts to data complexity, this method adapts to user state. It builds on [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by applying temporal adaptation to the sampling density rather than just step size.

### [Adaptive Preintegration Termination Criteria](9804-adaptive-preintegration-termination-criteria.md)
Adaptive Preintegration Termination Criteria is a method for determining when to stop subdividing a ray segment during volume rendering, specifically within the context of preintegration. It solves the problem of balancing rendering accuracy against computational cost by recursively splitting intervals only when the estimated error exceeds a threshold. It sets itself apart from standard adaptive sampling [1385-adaptive-sampling.md] by using the difference between two precomputed integral approximations ($C_1$ and $C_2$) as the error metric, rather than just local density or gradient magnitude.

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

### Loss-based Adaptive Sampling
**Definition**
A ray sampling strategy that allocates more samples to image regions with higher rendering loss, identified by partitioning the image into an 8x8 grid and computing mean loss per region [S2].

**Steps**
1.  Partition each input image into a grid of 64 regions ($R_j, j=1..64$) [S2].
2.  Randomly sample a uniform distribution of pixels $s_i$ from the input source views [S2].
3.  Normalize the region losses $H_i[j]$ into a probability distribution $f_i[j]$ [S2].
4.  Resample pixels for the next iteration by allocating $n_i \cdot f_i[j]$ samples to each region $R_j$, where $n_i$ is the total sample count per image [S2].

**Formulas**
$$ H_i[j] = \frac{1}{|r_j|} \sum_{(u,v) \in r_j} e_i^g[u, v] + e_i^p[u, v] $$
where $r_j = s_i \cap R_j$ represents pixels uniformly sampled from region $R_j$, $e_i^g$ is gradient loss, and $e_i^p$ is pixel loss [S2].

$$ f_i[j] = \frac{\mathcal{H}_i[j]}{\sum_{m=1}^{64} \mathcal{H}_i[m]} $$
where the denominator sums the loss statistics over all 64 regions [S2].

**Parameters**
*   Grid size: 8x8, resulting in 64 regions ($j=1..64$) [S2].
*   $n_i$: Total sample count per image used for resampling allocation [S2].

## Background
The concept of adaptive sampling in volume rendering originated with the need to handle the vast dynamic range of density values in scientific visualization, where large volumes of empty space (e.g., atmosphere, vacuum) dominate the computational budget while thin structures (e.g., isosurfaces, clouds) require high resolution [S6, S10, unofficial]. Early methods focused on gradient-based step sizing to ensure isosurface intersections were not missed [S7]. With the advent of neural radiance fields (NeRF), adaptive sampling evolved to include loss-based strategies that prioritize training samples in regions of high reconstruction error, as seen in [Efficient ray sampling for radiance fields reconstruction](S2) and [CoroNeRF](S1, unofficial). The family also includes interaction-aware variants designed for real-time user feedback [S1, unofficial][S2].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
