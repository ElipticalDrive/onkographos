# Regularization and Geometry Constraints

Also called: Geometric Regularization, Depth Priors, Frequency Constraints

## Overview
Regularization and Geometry Constraints are training-time techniques that modify the optimization objective or Gaussian parameters to enforce geometric plausibility, prevent over-reconstruction, and stabilize convergence in 3D Gaussian Splatting. They solve the problem where standard photometric losses (e.g., L1, SSIM) fail to distinguish between well-reconstructed surfaces and redundant "floaters," leading to blurred details, geometric hollowness, or unstable camera poses. This family sets itself apart from [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) by explicitly introducing auxiliary constraints beyond pixel-color matching, and from [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by focusing on structural integrity rather than physical light transport properties.

## Shared principles
All methods in this family compute a differentiable auxiliary loss term $\mathcal{L}_{reg}$ that is added to the total training loss:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{reg} \mathcal{L}_{reg} $$
where $\lambda_{reg}$ is a scalar weight. The core mechanism involves extracting a geometric or spectral property from the rendered output (or the input data) and comparing it against a target or constraint. The gradients of $\mathcal{L}_{reg}$ flow back through the differentiable rasterizer to update the Gaussian parameters $\theta$ (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$, and colors $c$) [Parent].

## Subtopics

### [Frequency Regularization](6130-frequency-regularization.md)
Minimizes frequency spectrum discrepancy to alleviate over-reconstruction.

### [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md)
Blurs 2D supervision signals to stabilize camera alignment and restrict high-frequency noise.

### [Depth Regularization](7616-depth-regularization.md)
Aligns rendered depth with monocular estimates to fix geometric hollowness.

### [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)
Transforms covariance to half-ellipsoids for sharp boundaries.

### [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)
Decouples center and opacity optimization via distinct depth renders.

### [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md)
Uses multi-scale reprojection errors for self-supervised geometry.

### Wild-GS
- [S1] A method that aligns pixel appearance features from image triplanes directly to 3D Gaussians and uses depth regularization and visibility maps to mitigate transient effects and constrain geometry.

### Source Feature Extrapolation
- [S2] A technique to improve extrapolation capabilities beyond source view frustums by padding source images and adding positional encoding channels for padded regions, preventing smearing artifacts in regions not visible by source images.
- [S2] Apply border padding to source images $I^{(i)}$.
- [S2] Create a channel $\Gamma$ containing pixel-wise positional encodings for padded regions: $\Gamma[u,v] = \gamma(u,v)$ if $(u,v)$ is not in $I^{(i)}$, else 0. Concatenate $I_{pad}$ and $\Gamma$ to form $I'(i) = \text{concatenate}(I_{pad}, \Gamma)$.
- [S2] $\gamma(\cdot)$ denotes positional encoding with exponential frequencies.

### SDF-Regularized 3DGS
- [S3] Methods to improve geometric fidelity in 3DGS by imposing physical constraints, such as Signed Distance Fields (SDFs), monocular depth priors, or multi-view geometric constraints, during optimization.
- [S3] Introduce a regularization term to minimize the difference between an ideal SDF and the SDF calculated from Gaussians.
- [S3] Use a dual-branch framework with GS- and SDF-branches, jointly optimized to provide mutual guidance.
- [S3] Incorporate geometric regularization such as monocular depth priors or multi-plane Gaussian optimization.

### FreGS
- [S4, unofficial] Applies progressive frequency regularization to primitives to prevent overfitting and high-frequency artifacts during training.

## Background
The problem of "floaters" and unstructured density clouds in 3D Gaussian Splatting was identified early, as optimization for photometric loss alone often fails to represent valid surfaces [S3]. Regularization techniques evolved to address specific failure modes: frequency-based methods to handle over-reconstruction [S2], depth-based methods to fix sparse-view hollowness [Parent], and structural modifications like half-Gaussians for sharp occlusions [S1]. The field also includes self-supervised approaches like cross-scale adaptation to avoid reliance on external depth priors [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
