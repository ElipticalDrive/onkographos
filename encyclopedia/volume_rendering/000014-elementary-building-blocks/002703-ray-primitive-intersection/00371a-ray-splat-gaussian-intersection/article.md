# Ray-Splat/Gaussian Intersection
## Overview
This family defines the geometric intersection between a camera ray and 3D Gaussian splats, which are ellipsoidal primitives used in novel view synthesis [G]. It solves the problem of accurately determining which splats contribute to a pixel and at what depth, enabling alpha blending and depth-based operations [G]. It distinguishes itself from sibling methods by handling the specific quadratic nature of Gaussian density fields, supporting both exact ellipsoid intersection and numerical stabilization for degenerate cases, unlike [Sphere Intersection](1620-sphere-intersection.md) which assumes constant radius or [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) which handles planar surfaces [G].

## Shared principles
The shared principles are inherited from [Ray-primitive intersection](769-ray-primitive-intersection.md).

## Subtopics

### [Exact Rational Ray-Surfel Evaluation](2915-exact-rational-ray-surfel-evaluation.md)
Exact Rational Ray-Surfel Evaluation computes the precise intersection point of a viewing ray with a flat surfel rectangle in the surfel's local $(u, v)$ coordinates. It solves the problem of determining where a pixel's ray hits the surfel plane to evaluate the 2D Gaussian opacity, avoiding the per-pixel cross products and divisions required by the original 2DGS formulation. It sets itself apart from [Ray-Splat/Gaussian Intersection](7916-ray-splat-gaussian-intersection.md) by using a homography approximation around the screen-space bounding box center to replace the exact ray-plane intersection with a rational function, significantly reducing computational cost while maintaining visual fidelity.

### [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md)
SPaGS is a real-time rendering method for 3D Gaussian Splatting (3DGS) that supports arbitrary spherical camera models, specifically equirectangular panoramas, by replacing the standard local affine approximation with a numerically stable ray-splat intersection approach. It solves the problem of rendering artifacts and numerical instability inherent in standard 3DGS when applied to wide-field-of-view or distorted panoramic views, where first-order affine projections fail to accurately represent the Gaussian distribution. Unlike ODGS, which uses suboptimal tangent plane approximations, or 3DGUT, which relies on the Unscented Transform, SPaGS achieves higher quality and speed by representing each viewing ray as the intersection of two perpendicular planes in camera space, thereby avoiding matrix inversion of the covariance.

### [Ray-Splat Intersection](6816-ray-splat-intersection.md)
Ray-Splat Intersection determines which 3D Gaussian splats in a scene are intersected by a camera ray, returning the closest hit for depth testing, picking, or alpha blending. It distinguishes itself from sibling methods by supporting both approximate spherical intersection (for performance) and true ellipsoid intersection (for geometric accuracy) within a hierarchical Bounding Volume Hierarchy (BVH) structure. Unlike [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md), which focuses on the parallel dispatch mechanism, this method defines the geometric traversal and intersection logic itself.

### [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md)
Splat Raycasting Dispatch manages the buffer preparation and dispatch call for ray-splat intersection tests, returning hit distances and splat indices. It operates on pre-transformed, packed buffer structures optimized for batched processing, distinguishing it from [Ray-Splat Intersection](6816-ray-splat-intersection.md), and targets standard Euclidean camera models rather than spherical panoramas like [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md).

### [Ray-splat Intersection with Numerical Stabilization](9972-ray-splat-intersection-with-numerical-stabilization.md)
This method computes the intersection depths between camera rays and quadric splats, providing a numerically stable solution when the quadratic coefficient is near zero. It differs from standard quadratic solvers by detecting degenerate cases where the quadric surface is nearly flat or the ray is nearly perpendicular to the principal axes, switching to a linear approximation to prevent floating-point overflow. It builds on [Ray-Splat/gaussian Intersection](7916-ray-splat-gaussian-intersection.md) by handling the specific numerical instability inherent in second-order primitives.

### [Quadratic Gaussian Splatting](9978-quadratic-gaussian-splatting.md)
Quadratic Gaussian Splatting (QGS) is a surface reconstruction method that replaces static Gaussian primitives with deformable quadric surfaces (specifically paraboloids) to capture intricate geometry. Unlike standard Gaussian Splatting which uses Euclidean distance for density modeling, QGS employs geodesic distance-based density distributions to maintain spatial coherence during deformation. It sets itself apart from its siblings by generalizing 3D Gaussian primitives to higher-order surfaces, allowing adaptive curvature modeling while retaining the efficiency of ray-splat intersection.

### [Ray-Point Distance Test](11003-ray-point-distance-test.md)
The Ray-Point Distance Test is a fragment shader technique that tests whether a viewing ray passes within a specified radius of up to 10 pre-selected 3D points, outputting a binary highlight color to indicate selection proximity. It differs from general ray-primitive intersection methods by operating on a discrete, small constant number of points rather than continuous geometric primitives, and by explicitly using the ray segment endpoints to compute perpendicular distance.

### Ray Marching with RBF Intersection
- [S1] Intersect the RBFs of the density representation with slices of thickness $\Delta x$ that are perpendicular to the view direction.
- [S1] Render the slices from far to near using alpha blending with GL_ONE and GL_SRC_ALPHA.
- [S1] Render the bounding quad of all intersections with the RBFs in each slice.
- [S1] For each pixel, retrieve $D(x)$ and $L_x$ from 3D textures.
- [S1] Set the RGB channels of the output to $D(x)L_x$.
- [S1] Set the alpha channel to the differential transmittance of the slice, computed as $\exp(-\frac{\tau}{t} D(x) \Delta x)$.

### Intersection Preserving
- [S2] A technique to discard Gaussians that do not directly intersect with the ray, based on the intersection points identified during depth reinitialization.
- [S2] Identify $\mathcal{I}_{max}$, the collection of all indexes $i_{max}(x)$ for all pixels $x$.
- [S2] Retain only Gaussians whose index $i$ is in $\mathcal{I}_{max}$.
- [S2] $\mathcal{G}^{int} = \{\mathcal{G}_i | i \in \mathcal{I}_{max} \wedge i \in [1, N]\}$, where $\mathcal{G}^{int}$ is the set of intersected Gaussians and $\mathcal{I}_{max}$ is the collection of all indexes $i_{max}(x)$ in the rendered indexes.

## Background
The concept of ray-splat intersection evolved from the need to render 3D Gaussian Splatting (3DGS) efficiently in real-time [G]. Standard 3DGS uses a rasterization-based approach that approximates the projection of Gaussians onto the image plane [G]. For wide-field-of-view cameras or spherical panoramas, this first-order affine approximation introduces significant artifacts, leading to [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md) which uses exact ray-splat intersection with perpendicular planes to avoid covariance matrix inversion [G]. Concurrently, [Ray-Splat Intersection](6816-ray-splat-intersection.md) introduced BVH-based traversal for more complex scenes, supporting both spherical approximations and true ellipsoid intersections [G]. Numerical stability issues in quadratic solvers for quadric splats led to [Ray-splat Intersection with Numerical Stabilization](9972-ray-splat-intersection-with-numerical-stabilization.md), which switches to linear approximations when the quadratic coefficient is negligible [G]. [Quadratic Gaussian Splatting](9978-quadratic-gaussian-splatting.md) further generalized this by using paraboloids and geodesic distances for surface reconstruction [G]. Additionally, [Intersection Preserving](7916-ray-splat-gaussian-intersection.md) discards Gaussians that do not directly intersect with the ray based on depth reinitialization [S2], while [Ray Marching with RBF Intersection](7916-ray-splat-gaussian-intersection.md) intersects RBFs with slices perpendicular to the view direction for volume rendering [S1].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
