# Regularization and Geometry Constraints


## Overview
Regularization and Geometry Constraints in 3D Gaussian Splatting (3DGS) address the inherent ambiguity of single-view or sparse-view reconstruction, where color-only supervision often leads to incorrect Gaussian positioning, floating artifacts, or collapsed geometry [S1][S2][S3]. This family of methods imposes auxiliary constraints—such as spectral consistency, depth plausibility, or structural covariance limits—to guide the optimization toward geometrically valid and visually sharp reconstructions [S1][S2][S3]. It sets itself apart from sibling methods like [Physical Parameter Optimization](4111-physical-parameter-optimization.md) or [Base 3DGS Optimization](7549-base-3dgs-optimization.md) by explicitly modifying the loss landscape or primitive parameters to enforce geometric priors, rather than relying solely on photometric reconstruction errors [S1][S2][S3].

## Shared principles
All methods in this family compute a differentiable auxiliary loss term $\mathcal{L}_{reg}$ that is added to the total training loss:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{reg} \mathcal{L}_{reg} $$
where $\lambda_{reg}$ is a scalar weight. The core mechanism involves extracting a geometric or spectral property from the rendered output (or the input data) and comparing it against a target or constraint. The gradients of $\mathcal{L}_{reg}$ flow back through the differentiable rasterizer to update the Gaussian parameters $\theta$ (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$, and colors $c$) [Parent].

## Subtopics

### [Frequency Regularization](6130-frequency-regularization.md)
Frequency regularization is a training-time technique that constrains the spectral content of the rendered output to guide Gaussian densification toward high-frequency regions. It solves the problem where standard 3DGS initialization and densification may fail to capture fine geometric details or high-frequency textures, leading to blurry reconstructions. Unlike [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md) which operates in the spatial domain via kernel smoothing, or [Depth Regularization](7616-depth-regularization.md) which enforces geometric consistency, frequency regularization operates in the spectral domain to explicitly encourage the representation of high-frequency signal components. See [Frequency Regularization](6130-frequency-regularization.md).

### [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md)
Smoothed 2D Supervision is a regularization technique that blurs 2D training images with a scheduled set of 2D Gaussian kernels to produce smoothed image gradients, stabilize camera alignment, and restrict the spectrum of the learned 3D scene. It sets itself apart from fixed-size smoothing by progressively reducing the blur intensity, allowing the model to transition from low-frequency stability to high-frequency detail. It differs from [Frequency Regularization](6130-frequency-regularization.md) by operating directly on the pixel supervision target rather than adjusting positional encoding frequencies, and from [Depth Regularization](7616-depth-regularization.md) by focusing on 2D image-space consistency rather than geometric depth constraints. See [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md).

### [Depth Regularization](7616-depth-regularization.md)
Depth regularization methods enforce geometric plausibility in differentiable volume rendering by minimizing the discrepancy between rendered depth maps and external depth targets (monocular estimates, stereo matches, or LiDAR). This family addresses the inherent ambiguity of single-view or sparse-view reconstruction, where color-only supervision often leads to incorrect Gaussian positioning, floating artifacts, or collapsed geometry. It sets itself apart from sibling methods like [Frequency Regularization](6130-frequency-regularization.md) or [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md) by operating directly on the geometric depth channel rather than color space or frequency domains, providing explicit metric or relative depth constraints. See [Depth Regularization](7616-depth-regularization.md).

### [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)
3D Half-Gaussian Splatting (3D-HGS) modifies the standard 3D Gaussian kernel into a half-ellipsoid to better model sharp surface boundaries and occlusions, addressing the "blooming" artifacts common in full Gaussians near object edges. It achieves this by transforming the covariance matrix into a local basis aligned with a splitting plane normal, projecting the covariance to enforce zero variance along the normal direction, and then transforming it back to world space. This method builds on [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md) by applying a structural constraint to the Gaussian parameters themselves rather than an auxiliary loss term. See [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md).

### [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)
Hard and Soft Depth Regularization is a technique for constraining the geometry of 3D Gaussian Splatting by encouraging spatial reshaping (movement of centers) rather than deforming Gaussian shapes, thereby preventing detail loss and blurred appearance caused by forcing anisotropic Gaussians to fit smooth monocular depth. It sets itself apart from standard depth regularization methods by decoupling the optimization of center positions and opacities through two distinct rendered depth maps, while freezing shape parameters (scale and rotation) during the process. This approach addresses the conflict where NeRF-style depth regularization forces primitive geometry to conform to smooth estimates, which conflicts with complex color appearance. See [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md).

### [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md)
Cross-scale geometric adaptation is a self-adaptive regularization technique that uses reprojection errors across multiple voxel scales to guide geometry learning without requiring explicit depth ground truth. It selects render depths with minimal reprojection error across different scales as pseudo labels, ensuring geometric integrity and reducing floaters more effectively than single-scale depth priors. Unlike siblings that rely on fixed schedules or external priors, this method dynamically adapts to scene frequencies by leveraging low-resolution voxels for coarse geometry early in training and high-resolution voxels for detail refinement later. See [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md).

### Wild-GS
- [S1] A method that aligns pixel appearance features from image triplanes directly to 3D Gaussians and uses depth regularization and visibility maps to mitigate transient effects and constrain geometry.

### Source Feature Extrapolation
- [S2] A technique to improve extrapolation capabilities beyond source view frustums by padding source images and adding positional encoding channels for padded regions, preventing smearing artifacts in regions not visible by source images.
- [S2] Apply border padding to source images $I^{(i)}$.
- [S2] Create a channel $\Gamma$ containing pixel-wise positional encodings for padded regions: $\Gamma[u,v] = \gamma(u,v)$ if $(u,v)$ is not in $I^{(i)}$, else $0$. Concatenate $I_{pad}$ and $\Gamma$ to form $I'(i) = \text{concatenate}(I_{pad}, \Gamma)$.
- [S2] $\gamma(\cdot)$ denotes positional encoding with exponential frequencies.

### SDF-Regularized 3DGS
- [S3] Methods to improve geometric fidelity in 3DGS by imposing physical constraints, such as Signed Distance Fields (SDFs), monocular depth priors, or multi-view geometric constraints, during optimization.
- [S3] Introduce a regularization term to minimize the difference between an ideal SDF and the SDF calculated from Gaussians.
- [S3] Use a dual-branch framework with GS- and SDF-branches, jointly optimized to provide mutual guidance.
- [S3] Incorporate geometric regularization such as monocular depth priors or multi-plane Gaussian optimization.

### FreGS
- [S4, unofficial] Applies progressive frequency regularization to primitives to prevent overfitting and high-frequency artifacts during training.

## Background
The original 3D Gaussian Splatting method optimizes for photometric loss, which often results in unstructured density clouds that do not represent a valid surface [S3]. To recover high-quality geometry, recent methods impose physical constraints during optimization [S3]. This includes integrating Signed Distance Fields (SDFs) into the 3DGS framework [S3], using monocular depth priors [S3], or applying frequency regularization to ensure consistent appearance across resolutions [S4, unofficial]. Popping artifacts arising from depth-sorting steps have also been addressed by replacing global sorting with per-pixel depth comparisons or integrating ray-tracing kernels [S3]. Additionally, gradient oscillations in scaling and rotation parameters have been stabilized by rescaling gradients with the inverse of the current scale [S4, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
