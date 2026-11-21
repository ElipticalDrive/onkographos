# SemGauss-SLAM

Also called: SGS-SLAM, NEDS-SLAM

## Overview
SemGauss-SLAM is a simultaneous localization and mapping (SLAM) method that integrates semantic information into the 3D Gaussian Splatting framework [S1]. It distinguishes itself from standard radiance-field optimization by distilling 2D semantic labels to guide the Gaussian representation, enabling semantic-aware mapping [S2, unofficial].

## How it works
Inputs: A sequence of RGB-D images (or RGB images with depth estimation), camera poses (initially estimated via visual odometry), and 2D semantic segmentation masks [S1].
Outputs: A set of 3D Gaussians with semantic attributes, optimized camera poses, and a semantic map [S2, unofficial].

Data Structures:
- 3D Gaussians: Each Gaussian $i$ is defined by position $\mu_i$, covariance $\Sigma_i$ (or rotation $R_i$ and scale $s_i$), opacity $\alpha_i$, spherical harmonics coefficients $c_i$, and a semantic label $l_i$ [S1][S2, unofficial].
- Semantic Label: $l_i$ is a discrete class identifier or a probability vector over semantic classes [S2, unofficial].

Steps:
1. Initialization: Initialize Gaussians using visual features or LiDAR points if available [S1].
2. Forward Rendering: Render the scene to obtain RGB and depth images using differentiable rasterization [S1].
3. Semantic Distillation: Obtain 2D semantic masks from a pre-trained 2D segmentation network or dataset annotations [S2, unofficial].
4. Loss Computation: Compute the photometric loss between rendered and input RGB images. Compute a semantic consistency loss between the rendered semantic features and the 2D semantic masks [S2, unofficial].
5. Optimization: Update Gaussian parameters (position, scale, rotation, opacity, color, semantics) and camera poses to minimize the combined loss [S2, unofficial].
6. Adaptive Control: Apply Gaussian splitting and pruning based on opacity and gradient magnitude, potentially guided by semantic boundaries to avoid redundant splitting [S1].

Formulas:
- Total Loss: $\mathcal{L} = \mathcal{L}_{rgb} + \lambda_{sem} \mathcal{L}_{sem}$ [S2, unofficial].
- $\mathcal{L}_{rgb}$: Photometric reconstruction loss [S1].
- $\mathcal{L}_{sem}$: Semantic distillation loss, measuring the discrepancy between projected 2D semantic labels and rendered semantic features [S2, unofficial].
- $\lambda_{sem}$: Weight for the semantic loss [S2, unofficial].

Parameters:
- $\lambda_{sem}$: Not given in the sources.
- Semantic segmentation model: Not specified, but implied to be a standard 2D method [S2, unofficial].

Edge cases and pitfalls:
- Semantic ambiguity in textureless regions may lead to incorrect label propagation [G].
- Misalignment between 2D semantic masks and 3D Gaussians due to depth errors can degrade map quality [G].

## Subtopics
- SGS-SLAM: A variant that considers Gaussian semantic information [S1].
- NEDS-SLAM: Another variant integrating semantic information into SLAM [S1].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Strengths: Enables semantic understanding in the reconstructed map, useful for robotics tasks like navigation and manipulation [S1].
Limitations: Relies on the accuracy of 2D semantic segmentation [S2, unofficial].

## Background
Origin: SemGauss-SLAM, along with SGS-SLAM and NEDS-SLAM, was introduced to enhance 3D Gaussian Splatting-based SLAM by incorporating semantic information [S1].
History: It builds on earlier methods like LIVGaussMap which used LiDAR for initialization [S1].
Usage: Used in robotics for relocalization, navigation, and 6D pose estimation [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
