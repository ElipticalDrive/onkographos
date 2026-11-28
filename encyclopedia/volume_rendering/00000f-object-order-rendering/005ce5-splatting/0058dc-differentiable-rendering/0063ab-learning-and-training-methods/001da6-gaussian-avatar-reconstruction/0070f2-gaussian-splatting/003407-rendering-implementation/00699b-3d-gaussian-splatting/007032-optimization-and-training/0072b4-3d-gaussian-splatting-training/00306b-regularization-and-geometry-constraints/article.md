# Regularization and Geometry Constraints

Also called: Geometric Regularization, Depth Constraints, Frequency Constraints

## Overview
Regularization and Geometry Constraints are training-time techniques for 3D Gaussian Splatting that prevent over-reconstruction, reduce floating artifacts, and enforce geometric fidelity by adding constraints beyond standard photometric loss. They set themselves apart from sibling methods like [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by directly modifying the optimization landscape of Gaussian parameters (position, shape, opacity) or rendered outputs (frequency, depth) rather than altering the physical properties of the primitives themselves.

## Shared principles
The shared mechanism across all methods in this family is the addition of a regularization term $\mathcal{L}_{reg}$ to the total training loss $\mathcal{L}_{total}$, typically combined with the photometric loss $\mathcal{L}_{photo}$ (e.g., L1 + SSIM):
$$ \mathcal{L}_{total} = \mathcal{L}_{photo} + \lambda \mathcal{L}_{reg} $$
where $\lambda$ is a hyperparameter weighting the regularization strength [Parent]. The regularization term $\mathcal{L}_{reg}$ is derived from either the frequency spectrum of the rendered image, the geometric consistency of depth maps, or the structural properties of the Gaussian covariance matrices. These constraints are applied during the gradient descent step to update Gaussian parameters, often requiring specific handling of gradients (e.g., freezing shape parameters during depth updates) to ensure stable convergence [S1].

## Subtopics

### [Frequency Regularization](6130-frequency-regularization.md)
Frequency Regularization (FR) minimizes the discrepancy between the frequency spectrum of rendered and ground truth images to alleviate over-reconstruction. It sets itself apart from [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md) by operating in the Fourier domain rather than the spatial or depth domain, and from [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md) by not enforcing specific covariance structures. It is chosen when the model exhibits "over-reconstruction" (redundant Gaussians in well-reconstructed regions) and needs to increase gradients in those areas by disentangling frequency components.

### [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md)
Smoothed 2D Supervision enhances robustness by blurring 2D training images with a scheduled set of 2D Gaussian kernels. It sets itself apart from [Frequency Regularization](6130-frequency-regularization.md) because it operates directly in the spatial domain on the rendered images rather than in the Fourier domain. It is chosen to stabilize camera alignment and restrict the spectrum of the learned 3D scene to prevent high-frequency noise from dominating early training.

### [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)
This method modifies the covariance structure of 3D Gaussian primitives to enforce a half-ellipsoid geometry, aligning the Gaussian shape with local surface normals. It sets itself apart from standard 3DGS by replacing full ellipsoids with truncated shapes that eliminate "dust" and noise artifacts. Unlike [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md) which constrains positions, this method directly regularizes the shape parameters. It is chosen when sharp transitions between object surfaces and empty space need to be modeled more accurately.

### [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)
Hard and Soft Depth Regularization prevents detail loss and blurred appearance by decoupling the optimization of Gaussian centers and opacities from their shape parameters. It sets itself apart from [Frequency Regularization](6130-frequency-regularization.md) and [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md) by explicitly freezing scale and rotation to enforce spatial reshaping (movement of centers) rather than shape deformation. It is chosen when standard depth regularization causes conflicts between anisotropic Gaussian shapes and complex color appearance.

### [FreGS](9212-fregs.md)
FreGS prevents overfitting and high-frequency artifacts by penalizing discrepancies between the frequency spectra of rendered and ground-truth images. It sets itself apart from sibling methods that constrain geometry directly (e.g., [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)) or use spatial gradients (e.g., [ABSGS](14222-geometry-regularization.md)) by enforcing spectral consistency. It is chosen to address "over-reconstruction" where the model captures noise or fine details that do not generalize.

### [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md)
Cross-scale geometric adaptation is a self-adaptive regularization technique that uses reprojection errors across multiple voxel scales to guide geometry learning without requiring explicit depth ground truth. It sets itself apart from sibling methods that may rely on complex scheduling or external priors by leveraging the consistency of low-frequency components early in training and high-frequency details later. It is chosen to significantly reduce floaters and ensure geometric integrity.

### [Geometry Regularization](14222-geometry-regularization.md)
Geometry Regularization improves geometric fidelity and smoothness by constraining the shape and orientation of Gaussian primitives, distinct from depth supervision which only constrains their 3D location. It sets itself apart from [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md) by enforcing surface-aligned covariance structures (flattening) and normal alignment rather than just positional depth errors. It is chosen when the reconstruction lacks surface-like properties and appears as an unstructured density cloud.

### Wild-GS
- [S1] A method that aligns pixel appearance features from image triplanes directly to 3D Gaussians and uses depth regularization and visibility maps to mitigate transient effects and constrain geometry.

### Source Feature Extrapolation
- [S2] A technique to improve extrapolation capabilities beyond source view frustums by padding source images and adding positional encoding channels for padded regions, preventing smearing artifacts in regions not visible by source images.
- [S2] Apply border padding to source images I(i).
- [S2] Create a channel $\Gamma$ containing pixel-wise positional encodings for padded regions: $\Gamma[u,v] = \gamma(u,v)$ if $(u,v)$ is not in $I(i)$, else 0. Concatenate $I_{pad}$ and $\Gamma$ to form $I'(i) = \text{concatenate}(I_{pad}, \Gamma)$.
- [S2] $\gamma(\cdot)$ denotes positional encoding with exponential frequencies.

### SDF-Regularized 3DGS
- [S3] Methods to improve geometric fidelity in 3DGS by imposing physical constraints, such as Signed Distance Fields (SDFs), monocular depth priors, or multi-view geometric constraints, during optimization.
- [S3] Introduce a regularization term to minimize the difference between an ideal SDF and the SDF calculated from Gaussians.
- [S3] Use a dual-branch framework with GS- and SDF-branches, jointly optimized to provide mutual guidance.
- [S3] Incorporate geometric regularization such as monocular depth priors or multi-plane Gaussian optimization.

### MVG-Splatting
- [S4] A method that employs a depth map to enforce geometric consistency, specifically applying geometric consistency only in the near and far regions of the scene to effectively mitigate under-reconstruction in these critical regions while avoiding the risk of over-reconstruction.

### Shape Energy Constraint
- [S5] An energy term defined as the combination of opacity energy and boundary energy, used to preserve the shape, opacity, and unitized colors (spherical harmonics) of Gaussian radiance fields during deformation.
- [S5] Relying solely on feature energy is insufficient to preserve opacity and unitized colors; gray Gaussians with SH values close to zero appear and their size becomes uncontrolled because they cannot be constrained by feature energy alone.
- [S5] Without $E_{shape}$, the optimization process becomes unstable and artifacts appear; with $E_{shape}$, the process stabilizes and these artifacts are mitigated.

## Background
The need for regularization in 3D Gaussian Splatting arises from the fact that standard optimization for photometric loss often results in unstructured density clouds that do not represent a valid surface [S3]. Early methods focused on densification strategies, but as the field evolved, constraints were introduced to improve geometric fidelity. SDF-based methods and depth priors were incorporated to provide physical constraints [S3]. Frequency regularization was proposed to address over-reconstruction and improve densification by operating in the Fourier domain [S2]. Smoothed 2D supervision was introduced to stabilize camera alignment and prevent high-frequency noise [S2]. Half-Gaussian transformations were developed to model sharp surface transitions [S1]. These methods collectively address the limitations of pure photometric optimization in producing high-quality, geometrically consistent 3D scenes.

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S5] As-Rigid-As-Possible Deformation of Gaussian Radiance Fields, https://doi.org/10.1109/TVCG.2025.3555404, fetched 2026-10-30
