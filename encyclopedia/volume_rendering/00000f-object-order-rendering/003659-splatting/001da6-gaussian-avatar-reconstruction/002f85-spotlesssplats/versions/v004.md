# SpotlessSplats
## Overview
SpotlessSplats is a robust Gaussian Avatar Reconstruction method designed to improve reconstruction quality in casual captures by ignoring transient distractors [S1][S2, unofficial]. It enhances the standard Gaussian Splatting pipeline by detecting outliers in a richer, pre-trained feature space, thereby increasing robustness against noise and occlusions [S1][S2, unofficial]. It builds on [Gaussian Avatar Reconstruction](1575-gaussian-avatar-reconstruction.md) by adding a feature-based outlier rejection mechanism.

## How it works
Inputs: A set of posed images of a dynamic subject, a parametric body model (e.g., SMPL), and a pre-trained feature extractor.
Outputs: A set of pose-conditioned 3D Gaussians representing the avatar, with transient distractors ignored.

Data Structures:
- Gaussian Primitives: Standard 3D Gaussian parameters (mean $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients $c$).
- Feature Embeddings: High-dimensional feature vectors extracted from the pre-trained network for each Gaussian or pixel.

Steps:
1. **Initialization**: Initialize Gaussians on the parametric mesh surface, conditioned on pose and shape parameters [S1][S2, unofficial].
2. **Feature Extraction**: For each rendered pixel or Gaussian, extract a feature embedding using a pre-trained network. This space is "richer" than raw RGB, allowing better discrimination of semantic content [S1][S2, unofficial].
3. **Outlier Detection**: Compare the extracted feature embeddings against expected values or cluster centers in the pre-trained feature space. Identify outliers that correspond to transient distractors (e.g., moving background elements, temporary occlusions) [S1][S2, unofficial].
4. **Robust Optimization**: During differentiable rendering optimization, mask out or down-weight the gradients from detected outliers. This prevents transient distractors from corrupting the Gaussian parameters [S1][S2, unofficial].
5. **Deformation Update**: Update the pose-conditioned deformation network weights using the remaining inlier gradients [S1][S2, unofficial].

Parameters:
- Feature Space Dimension: Not given in the sources.
- Outlier Threshold: Not given in the sources.

Edge Cases:
- If the pre-trained feature space does not generalize to the specific subject or lighting, outlier detection may fail.
- Precision issues in feature comparison may lead to false positives (removing valid geometry) or false negatives (keeping distractors).

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Improves reconstruction quality in casual captures [S1][S2, unofficial].
Strengths: Enhanced robustness to transient distractors; better quality in uncontrolled environments.
Limitations: Requires a pre-trained feature extractor; increased computational overhead for feature extraction and comparison.
When to use: Casual captures with potential transient occlusions or distractors.
When not to use: Controlled studio environments with clean data; real-time applications where feature extraction latency is prohibitive.

## Background
Origin: SpotlessSplats was introduced to enhance robustness in Gaussian Avatar Reconstruction by leveraging pre-trained feature spaces for outlier detection [S1][S2, unofficial].
History: It follows SWAG, which used unsupervised transient Gaussians, by moving to a richer feature space for detection [S1][S2, unofficial].
Usage: Used in dynamic human avatar reconstruction from casual video captures.

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
