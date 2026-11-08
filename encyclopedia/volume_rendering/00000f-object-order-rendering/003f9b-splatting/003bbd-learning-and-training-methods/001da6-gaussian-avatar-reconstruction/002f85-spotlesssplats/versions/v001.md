# SpotlessSplats


## Overview
SpotlessSplats is a robustness-enhancing variant of Gaussian Avatar Reconstruction designed to improve reconstruction quality in casual captures by ignoring transient distractors [S1][S2, unofficial]. It distinguishes itself from sibling methods like SWAG by detecting outliers in a richer, pre-trained feature space rather than relying solely on learned appearance embeddings or unsupervised transient Gaussian training [S1][S2, unofficial].

## How it works
Inputs: A set of 3D Gaussians (position, covariance, opacity, spherical harmonics coefficients) and a collection of input images with known camera poses.
Outputs: An optimized set of 3D Gaussians representing the static scene or avatar, excluding transient occluders.

Data Structures:
- Gaussian $i$: Position $\mathbf{p}_i \in \mathbb{R}^3$, Covariance $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (or quaternion/scale representation), Opacity $\alpha_i \in [0, 1]$, Spherical Harmonics coefficients $\mathbf{c}_{i, l}$ for view-dependent color.
- Feature Vector $\mathbf{f}_i$: A high-dimensional feature embedding for each Gaussian, extracted from a pre-trained network.

Steps:
1. **Initialization**: Initialize Gaussians using standard 3DGS densification or capture geometry [G].
2. **Feature Extraction**: For each Gaussian $i$, compute a feature vector $\mathbf{f}_i$ using a pre-trained feature extractor (e.g., DINO or similar self-supervised vision transformer) based on the Gaussian's projected image region or associated pixel features [S1][S2, unofficial].
3. **Outlier Detection**: Identify transient distractors by detecting outliers in the feature space $\{\mathbf{f}_i\}$. Gaussians whose feature vectors deviate significantly from the cluster of static scene features are marked as transient [S1][S2, unofficial].
4. **Optimization**: Train the remaining Gaussians (static) using standard differentiable rendering loss (e.g., L1 + D-SSIM) while suppressing or removing the identified transient Gaussians [S1][S2, unofficial].

Parameters:
- Feature extractor: Not specified in sources.
- Outlier threshold: Not specified in sources.

Edge cases:
- Precision: Feature space dimensionality affects outlier detection sensitivity.
- Boundaries: Transient objects that are persistent across many frames may not be detected as outliers.

## Subtopics

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Improves reconstruction quality in casual captures compared to standard Gaussian Avatar Reconstruction [S1][S2, unofficial].
Strengths: Enhanced robustness to transient distractors (e.g., moving people, shadows) in casual video captures [S1][S2, unofficial].
Limitations: Requires a pre-trained feature extractor, adding computational overhead during feature extraction [S1][S2, unofficial].
When to use: When capturing casual videos with transient occluders and high-quality static reconstruction is desired [S1][S2, unofficial].
When not: When real-time performance is critical and feature extraction overhead is prohibitive, or when the scene is static and clean [S1][S2, unofficial].

## Background
Origin: SpotlessSplats was introduced as an extension to Gaussian Avatar Reconstruction to address robustness issues in casual captures [S1][S2, unofficial]. It builds on the idea of using feature spaces for outlier detection, following SWAG's approach of handling transient occluders [S1][S2, unofficial].
History: The method is cited in recent surveys on 3D Gaussian Splatting applications (2025) [S1][S2, unofficial].
Usage: Used for novel view synthesis and avatar reconstruction from casual video captures where transient distractors are common [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
