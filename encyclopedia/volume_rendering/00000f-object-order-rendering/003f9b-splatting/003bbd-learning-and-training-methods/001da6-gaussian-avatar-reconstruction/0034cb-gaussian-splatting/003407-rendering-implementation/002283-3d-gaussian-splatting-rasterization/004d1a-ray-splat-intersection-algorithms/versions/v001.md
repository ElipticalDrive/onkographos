# Ray-Splat Intersection Algorithms

Also called: Ray-based Gaussian Evaluation, Plücker-based Splatting, QuadBox Traversal

## Overview
Ray-splat intersection algorithms compute the contribution of 3D Gaussian primitives to a specific pixel by evaluating the Gaussian's value along the viewing ray, rather than projecting the primitive to screen space and rasterizing it as a 2D ellipse. This approach solves the problem of perspective-correct evaluation and visibility determination without relying on conservative bounding box approximations. It sets itself apart from standard 3D Gaussian Splatting rasterization by providing closed-form, numerically stable solutions for ray-primitive intersection, often avoiding matrix inversions or reducing tile over-approximation [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md], [9231-quadbox.md].

## Shared principles
All methods in this family operate on the principle of evaluating the Gaussian density function $G(\mathbf{u}) = \exp(-\frac{1}{2}\mathbf{u}^\top \Sigma^{-1} \mathbf{u})$ along the viewing ray. Instead of rasterizing the projected 2D covariance $\Sigma_{2D}$, these algorithms determine the intersection or distance between the ray and the 3D Gaussian center in a transformed space. The core mathematical invariant is the squared Mahalanobis distance $\rho^2$ from the ray to the Gaussian center, which determines the alpha value. This requires transforming the ray or the primitive into a coordinate system where the distance calculation is algebraic and stable, typically avoiding the explicit inversion of the full 4x4 projection matrix during per-pixel evaluation.

## Subtopics

### [Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates](3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md)
Computes the squared distance from the viewing ray to the Gaussian center using dual Plücker coordinates derived from ray planes. It sets itself apart by providing a closed-form, numerically stable solution that handles degenerate ellipsoids and avoids matrix inversion. Choose this when perspective-correct accuracy and stability are critical.

### [Gaussian Opacity Fields (GOF)](5992-gaussian-opacity-fields-gof.md)
Uses ray-tracing-style intersection for accurate depth. Choose this when precise depth ordering and opacity field representation are required.

### [QuadBox](9231-quadbox.md)
Uses four axis-aligned bounding boxes (AABBs) to tightly encapsulate projected 2D Gaussians. It sets itself apart by replacing the conservative single AABB with a geometry-aware QuadBox, reducing tile over-approximation. Choose this for optimized tile traversal in rasterization pipelines.

## Background
The concept of evaluating splats via ray intersection originates from early volume rendering and point-based rendering techniques, where ray-primitive intersection was preferred for accuracy over rasterization approximations. In the context of 3D Gaussian Splatting, standard rasterization projects Gaussians to 2D ellipses. However, perspective-correct evaluation and efficient visibility culling led to the development of ray-based methods. The Plücker coordinate approach [3403-perspective-correct-3d-gaussian-splatting-via-pl-cker-coordi.md] and QuadBox traversal [9231-quadbox.md] are recent developments aimed at improving the efficiency and accuracy of Gaussian rasterization by leveraging algebraic ray intersection properties.

## Sources
[S1] Perspective-Correct 3D Gaussian Splatting via Plücker Coordinates, https://github.com/nerfstudio-project/gsplat, 2024
