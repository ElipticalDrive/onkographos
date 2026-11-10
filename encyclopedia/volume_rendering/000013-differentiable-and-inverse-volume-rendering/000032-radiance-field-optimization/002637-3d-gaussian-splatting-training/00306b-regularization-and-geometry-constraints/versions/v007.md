# Regularization and Geometry Constraints

Also called: Geometric Priors, Auxiliary Losses, Structural Constraints

## Overview
Regularization and Geometry Constraints are techniques applied during the differentiable training of 3D Gaussian Splatting (3DGS) to enforce geometric consistency, suppress artifacts, and improve surface fidelity beyond what photometric RGB loss alone can achieve [Parent]. They address the "hollowness," "blooming," and high-frequency noise common in sparse-view or unconstrained optimization by introducing auxiliary losses or structural modifications to the Gaussian primitives [S1][S4, unofficial]. Unlike sibling methods that focus on training efficiency or specific camera models, this family explicitly targets the metric accuracy and structural integrity of the reconstructed scene, often by leveraging external priors (depth, frequency) or internal geometric constraints (covariance shape) [S1][S2].

## Shared principles
All methods in this family compute a differentiable auxiliary loss term $\mathcal{L}_{reg}$ or apply a structural constraint that is integrated into the total training objective:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{reg} \mathcal{L}_{reg} $$
where $\lambda_{reg}$ is a scalar weight balancing the regularization against the photometric loss [Parent]. The core mechanism involves extracting a geometric or spectral property from the rendered output (or the input data) and comparing it against a target or constraint. The gradients of $\mathcal{L}_{reg}$ flow back through the differentiable rasterizer to update the Gaussian parameters $\theta$ (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$, and colors $c$) [Parent].

## Subtopics

### [Frequency Regularization](6130-frequency-regularization.md)
Frequency regularization modifies the optimization trajectory by penalizing or encouraging specific spectral components in the rendered output or Gaussian parameters. It addresses overreconstruction and underrepresentation, which lead to blur and visual inconsistencies. Unlike sibling methods that rely on geometric priors like depth, it operates in the spectral domain to control spatial frequency content. It sets itself apart by directly targeting high-frequency details that standard densification may miss, pushing Gaussians toward high-frequency regions to improve sharpness. See [Frequency Regularization](6130-frequency-regularization.md).

### [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md)
Smoothed 2D Supervision enhances the robustness of joint optimization by blurring 2D training images with a scheduled set of 2D Gaussian kernels. It stabilizes camera alignment by producing smoothed image gradients and restricts the spectrum of the learned 3D scene to prevent high-frequency noise. It sets itself apart from siblings like Frequency Regularization by operating directly on the 2D image space supervision signal rather than modifying the 3D representation's spectral properties internally. See [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md).

### [Depth Regularization](7616-depth-regularization.md)
Depth Regularization applies a differentiable auxiliary loss term $\mathcal{L}_{reg}$ to constrain the geometric structure of 3D Gaussian primitives during training, typically by aligning rendered depth or disparity maps with monocular depth estimates or smoothness priors [Parent]. It distinguishes itself from siblings like Frequency Regularization by explicitly targeting metric depth accuracy and surface completeness, addressing the "hollowness" and geometric degradation common in sparse-view scenarios. See [Depth Regularization](7616-depth-regularization.md).

### [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)
3D Half-Gaussian Splatting (3D-HGS) modifies the standard 3D Gaussian kernel into a half-ellipsoid to better model sharp surface boundaries and occlusions, addressing "blooming" artifacts near object edges. It achieves this by transforming the covariance matrix into a local basis aligned with a splitting plane normal, projecting the covariance to enforce zero variance along the normal direction, and transforming it back to world space. This method builds on Regularization and Geometry Constraints by applying a structural constraint to the Gaussian parameters themselves rather than an auxiliary loss term. See [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md).

### [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)
Hard and Soft Depth Regularization constrains geometry by encouraging spatial reshaping (movement of centers) rather than deforming Gaussian shapes, preventing detail loss and blurred appearance caused by forcing anisotropic Gaussians to fit smooth monocular depth. It sets itself apart by decoupling the optimization of center positions and opacities through two distinct rendered depth maps, while freezing shape parameters (scale and rotation) during the process. This approach addresses the conflict where NeRF-style depth regularization forces primitive geometry to conform to smooth estimates, conflicting with complex color appearance. See [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md).

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
The problem of geometric inconsistency in 3D Gaussian Splatting arises because standard optimization relies solely on photometric RGB loss, which often results in unstructured density clouds that do not represent valid surfaces [S3]. Early extensions introduced depth priors and frequency constraints to stabilize training and improve surface quality [S1][S4, unofficial]. The concept of using auxiliary losses for geometry has roots in NeRF regularization techniques, which were adapted for Gaussian primitives to address issues like "hollowness" and "blooming" [S1][S3]. Recent surveys highlight the trend toward integrating physical constraints like SDFs and multi-view geometric consistency to enhance reconstruction fidelity [S3][S4, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
