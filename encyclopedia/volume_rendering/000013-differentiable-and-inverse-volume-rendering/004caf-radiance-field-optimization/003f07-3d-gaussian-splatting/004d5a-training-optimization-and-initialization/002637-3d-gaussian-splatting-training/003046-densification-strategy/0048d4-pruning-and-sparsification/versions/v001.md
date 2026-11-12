# Pruning and Sparsification

Also called: Gaussian Pruning, Model Compression

## Overview
Pruning and Sparsification are post-training or in-training strategies that reduce the number of 3D Gaussians in a scene to improve rendering efficiency, reduce memory footprint, or correct reconstruction artifacts. Unlike [Densification Strategy](6234-densification-strategy.md), which increases density to capture detail, this family removes redundant or insignificant primitives. It is distinguished from sibling methods by its focus on removing Gaussians based on specific criteria such as accumulated rendering significance, second-order uncertainty (Fisher Information), or spatial depth, rather than simple opacity thresholds or gradient norms alone.

## Shared principles
All pruning methods in this family operate on a set of Gaussians $\mathcal{G} = \{g_1, \dots, g_n\}$ and remove a subset $G_{prune} \subset \mathcal{G}$ to produce a pruned set $\mathcal{G}'$. The shared mechanisms include:

1.  **Scoring Metric:** Each Gaussian $g_i$ is assigned a scalar score $S_i$ representing its "redundancy" or "insignificance." Lower scores indicate higher candidates for removal. Common metrics include:
    *   **Accumulated Significance:** The sum of alpha blending coefficients $w_i^p$ over all pixels $p$ and views [Child 9676].
    *   **Uncertainty/Sensitivity:** The log-determinant of the Fisher Information matrix block corresponding to the Gaussian's spatial parameters [Child 9680].
    *   **Opacity:** A simple threshold $o_{min}$ below which Gaussians are considered candidates [Child 9676].
2.  **Selection:** Gaussians are sorted by their score $S_i$ (ascending). The bottom $N_{prune}$ Gaussians are selected for removal.
3.  **Removal:** Selected Gaussians are deleted from the active set. The data structure (e.g., GPU buffer) is compacted to maintain contiguous memory layout [Parent].
4.  **Refinement (Optional):** Some methods employ a prune-refine pipeline where the remaining Gaussians are fine-tuned via gradient descent to recover quality lost during pruning [Child 9680].

## Subtopics

### [Significance-Aware Pruning](9676-significance-aware-pruning.md)
[Significance-Aware Pruning](9676-significance-aware-pruning.md) removes Gaussians based on their accumulated contribution to the rendered image, solving the problem where standard opacity pruning removes large, structurally significant Gaussians that have low opacity but high transmittance. It differs from [Principled Uncertainty Pruning](9680-principled-uncertainty-pruning.md) by using alpha blending coefficients rather than second-order derivatives. It is chosen when computational efficiency is prioritized over theoretical optimality and when standard opacity pruning leads to structural artifacts.

### [Principled Uncertainty Pruning](9680-principled-uncertainty-pruning.md)
[Principled Uncertainty Pruning](9680-principled-uncertainty-pruning.md) is a post-hoc compression method that prunes Gaussians based on a sensitivity score derived from the Fisher Information matrix. It solves the problem of aggressive compression by quantifying the uncertainty of spatial parameters, allowing for up to 90% pruning while preserving visual fidelity. It differs from heuristic methods by using a second-order approximation of the reconstruction error. It is chosen when maximum model compression is required without significant quality loss, assuming the model is fully converged.

### Depth-Based Gaussian Pruning
A minor variant that varies the pruning threshold based on the relative depth (z-axis value) of 3D Gaussians to preserve points at the far plane in forward-facing scenes [S1].

**Definition:**
The pruning threshold $t_p$ is scaled by a multiplier $w_p$ determined by the relative depth of each Gaussian [S1]. The lowest threshold is applied to the farthest point to prevent over-pruning of distant geometry [S1].

**Steps:**
1.  Determine the relative depth for each 3D Gaussian based on its z-axis value [S1].
2.  Scale the base pruning threshold $t_p$ by a multiplier $w_p$ determined by the relative depth [S1].

**Parameters:**
*   $t_p$: The base 3D Gaussian pruning threshold. Value: $5 \times 10^{-3}$ for real defocus blur dataset; $1 \times 10^{-2}$ for real camera motion blur dataset [S1].
*   $w_p$: The pruning threshold multiplier based on depth. Value: 3 [S1].

## Background
Pruning in 3D Gaussian Splatting originated as a component of the adaptive density control mechanism in the original 3D-GS paper [14], where it was used to remove low-opacity Gaussians during training. Later works, such as Deblurring 3D Gaussian Splatting [S1], introduced depth-aware pruning to handle specific artifacts in forward-facing scenes. More recently, Principled Uncertainty Pruning [S1] adapted techniques from neural network pruning (Fisher Information) to 3D-GS for post-training compression.

## Sources
[S1] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
