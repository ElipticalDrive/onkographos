# Regularization and Geometry Constraints

Also called: Geometric Regularization, Depth Regularization, Frequency Regularization

## Overview
Regularization and Geometry Constraints are training-time techniques for 3D Gaussian Splatting that enforce structural integrity, suppress artifacts (such as floaters or over-reconstruction), and improve geometric fidelity by adding constraints beyond standard photometric loss [S1, S2][S4, unofficial]. Unlike [Base 3DGS Optimization](7549-base-3d-gaussian-splatting.md) which relies primarily on L1/LPIPS photometric errors, this family explicitly targets the geometry of the Gaussian primitives or the frequency spectrum of the rendered images to resolve ambiguities in monocular or sparse-view reconstruction [S2][S4, unofficial].

## Shared principles
The core mechanism involves augmenting the total training loss $\mathcal{L}_{total}$ with a regularization term $\mathcal{L}_{reg}$:
$$
\mathcal{L}_{total} = \mathcal{L}_{photometric} + \lambda \mathcal{L}_{reg}
$$
where $\lambda$ is a weighting hyperparameter [S1, S2][S4, unofficial]. The regularization term $\mathcal{L}_{reg}$ is derived from one of the following domains:
1.  **Frequency Domain:** Constraining the spectral content of the rendered image to match the ground truth, thereby preventing high-frequency noise or over-smoothing [S2][S4, unofficial].
2.  **Geometric Shape:** Modifying the covariance matrix or scale parameters of the Gaussians to enforce surface-like properties (e.g., flatness, half-ellipsoids) [S1][S4, unofficial].
3.  **Depth Supervision:** Using monocular depth estimates or multi-view reprojection errors as pseudo-ground truth to constrain Gaussian positions and opacities [S1, S2][S4, unofficial].
4.  **Spatial Smoothing:** Blurring the target supervision signal to stabilize gradient propagation during early training stages [S2][S4, unofficial].

## Subtopics

### [Frequency Regularization](6130-frequency-regularization.md)
Frequency Regularization (FR) minimizes the discrepancy between the frequency spectrum of rendered images and ground truth to alleviate over-reconstruction and improve Gaussian densification. Unlike naive pixel-wise L1 loss, which results in small gradients in over-reconstructed regions and prevents effective densification, FR explicitly disentangles frequency components to increase pixel gradients in these regions, enabling adaptive densification. It builds on [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md) by adding a frequency-domain loss term to the total training loss.

### [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md)
Smoothed 2D Supervision is a regularization technique that enhances the robustness of joint optimization by smoothing the 2D supervision signal through the application of scheduled 2D Gaussian kernels to training images. It solves the problem of unstable camera alignment and noisy gradient signals during the learning of 3D scene representations by restricting the spectrum of the learned scene and producing smoother image gradients. Unlike [Frequency Regularization](6130-frequency-regularization.md) which operates in the frequency domain, or [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md) which modifies geometric covariance, this method operates directly on the 2D pixel supervision space. It is distinct from [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md) as it does not use depth priors but rather blurs the color/intensity targets.

### [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)
This method modifies the covariance structure of 3D Gaussians to enforce half-ellipsoid geometry, allowing them to model sharp surface boundaries and occlusions more effectively than full ellipsoids. It solves the problem of "bloated" or "double-sided" Gaussian artifacts that occur when full ellipsoids intersect object surfaces, by projecting the covariance into a local basis aligned with a surface normal and regularizing the depth component. Unlike [Frequency Regularization](6130-frequency-regularization.md) which operates in the spectral domain, or [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md) which uses external depth priors, this method intrinsically alters the Gaussian shape during training to match surface geometry.

### [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)
Hard and Soft Depth Regularization is a training strategy for 3D Gaussian Splatting that prevents detail loss and blurred appearance by constraining geometry updates to spatial reshaping (movement of centers) rather than deforming Gaussian shapes. It solves the conflict where forcing anisotropic Gaussians to fit smooth monocular depth estimates causes the primitive geometry to conform to the depth rather than the complex color appearance. It sets itself apart from standard depth regularization by freezing scale and rotation parameters and rendering two distinct depth maps to independently update center positions and opacities.

### [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md)
Cross-scale geometric adaptation is a self-adaptive regularization method that uses reprojection errors across multiple voxel scales to guide geometry learning without requiring explicit depth ground truth. It selects render depths with minimal reprojection error across different scales as pseudo labels, ensuring geometric integrity and reducing floaters more robustly than single-scale depth priors.

### [Geometry Regularization](14222-geometry-regularization.md)
Geometry regularization improves the structural integrity and smoothness of reconstructed surfaces in 3D Gaussian Splatting by constraining the geometric shape of the Gaussians, distinct from depth supervision which primarily constrains positional accuracy. It addresses the issue of "floaters" and noisy geometry by enforcing two specific constraints: aligning the orientation of rendered depth normals with per-Gaussian normals, and flattening each Gaussian along its smallest axis to better approximate surface surflets. This method is particularly useful for reducing artifacts in complex geometries where standard photometric loss alone leads to ambiguous depth estimates.

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
The application of regularization in 3D Gaussian Splatting emerged as a response to the limitations of pure photometric optimization, which often leads to unstructured density clouds, floaters, and poor geometric fidelity [S1, S3]. Early works like [Base 3DGS Optimization](7549-base-3d-gaussian-splatting.md) focused on densification strategies, but subsequent research identified that geometric constraints were necessary for high-quality surface reconstruction [S1, S3]. Techniques such as [Frequency Regularization](6130-frequency-regularization.md) and [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md) were introduced to stabilize training and improve gradient flow [S2][S4, unofficial]. Methods like [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md) and [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md) addressed specific geometric artifacts like "bloated" Gaussians and depth ambiguity [S1]. The integration of SDFs and multi-scale reprojection errors further refined the geometric consistency of the representations [S1, S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
