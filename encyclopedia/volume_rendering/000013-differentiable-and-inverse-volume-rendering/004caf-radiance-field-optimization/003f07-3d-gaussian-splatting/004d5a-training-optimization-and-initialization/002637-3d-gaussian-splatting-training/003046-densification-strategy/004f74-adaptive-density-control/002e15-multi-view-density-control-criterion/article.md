# Multi-View Density Control Criterion

Also called: Absolute-gradient split proxy, FastGS criterion, Multi-view Adaptive Density Control (ADC)

## Overview
A densification strategy for multi-view training that resolves invalid vector arithmetic by computing error metrics using norms and sums of norms instead of adding gradient vectors from different view spaces [S1, unofficial]. It sets itself apart from standard Adaptive Density Control (ADC) by ensuring that gradients projected from multiple camera viewpoints into different 2D image planes are aggregated in a space-independent manner, preventing the nullification of gradient signals [S1, unofficial].

## How it works
**Inputs and outputs:**
*   **Input:** A set of $N$ viewpoints (images) used during training. For each Gaussian $G$, the 2D positional gradients $\nabla_{\pi} L$ at all pixels $\pi$ that project onto $G$ across all viewpoints.
*   **Output:** A scalar error metric $E_1(G)$ used to decide whether to split or clone the Gaussian.

**Data structures:**
*   No new data structures are required beyond the standard Gaussian attributes and gradient buffers. The method modifies the aggregation logic of the existing gradient accumulation step.

**Numbered steps:**
1.  For each Gaussian $G$ in the scene, initialize the error accumulator $E_1(G) = 0$.
2.  Iterate over all $N$ viewpoints in the current training batch.
3.  For each viewpoint, identify all pixels $\pi$ that are covered by the projection of Gaussian $G$.
4.  Retrieve the 2D positional gradient $\nabla_{\pi} L$ for each covered pixel. Note that these gradients exist in different 2D coordinate spaces corresponding to their respective image planes.
5.  Compute the squared Euclidean norm of each gradient vector: $\|\nabla_{\pi} L\|^2$.
6.  Sum these squared norms across all pixels and all viewpoints to update the error metric:
    $$ E_1(G) = \sum_{k=1}^{N} \sum_{\pi_i = k} \|\nabla_{\pi_i} L\|^2 $$
    where $N$ is the number of viewpoints, $\pi_i$ is a pixel in viewpoint $k$, and $\nabla_{\pi_i} L$ is the gradient at pixel $\pi_i$ [S1, unofficial].
7.  Compare $E_1(G)$ against the densification thresholds (clone threshold, split threshold) defined in the parent [Adaptive Density Control](9188-adaptive-density-control.md).

**Parameters:**
*   **DSTH (DS Threshold):** A hyperparameter controlling the strictness of densification constraints.
    *   Default value: 30 [S3, unofficial].
    *   Specific scene value: 60 for the "playroom" scene in the DeepBlending dataset [S3, unofficial].
    *   Effect: Increasing DSTH imposes stricter constraints, reducing the number of Gaussians (NGS), rendering quality, and training time [S3, unofficial].

**Pitfalls:**
*   **Invalid Vector Addition:** Standard ADC adds 2D positional gradients directly. In multi-view training, a Gaussian is projected to multiple image planes. Adding 2D positional gradients from different image planes (different spaces) is mathematically invalid [S1, unofficial].
*   **Nullification:** This invalid arithmetic can result in zero vectors even if the underlying 3D gradients are identical and large, leading to a failure to densify where it is needed [S1, unofficial]. The Multi-View Density Control Criterion avoids this by using norms (scalars) which are space-independent.

## Subtopics

### DS Threshold (DSTH)
A hyperparameter threshold that controls the densification constraints of Gaussians [S3, unofficial]. Increasing it imposes stricter constraints, reducing the number of Gaussians (NGS), rendering quality, and training time [S3, unofficial]. The default value is 30, but it is set to 60 for the playroom scene in the DeepBlending dataset, while other scenes use 30 [S3, unofficial].

## Operating characteristics
*   **Performance:** Yields an average improvement of 0.67 dB in PSNR and 0.028 in LPIPS compared to standard methods, while using 28% fewer primitives at the same time [S2, unofficial].
*   **Efficiency:** On the "garden" scene, it produces 148,658 primitives compared to 240,075 with standard positional-gradient rules, achieving a higher PSNR (27.06 vs 26.83) and lower LPIPS (0.1247 vs 0.1303) [S2, unofficial].
*   **Usage:** Applied during the teacher training phase to control primitive count and distribution [S2, unofficial].
*   **Strengths:** Resolves the gradient nullification issue in multi-view settings; reduces memory footprint (fewer primitives) while improving or maintaining reconstruction quality.
*   **Limitations:** Requires careful tuning of DSTH for specific scenes (e.g., DeepBlending playroom) to avoid over-constraining the density [S3, unofficial].

## Background
The method was proposed to address the incompatibility of standard 3D Gaussian Splatting densification with efficient multi-view training [S1, unofficial]. Standard ADC relies on summing 2D gradients, which fails when gradients come from different camera views [S1, unofficial]. The Multi-View Density Control Criterion was introduced in "Efficient multi-view training for 3D Gaussian Splatting" [S1, unofficial] as a space-independent error metric. It has since been utilized in subsequent works such as "Bake It Till You Make It" [S2, unofficial] and "Loss-Guided Density Control" [S3, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
[S2] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S3] Loss-Guided Density Control and Efficient Backward Optimisation for Compact Gaussian Splatting, https://doi.org/10.21203/rs.3.rs-10572680/v1, fetched 2026-10-26 (unofficial)
