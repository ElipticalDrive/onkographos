# Pruning and Sparsity Control
Also called: Gaussian sparsity optimization, 3DGS pruning

## Overview
Pruning and Sparsity Control methods reduce the number of active 3D Gaussians in a radiance field to lower memory consumption and storage requirements while maintaining rendering quality [S1, 12710-gaussian-volume-mask.md, unofficial]. Unlike standard 3DGS which relies on opacity thresholds and stops densifying mid-training, these methods actively eliminate redundant primitives throughout the optimization process [12710-gaussian-volume-mask.md]. They are distinguished from sibling methods like [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md) by using explicit geometric or volume-based criteria rather than gradient magnitude alone [14019-visibility-based-pruning-for-3d-gaussian-splatting.md].

## Shared principles
All methods in this family aim to reduce the Gaussian count $N$ by identifying and removing primitives that contribute negligibly to the final rendered image. The core mechanism involves a selection criterion (binary mask, visibility count, or gradient threshold) applied to Gaussian attributes (scale, opacity, position).

1.  **Selection Criterion:** A scalar value or binary flag is computed for each Gaussian $n$ to determine its "essentiality."
2.  **Elimination:** Gaussians failing the criterion are removed from the scene representation.
3.  **Integration:** Pruning is typically integrated into the training loop, either periodically (e.g., at densification intervals) or as a post-training refinement step.
4.  **Loss Regularization:** Many methods introduce a regularization term to the loss function to encourage sparsity or guide the optimizer to shrink unneeded Gaussians before pruning [12710-gaussian-volume-mask.md][14019-visibility-based-pruning-for-3d-gaussian-splatting.md].

## Subtopics

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
A densification strategy that reduces active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter [12710-gaussian-volume-mask.md]. It addresses excessive memory requirements by masking out unessential Gaussians throughout the entire training process, considering both geometric volume and transparency [12710-gaussian-volume-mask.md]. Unlike standard 3DGS, it does not stop densifying in the middle of training [12710-gaussian-volume-mask.md]. Choose this method when consistent sparsity control via learnable parameters is desired.

### [Visibility-Based Pruning for 3D Gaussian Splatting](14019-visibility-based-pruning-for-3d-gaussian-splatting.md)
Removes occluded 3D Gaussian kernels during optimization to create a compact, 'hard' geometry representation by concentrating alpha values into remaining kernels [14019-visibility-based-pruning-for-3d-gaussian-splatting.md]. It distinguishes itself by using explicit geometric occlusion checks (depth sorting) rather than gradient magnitude thresholds [14019-visibility-based-pruning-for-3d-gaussian-splatting.md]. Choose this method when geometric accuracy and removal of truly occluded primitives is prioritized.

### Fine-Tuning and Iterative Pruning
A post-training refinement technique that combines binary masks from blockwise quadratic unconstrained binary optimization (QUBO) solvers to retain a specific number of Gaussians [S1, unofficial]. It fine-tunes the retained Gaussians with densification disabled using a weighted loss of $L_{refine} = 0.8 L_1(I, I_{gt}) + 0.2 (1 - \text{SSIM}(I, I_{gt}))$ [S1, unofficial]. For iterative pruning, the process of partitioning, opacity response computation, budget allocation, and quadratic selection is repeated on the refined model until the target retention ratio is reached [S1, unofficial]. Choose this method when a strict, pre-defined number of Gaussians is required for deployment.

## Background
The proliferation of Gaussians in 3D Gaussian Splatting (3DGS) leads to high memory and storage costs, often including redundant primitives with negligible volume or transparency [12710-gaussian-volume-mask.md]. Standard 3DGS uses opacity-based pruning which may leave many redundant Gaussians [12710-gaussian-volume-mask.md]. Recent works like SplatQ [S1, unofficial] and visibility-based pruning [14019-visibility-based-pruning-for-3d-gaussian-splatting.md] address these inefficiencies. SplatQ evaluates performance on 27 scenes from 5 benchmarks (Mip-NeRF 360, Deep Blending, Tanks and Temples, NeRF Synthetic, Ref-NeRF/Shiny Blender) [S1, unofficial].

## Sources
[S1] SplatQ: Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection, https://arxiv.org/abs/2610.13503, fetched 2026-10-29 (unofficial)
