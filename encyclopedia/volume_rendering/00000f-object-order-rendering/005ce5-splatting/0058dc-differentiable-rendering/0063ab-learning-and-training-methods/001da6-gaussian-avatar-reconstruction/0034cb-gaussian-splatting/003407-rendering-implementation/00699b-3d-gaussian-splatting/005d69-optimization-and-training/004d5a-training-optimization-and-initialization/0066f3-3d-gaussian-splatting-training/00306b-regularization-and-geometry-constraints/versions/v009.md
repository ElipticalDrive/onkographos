# Regularization and Geometry Constraints

Also called: Geometric Priors, Training Regularization

## Overview
Regularization and Geometry Constraints are techniques applied during the training of 3D Gaussian Splatting to improve geometric fidelity, suppress artifacts (such as floaters and over-reconstruction), and stabilize optimization. While standard 3DGS optimization [3761-3d-gaussian-splatting-training.md] minimizes photometric reconstruction error, these methods introduce auxiliary losses or structural constraints that guide the Gaussian primitives toward physically plausible shapes and positions. They distinguish themselves from sibling methods like [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by focusing on the structural integrity of the representation rather than the physical accuracy of material properties, and from [Base 3DGS Optimization](7549-base-3dgs-optimization.md) by explicitly modifying the loss landscape or parameter space to enforce geometric priors.

## Shared principles
All methods in this family compute a differentiable auxiliary loss term $\mathcal{L}_{reg}$ that is added to the total training loss:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{reg} \mathcal{L}_{reg} $$
where $\lambda_{reg}$ is a scalar weight. The core mechanism involves extracting a geometric or spectral property from the rendered output (or the input data) and comparing it against a target or constraint. The gradients of $\mathcal{L}_{reg}$ flow back through the differentiable rasterizer to update the Gaussian parameters $\theta$ (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$, and colors $c$) [Parent].

## Subtopics

### [Frequency Regularization](6130-frequency-regularization.md)
Frequency Regularization (FR) minimizes the discrepancy between the frequency spectrum of rendered images and ground truth to alleviate over-reconstruction and improve Gaussian densification [6130-frequency-regularization.md]. It solves the problem where standard pixel-wise losses (e.g., L1) fail to differentiate between well-reconstructed and over-reconstructed regions, leading to redundant Gaussian creation and blurred details [6130-frequency-regularization.md]. Unlike naive pixel-wise losses, FR increases pixel gradients in over-reconstructed regions by explicitly disentangling frequency components, enabling adaptive densification [6130-frequency-regularization.md].

### [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md)
Smoothed 2D Supervision enhances the robustness of joint optimization in radiance fields by smoothing the 2D supervision signal [7213-smoothed-2d-supervision.md]. It addresses the instability caused by high-frequency noise in raw image gradients, which can lead to poor camera alignment and overfitting to noise [7213-smoothed-2d-supervision.md]. Unlike standard supervision that uses raw pixels, this method applies a scheduled 2D Gaussian blur to training images, stabilizing gradients and restricting the spectrum of the learned 3D scene [7213-smoothed-2d-supervision.md]. It sets itself apart from [Frequency Regularization](6130-frequency-regularization.md) by operating directly on the 2D image domain rather than modifying the positional encoding or loss weights in the frequency domain.

### [Depth Regularization](7616-depth-regularization.md)
Depth Regularization applies a differentiable auxiliary loss term $\mathcal{L}_{reg}$ to constrain the geometric structure of 3D Gaussian primitives during training, typically by aligning rendered depth or disparity maps with monocular depth estimates or smoothness priors [Parent]. It distinguishes itself from sibling methods like [Frequency Regularization](6130-frequency-regularization.md) and [Smoothed 2D Supervision](7213-smoothed-2d-supervision.md) by explicitly targeting the metric depth accuracy and surface completeness, addressing the "hollowness" and geometric degradation common in sparse-view scenarios [7616-depth-regularization.md].

### [3D Half-Gaussian Splatting Covariance Transformation and Regularization](9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md)
3D Half-Gaussian Splatting (3D-HGS) modifies the standard 3D Gaussian kernel into a half-ellipsoid to better model sharp surface boundaries and occlusions, effectively representing the sharp transitions between object surfaces and empty space [9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md]. It achieves this by transforming the covariance matrix into a local basis aligned with a splitting plane normal, projecting the covariance to enforce near-zero variance along the normal direction, and then transforming it back to world space [9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md]. This method builds on [Regularization and Geometry Constraints](6269-regularization-and-geometry-constraints.md) by applying a structural constraint to the Gaussian parameters themselves rather than an auxiliary loss term [G].

### [Hard and Soft Depth Regularization](9178-hard-and-soft-depth-regularization.md)
Hard and Soft Depth Regularization is a technique for 3D Gaussian Splatting that constrains geometry by encouraging the spatial reshaping (movement of centers) of primitives rather than deforming their shapes, thereby preventing the detail loss and blurred appearance caused by forcing anisotropic Gaussians to fit smooth monocular depth [9178-hard-and-soft-depth-regularization.md]. It sets itself apart from standard depth regularization methods by decoupling the optimization of center positions and opacities through two distinct rendered depth maps, allowing coarse depth fitting and fine color appearance to be balanced independently [9178-hard-and-soft-depth-regularization.md].

### [Cross-scale geometric adaptation](10796-cross-scale-geometric-adaptation.md)
Cross-scale geometric adaptation is a self-adaptive regularization technique that uses reprojection errors across multiple voxel scales to guide geometry learning without requiring explicit depth ground truth [10796-cross-scale-geometric-adaptation.md]. It selects render depths with minimal reprojection error across different scales as pseudo labels, ensuring geometric integrity and reducing floaters more effectively than single-scale depth priors [10796-cross-scale-geometric-adaptation.md]. Unlike siblings that rely on fixed schedules or external priors, this method dynamically adapts to scene frequencies by leveraging low-resolution voxels for coarse geometry early in training and high-resolution voxels for detail refinement later [10796-cross-scale-geometric-adaptation.md].

### [Geometry Regularization](14222-geometry-regularization.md)
Geometry regularization improves the geometric fidelity and smoothness of 3D Gaussian Splatting scenes by constraining the shape and orientation of the Gaussian kernels, distinct from depth supervision which only constrains their spatial location [14222-geometry-regularization.md]. It sets itself apart from sibling methods like [Depth Regularization](7616-depth-regularization.md) by enforcing structural constraints (flattening and normal alignment) rather than positional errors, and from [Frequency Regularization](6130-frequency-regularization.md) by operating in the spatial domain to reduce floaters and improve surface continuity [14222-geometry-regularization.md].

### Wild-GS
- [S1] A method that aligns pixel appearance features from image triplanes directly to 3D Gaussians and uses depth regularization and visibility maps to mitigate transient effects and constrain geometry.

### Source Feature Extrapolation
- [S2] A technique to improve extrapolation capabilities beyond source view frustums by padding source images and adding positional encoding channels for padded regions, preventing smearing artifacts in regions not visible by source images.
- [S2] Apply border padding to source images I(i).
- [S2] Create a channel Γ containing pixel-wise positional encodings for padded regions: Γ[u,v] = γ(u,v) if (u,v) is not in I(i), else 0. Concatenate I_pad and Γ to form I'(i) = concatenate(I_pad, Γ).
- [S2] γ(·) denotes positional encoding with exponential frequencies.

### SDF-Regularized 3DGS
- [S3] Methods to improve geometric fidelity in 3DGS by imposing physical constraints, such as Signed Distance Fields (SDFs), monocular depth priors, or multi-view geometric constraints, during optimization.
- [S3] Introduce a regularization term to minimize the difference between an ideal SDF and the SDF calculated from Gaussians.
- [S3] Use a dual-branch framework with GS- and SDF-branches, jointly optimized to provide mutual guidance.
- [S3] Incorporate geometric regularization such as monocular depth priors or multi-plane Gaussian optimization.

### FreGS
- [S4, unofficial] Applies progressive frequency regularization to primitives to prevent overfitting and high-frequency artifacts during training.

## Background
The concept of regularization in 3D Gaussian Splatting emerged as researchers identified limitations in the original formulation, which optimized solely for photometric reconstruction. This often led to "hollow" geometries, floaters, and over-reconstruction in sparse-view scenarios. Early works like FreGS [S4, unofficial] introduced frequency-based losses to address overfitting, while others like [7616-depth-regularization.md] and [14222-geometry-regularization.md] incorporated geometric priors such as monocular depth estimates and surface normal constraints to enforce physical plausibility. The field has since expanded to include cross-scale adaptation [10796-cross-scale-geometric-adaptation.md] and structural modifications like half-Gaussians [9150-3d-half-gaussian-splatting-covariance-transformation-and-reg.md] to better handle sharp boundaries.

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] DINER: Depth-aware Image-based NEural Radiance fields, https://doi.org/10.1109/cvpr52729.2023.01198, fetched 2026-10-29
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
