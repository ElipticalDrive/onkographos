# Smoothness Priors
Also called: Geometric Smoothness Regularization, Gradient Smoothness Loss, SDF Total Variation Loss
## Overview
Smoothness priors are regularization terms applied during training and inference to reduce surface noise and improve geometric smoothness in neural surface reconstruction [S1, unofficial]. They distinguish themselves from sibling methods by targeting the geometric consistency of the signed distance field (SDF) and its gradients, rather than just radiance or color consistency.
## Shared principles
The family relies on two primary mechanisms to enforce smoothness:
1.  **3D Gaussian Convolution:** Applying a 3D convolution with a Gaussian kernel to the geometry features or SDF grid. This is typically used during the coarse training stage and/or as a post-processing step [S1, unofficial].
2.  **Gradient and SDF Regularization:** Applying a gradient smoothness loss during fine training and an SDF total variation (TV) loss on the SDF voxel grid [S1, unofficial].

The effectiveness of these priors is critical; removing the Gaussian kernel during coarse training has been shown to increase Chamfer Distance (CD) error from 0.72 to 0.74 [S1, unofficial]. Conversely, removing it during post-processing slightly reduces CD from 0.720 to 0.715, suggesting a nuanced role in final refinement [S1, unofficial].
## Subtopics

### [Voxurf Smoothness](929-voxurf-smoothness.md)
The specific implementation in Voxurf, combining 3D Gaussian convolution for the coarse stage with gradient smoothness and SDF TV losses for the fine stage.

## Background
The concept originates from the need to stabilize neural surface reconstruction, particularly in voxel-based methods like Voxurf [S1, unofficial]. The specific ablation studies regarding the impact of Gaussian kernels and gradient losses were detailed in the supplementary materials of the Voxurf paper [S1, unofficial].
## Sources
[S1] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
