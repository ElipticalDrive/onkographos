# Ray-Splat/Gaussian Intersection

Also called: Ray-Gaussian Intersection, Ray-Ellipsoid Intersection

## Overview
This family defines the geometric primitives and algorithms for determining the intersection between a camera ray and 3D Gaussian splats, which are ellipsoidal density distributions used in novel view synthesis. It solves the problem of efficiently identifying which splats contribute to a pixel's color and depth, enabling alpha blending and depth testing. It distinguishes itself from sibling methods by handling the specific anisotropic, probabilistic nature of Gaussian splats (defined by covariance matrices or scale/rotation parameters) rather than hard geometric primitives like boxes or triangles.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md)
SPaGS is a real-time rendering method for 3D Gaussian Splatting (3DGS) that supports arbitrary spherical camera models, specifically equirectangular panoramas, by replacing the standard local affine approximation with a numerically stable ray-splat intersection approach. It solves the problem of rendering artifacts and numerical instability inherent in standard 3DGS when applied to wide-field-of-view or distorted panoramic views, where first-order affine projections fail to accurately represent the Gaussian distribution. Unlike ODGS, which uses suboptimal tangent plane approximations, or 3DGUT, which relies on the Unscented Transform, SPaGS achieves higher quality and speed by representing each viewing ray as the intersection of two perpendicular planes in camera space, thereby avoiding matrix inversion of the covariance.

### [Ray-Splat Intersection](6816-ray-splat-intersection.md)
Ray-Splat Intersection determines which 3D Gaussian splats in a scene are intersected by a camera ray, returning the closest hit for depth testing, picking, or alpha blending. It distinguishes itself from sibling methods by supporting both approximate spherical intersection (for performance) and true ellipsoid intersection (for geometric accuracy) within a hierarchical Bounding Volume Hierarchy (BVH) structure. Unlike [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md), which focuses on the parallel dispatch mechanism, this method defines the geometric traversal and intersection logic itself.

### [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md)
Splat Raycasting Dispatch is the computational kernel that performs ray-splat intersection tests for 3D Gaussian splats, returning hit distances and splat indices. It solves the problem of determining which splats intersect a given ray within a specified depth range, serving as the core intersection primitive for raycasting-based Gaussian splatting renderers. It differs from [Ray-Splat Intersection](6816-ray-splat-intersection.md) by operating on pre-transformed, packed buffer structures optimized for batched processing, and from [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md) by targeting standard Euclidean camera models rather than spherical panoramas.

### Ray Marching with RBF Intersection
- Intersect the RBFs of the density representation with slices of thickness $\Delta x$ that are perpendicular to the view direction [S1].
- Render the slices from far to near using alpha blending with GL_ONE and GL_SRC_ALPHA [S1].
- Render the bounding quad of all intersections with the RBFs in each slice [S1].
- For each pixel, retrieve $D(x)$ and $L_x$ from 3D textures [S1].
- Set the RGB channels of the output to $D(x)L_x$ [S1].
- Set the alpha channel to the differential transmittance of the slice, computed as $\exp(-\frac{\tau}{t} D(x) \Delta x)$ [S1].

### Intersection Preserving
- **Definition**: A technique to discard Gaussians that do not directly intersect with the ray, based on the intersection points identified during depth reinitialization [S2].
- **Steps**: Identify $\mathcal{I}_{max}$, the collection of all indexes $i_{max}(x)$ for all pixels $x$. Retain only Gaussians whose index $i$ is in $\mathcal{I}_{max}$ [S2].
- **Formula**: $\mathcal{G}^{int} = \{\mathcal{G}_i | i \in \mathcal{I}_{max} \wedge i \in [1, N]\}$, where $\mathcal{G}^{int}$ is the set of intersected Gaussians and $\mathcal{I}_{max}$ is the collection of all indexes $i_{max}(x)$ in the rendered indexes [S2].

## Background
The intersection of rays with Gaussian primitives is fundamental to 3D Gaussian Splatting (3DGS) [G]. Standard 3DGS uses a local affine approximation for projection, which fails for wide-field-of-view cameras [S1]. SPaGS addresses this by using plane-based intersection to avoid covariance inversion [S1]. BVH-based intersection methods allow for efficient culling of non-intersecting splats, supporting both spherical approximations and true ellipsoid tests [S1]. Intersection preserving techniques prune Gaussians that do not geometrically intersect the ray, optimizing rendering performance [S2]. Ray marching with RBF intersection is an alternative approach for density fields, using slice-based rendering [S1].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
