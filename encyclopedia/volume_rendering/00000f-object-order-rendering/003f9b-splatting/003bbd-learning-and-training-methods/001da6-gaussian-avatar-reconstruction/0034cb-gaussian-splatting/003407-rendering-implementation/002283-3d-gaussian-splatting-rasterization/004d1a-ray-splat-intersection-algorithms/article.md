# Ray-Splat Intersection Algorithms

Also called: Ray-Gaussian Intersection, Ray-Traced Gaussian Rendering, Plücker-Based Gaussian Intersection

## Overview
Ray-splat intersection algorithms compute the contribution of 3D Gaussians to a pixel by analytically evaluating the Gaussian density along the viewing ray, rather than projecting the 3D Gaussian onto the 2D image plane. This approach solves the problem of numerical instability and geometric inaccuracy that arises when splats become degenerate or when perspective-correct depth estimation is required [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md][5992-gaussian-opacity-fields-gof.md]. Unlike standard rasterization which approximates the 3D Gaussian as a 2D ellipse, these methods treat the Gaussian as a volumetric primitive, enabling accurate depth extraction and surface reconstruction [5992-gaussian-opacity-fields-gof.md].

## Shared principles
All ray-splat intersection methods share the fundamental mechanism of evaluating the Gaussian function $G(\mathbf{x}) = \exp(-\frac{1}{2}(\mathbf{x}-\mathbf{\mu})^\top \Sigma^{-1} (\mathbf{x}-\mathbf{\mu}))$ along a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$. The core mathematical challenge is computing the squared Mahalanobis distance $\rho^2$ between the ray and the Gaussian center without explicitly inverting the covariance matrix or handling degenerate cases where the determinant approaches zero.

The shared principles include:
1.  **Ray-Primitive Intersection**: Determining the intersection depth $t$ and the normal $\mathbf{n}$ at the point of closest approach or maximum density [5992-gaussian-opacity-fields-gof.md].
2.  **Analytical Evaluation**: Computing the density value $E(G_k, \mathbf{o}, \mathbf{r})$ analytically along the ray, which serves as the opacity contribution for volume rendering [5992-gaussian-opacity-fields-gof.md].
3.  **Culling and Bounding**: Using tight bounding volumes (AABBs, Plücker-based bounds, or QuadBoxes) to cull Gaussians that do not intersect the viewing frustum or tile [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md][9231-quadbox.md].
4.  **Volume Rendering Accumulation**: Integrating the density contributions along the ray using front-to-back compositing to produce the final pixel color and depth [5992-gaussian-opacity-fields-gof.md].

## Subtopics

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
Uses dual Plücker coordinates to compute the squared distance $\rho^2$ from the viewing ray to the Gaussian center in normalized splat space. It sets itself apart by providing a numerically stable, matrix-inversion-free approach that handles general non-degenerate 3D Gaussians, solving the instability of standard rasterization when axes vanish. Choose this for stable rendering and optimization of degenerate splats.

### [Gaussian Opacity Fields (GOF)](5992-gaussian-opacity-fields-gof.md)
Replaces 2D projection rasterization with ray-tracing-based volume rendering. It analytically evaluates Gaussian contributions along viewing rays to achieve accurate depth estimation and derive level-set surfaces. Choose this when geometric consistency, surface extraction, or accurate depth maps are required.

### [QuadBox](9231-quadbox.md)
Uses four axis-aligned bounding boxes (AABBs) to tightly encapsulate projected 2D Gaussians. It sets itself apart by replacing the single conservative AABB with four adaptive sub-boxes that symmetrically fit the elliptical footprint, reducing tile over-approximation. Choose this for efficient rasterization preprocessing with reduced redundant computations.

## Background
The concept of ray-splat intersection originates from the need to improve the geometric fidelity of 3D Gaussian Splatting (3DGS). Standard 3DGS rasterization projects 3D Gaussians to 2D, which can lead to artifacts and inaccurate depth when splats are large or degenerate [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md]. The use of Plücker coordinates for ray-splat intersection was adapted from earlier work on perspective-correct rendering of ellipsoids [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md]. Gaussian Opacity Fields (GOF) extended this by integrating the ray-Gaussian intersection into a full volume rendering pipeline for surface extraction [5992-gaussian-opacity-fields-gof.md]. QuadBox optimized the rasterization stage by improving the tightness of bounding volumes [9231-quadbox.md]. These methods are used in high-fidelity 3D reconstruction, novel view synthesis, and geometric modeling from point clouds or images.

## Sources
[S1] 3D Gaussian Splatting Rasterization Notes, 2024
