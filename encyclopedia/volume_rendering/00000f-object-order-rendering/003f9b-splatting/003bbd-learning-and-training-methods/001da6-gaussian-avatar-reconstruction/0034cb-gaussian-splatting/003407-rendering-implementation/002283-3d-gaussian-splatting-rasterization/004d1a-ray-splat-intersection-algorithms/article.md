# Ray-Splat Intersection Algorithms

Also called: Ray-based Gaussian Evaluation, Ray-Splat Traversal

## Overview
Ray-splat intersection algorithms determine the contribution of 3D Gaussian primitives to a pixel by evaluating the Gaussian density along the viewing ray rather than projecting the 3D covariance to 2D screen space. This approach solves the numerical instability and projection errors inherent in standard rasterization, particularly for degenerate splats or extreme perspective. It sets itself apart from sibling methods like [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by avoiding the explicit inversion of the 4x4 projection matrix during per-pixel evaluation, trading slightly higher per-pixel cost for robustness to degeneracy [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md].

## Shared principles
All ray-splat intersection methods share the fundamental mechanism of evaluating the Gaussian density function $G(\mathbf{u}) = \exp(-\frac{1}{2}\rho^2)$ along the viewing ray, where $\rho$ is the distance from the ray to the Gaussian center in a normalized space.

1.  **Ray Representation**: The viewing ray passing through a pixel is represented either as a line in 3D space or as the intersection of two planes. In Plücker coordinate approaches, the ray is defined by its direction $\mathbf{d}$ and moment $\mathbf{m}$ [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md].
2.  **Distance Metric**: The core computation is the squared Mahalanobis distance $\rho^2$ from the ray to the Gaussian center. In normalized splat space (where the Gaussian is a unit sphere), this simplifies to $\rho^2 = \frac{\|\mathbf{m}\|^2}{\|\mathbf{d}\|^2}$ [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md].
3.  **Bounding Volume**: To cull non-intersecting splats, tight bounding volumes are computed. Standard methods use Axis-Aligned Bounding Boxes (AABBs), while advanced methods like QuadBox use adaptive sub-boxes to reduce over-approximation [9231-quadbox.md].
4.  **Opacity Thresholding**: A minimum opacity threshold $\alpha_{\min}$ is used to filter out Gaussians that contribute negligibly, ensuring numerical stability and reducing computational load [9231-quadbox.md].

## Subtopics

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
This method provides a numerically stable, matrix-inversion-free approach for perspectively accurate rendering of non-degenerate 3D Gaussians. It computes the exact distance between a viewing ray and the Gaussian center in normalized splat space using Plücker coordinates. Unlike standard rasterization, it evaluates the Gaussian density along the per-pixel viewing ray, ensuring stability even when splat axes vanish. It sets itself apart from siblings like [Gaussian Opacity Fields (GOF)](5992-gaussian-opacity-fields-gof.md) by avoiding explicit 4x4 matrix inversion, trading per-pixel cost for reduced setup overhead and robustness.

### [Gaussian Opacity Fields (GOF)](5992-gaussian-opacity-fields-gof.md)
Uses ray-tracing-style intersection for accurate depth.

### [QuadBox](9231-quadbox.md)
QuadBox is a Gaussian-tile intersection method that uses four axis-aligned bounding boxes (AABBs) to tightly encapsulate projected 2D Gaussians. It reduces tile over-approximation and redundant computations in the rasterizer preprocess stage. It sets itself apart from its siblings by replacing the single conservative AABB with four adaptive sub-boxes that follow the ellipse's major axis and are geometrically stretched to fit the minor axis, combined with a branch-free single-pass traversal algorithm (QPass).

## Background
The concept of ray-splat intersection originates from the need to render Gaussian primitives with higher fidelity than standard screen-space rasterization allows. Standard 3D Gaussian Splatting [11838-primitive-representation.md] projects 3D covariances to 2D, which can fail for degenerate cases or extreme perspectives. The Plücker coordinate approach [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md] adapts techniques from ray tracing to provide a closed-form solution for the distance metric. QuadBox [9231-quadbox.md] improves upon the preprocessing stage of rasterization by refining the bounding volume hierarchy to reduce overdraw. These methods are used in high-fidelity Gaussian Avatar Reconstruction [Gaussian Avatar Reconstruction] and advanced rendering pipelines [Speedy-Splat Rendering Pipeline].

## Sources
[S1] 3D Gaussian Splatting Rasterization Notes, URL, date fetched
