# Hu et al. Density-based Sampling
Also called: Density-based Sampling
## Overview
A hierarchical sampling strategy for neural radiance fields that optimizes computational cost by restricting the evaluation of the fine-grained network to regions of significant geometric presence. It differs from standard stratified sampling by skipping empty space entirely and from sibling methods like [Hierarchical Density Adaptation](4868-hierarchical-density-adaptation.md) by using a fixed density threshold rather than learned importance weights to determine sampling locations.
## How it works
### Inputs and outputs
- **Inputs**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, coarse network $N_c$, fine network $N_f$, density threshold $\tau$.
- **Outputs**: Final color and depth for the ray.

### Data structures
- **Valid points**: 3D coordinates $\mathbf{x}$ where the coarse network output density $\sigma_c(\mathbf{x}) > 0$.
- **Pivotal points**: 3D coordinates $\mathbf{x}$ where $\sigma_c(\mathbf{x}) > \tau$.

### Steps
1. **Coarse Sampling**: Sample points $\mathbf{x}_i$ along the ray using a coarse strategy (e.g., uniform or stratified).
2. **Coarse Evaluation**: Evaluate the coarse network $N_c(\mathbf{x}_i, \mathbf{d})$ to obtain density $\sigma_c$.
3. **Filtering**:
   - Discard points where $\sigma_c \le 0$ (empty space).
   - Identify **pivotal points** where $\sigma_c > \tau$.
4. **Fine Sampling**:
   - For pivotal points, generate additional fine-grained samples in the vicinity (or use the pivotal points themselves if the architecture allows direct evaluation).
   - Evaluate the fine network $N_f$ only at these pivotal locations.
   - Non-pivotal valid points ($0 < \sigma_c \le \tau$) are typically skipped for fine evaluation or handled by the coarse network alone, depending on the specific implementation of the volume rendering integral.
5. **Rendering**: Composite the color and depth using the densities and colors from the evaluated points.

### Parameters
- $\tau$: Density threshold. Typical values are not explicitly given in the sources, but it must be $> 0$. [G] In practice, this is often a small positive value (e.g., $10^{-3}$ to $10^{-2}$) to filter out noise while retaining surface details.

### Edge cases
- **Zero density**: Points with $\sigma_c = 0$ are ignored, preventing unnecessary computation in empty space.
- **Threshold selection**: If $\tau$ is too high, fine details may be missed; if too low, the computational benefit diminishes.

## Subtopics

### Density-based Region Proposal
A variant applied to 3D object detection in NeRFs. It uses the NeRF density field to propose regions of interest. It is distinguished by the finding that using density alone is superior to using view-dependent radiance for this task, as radiance information is not necessary or beneficial for region proposal confidence. [S2]

## Operating characteristics
- **Time Complexity**: Reduced compared to full fine sampling. The fine network is evaluated only at pivotal points, which are a subset of all valid points.
- **Memory Complexity**: Similar to standard NeRF, as it does not require storing additional large structures, but reduces the number of forward passes through the fine network.
- **Strengths**: Significant speedup in scenes with large empty volumes.
- **Limitations**: Requires a well-calibrated threshold $\tau$. Performance may degrade if the coarse network's density estimates are inaccurate.
- **When to use**: Scenes with significant empty space or where computational resources are limited.
- **When not to use**: Scenes where fine details are distributed uniformly or where the coarse network fails to accurately predict density.

## Background
Proposed by Hu et al. [HLC∗ 22] in the context of constructing generalizable radiance fields. The method addresses the inefficiency of evaluating the fine network at all points, observing that many 3D points have zero density and do not contribute to geometry or radiance. [S1] It is related to work by Shao et al. [SZZ∗ 22] who use SDF fields for surface sampling, and Wan et al. [WRB∗ 23] who use volume density or SDF estimates for surface localization. [S1]

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] NeRF-RPN: A general framework for object detection in NeRFs, https://doi.org/10.1109/cvpr52729.2023.02253, fetched 2026-10-26
