# Pruning and Sparsification

Also called: Gaussian Sparsification, Primitive Removal

## Overview
Pruning and Sparsification refers to the set of techniques used during the training of 3D Gaussian Splatting (and related radiance field representations) to remove low-significance primitives and manage opacity values to prevent unbounded growth, reduce memory footprint, and eliminate artifacts such as floaters. It distinguishes itself from densification strategies by focusing on the removal or reassignment of existing primitives rather than the creation of new ones, often relying on opacity thresholds, gradient-based sensitivity scores, or contribution metrics to determine which primitives are redundant.

## Shared principles
The core mechanism involves periodically evaluating the contribution or error associated with each primitive $g_k \in \mathcal{G}$ and modifying the population $\mathcal{G}$ accordingly. Common principles include:

1.  **Gradient/Error Tracking:** Most methods track a scalar metric per Gaussian, such as the magnitude of the 2D positional gradient $\tau_k = \frac{1}{N_{\text{views}}} \sum_{v} \| \frac{\partial L}{\partial \mu_v^k} \|_2$ [S4, S8, unofficial] or rendering error $E_k$ [S1, unofficial].
2.  **Periodic Execution:** Densification and pruning are not performed at every optimization step to allow the optimizer to stabilize. They are executed every $N$ iterations (e.g., `refineEvery` = 100) [S5, S8, unofficial].
3.  **Splitting and Cloning:** When a Gaussian exceeds a threshold, it is replaced by children. Splitting typically involves sampling new means from a normal distribution scaled by the parent's covariance, while cloning duplicates the parent [S8, unofficial].
4.  **Pruning:** Gaussians with low opacity $\alpha_k < \alpha_{\min}$ or low significance are removed to reduce memory footprint and prevent floaters [S1, unofficial][S2].
5.  **Buffer Management:** Pruned or split Gaussians require buffer compaction or flag-based marking to maintain contiguous memory access for rasterization [Parent].

## Subtopics

### [Pruning and Opacity Management](9314-pruning-and-opacity-management.md)
Pruning and Opacity Management refers to the set of techniques used during the training of 3D Gaussian Splatting (and related radiance field representations) to remove low-significance primitives and manage opacity values to prevent unbounded growth, reduce memory footprint, and eliminate artifacts such as floaters. It distinguishes itself from densification strategies by focusing on the removal or reassignment of existing primitives rather than the creation of new ones, often relying on opacity thresholds, gradient-based sensitivity scores, or contribution metrics to determine which primitives are redundant.

### [Significance-Aware Pruning](9676-significance-aware-pruning.md)
Significance-Aware Pruning is a method for removing redundant 3D Gaussians during training by evaluating their actual contribution to the rendered image via accumulated alpha blending coefficients, rather than relying solely on opacity or spatial size. It solves the problem where standard opacity pruning removes large, structurally important Gaussians with low opacity, and size pruning causes an explosion in Gaussian count as large primitives are replaced by many small ones. Unlike standard opacity pruning, which ignores spatial extent and transmittance, this method spares Gaussians that have low opacity but high significance to the scene reconstruction.

### [Principled Uncertainty Pruning](9680-principled-uncertainty-pruning.md)
Principled Uncertainty Pruning (PUP) is a post-hoc pruning technique that removes redundant 3D Gaussians from a pretrained 3D Gaussian Splatting (3D-GS) model by assessing their sensitivity to the reconstruction error using a Fisher Information-based approximation of the Hessian. Unlike heuristic methods or those relying on visibility scores, PUP computes a mathematically principled uncertainty score based on the spatial parameters (mean and scale) of each Gaussian, enabling the removal of up to 90% of primitives while preserving fine foreground details and achieving higher image quality metrics than alternatives like LightGaussian. It operates via a multi-round prune-refine pipeline that can be applied to any converged 3D-GS model without modifying its original training pipeline.

### GaussianSPA
- [S1, unofficial] Proposes an optimizing-sparsifying loop that interleaves standard training with a sparsification operator.
- [S1, unofficial] Removes redundant primitives while boosting the influence of the remaining collection.

### Depth-Based Gaussian Pruning
- [S2] A pruning strategy that varies the pruning threshold based on the relative depth (z-axis value) of 3D Gaussians to preserve points at the far plane in forward-facing scenes.
- [S2] Determine the relative depth for each 3D Gaussian based on its z-axis value.
- [S2] Scale the base pruning threshold $t_p$ by a multiplier $w_p$ determined by the relative depth. The lowest threshold is applied to the farthest point.
- [S2] $t_p$: The base 3D Gaussian pruning threshold. Value: $5 \times 10^{-3}$ for real defocus blur dataset; $1 \times 10^{-2}$ for real camera motion blur dataset.
- [S2] $w_p$: The pruning threshold multiplier based on depth. Value: 3.

## Background
The concept of adaptive density control, including pruning, was introduced in the original 3D Gaussian Splatting paper [14] (cited in [S2]) to manage the number of 3D Gaussians through periodic densification and pruning. Subsequent research has expanded on this foundation. For instance, [S1, unofficial] surveys various sparsification techniques, including GaussianSPA [86], which interleaves training with sparsification, and Speedy-Splat [85], which applies primitive-sparse selection. [S2] introduces depth-based pruning specifically for deblurring tasks in forward-facing scenes to compensate for sparsity at the far end of the scene.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
