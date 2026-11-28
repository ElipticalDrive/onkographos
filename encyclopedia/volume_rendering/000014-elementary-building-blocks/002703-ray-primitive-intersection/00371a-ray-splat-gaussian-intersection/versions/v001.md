# Ray-Splat/Gaussian Intersection
Also called: Ray-Gaussian Intersection, Splat Raycasting

## Overview
This family defines methods for determining the intersection between a camera ray and 3D Gaussian splats, which are ellipsoidal primitives defined by position, covariance (or scale/rotation), and opacity. It solves the problem of efficiently identifying which splats contribute to a pixel or picking ray, enabling correct depth sorting and alpha blending in splat-based rendering. It differs from sibling geometric intersection methods (e.g., [Ray-box intersection](776-ray-box-intersection.md)) by handling non-convex, soft-edged volumetric primitives rather than hard-surface boundaries, and from [Ray-Sphere Intersection](3539-ray-sphere-intersection.md) by supporting arbitrary ellipsoidal shapes and opacity falloff.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model.

**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model.

**Parallel Execution**
Computations are mapped to parallel threads. Each thread typically processes one or more pixels or samples. Memory access patterns are optimized for coalescing. Synchronization is used to manage data dependencies between threads.

## Subtopics

### [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md)
SPaGS is a real-time rendering method for 3D Gaussian Splatting (3DGS) that supports arbitrary spherical camera models, specifically equirectangular panoramas, by replacing the standard local affine approximation with a numerically stable ray-splat intersection approach. It solves the problem of rendering artifacts and numerical instability inherent in standard 3DGS when applied to wide-field-of-view or distorted panoramic views, where first-order affine projections fail to accurately represent the Gaussian distribution. Unlike ODGS, which uses suboptimal tangent plane approximations, or 3DGUT, which relies on the Unscented Transform, SPaGS achieves higher quality and speed by representing each viewing ray as the intersection of two perpendicular planes in camera space, thereby avoiding matrix inversion of the covariance.

### [Ray-Splat Intersection](6816-ray-splat-intersection.md)
Ray-Splat Intersection is a method for determining which 3D Gaussian splats in a scene intersect a given camera ray, supporting both true ellipsoid geometry and approximate spherical bounds. It solves the problem of efficient spatial querying for splat-based rendering and interaction (picking) by organizing splats into a Bounding Volume Hierarchy (BVH). Unlike sibling methods that may rely on screen-space projection or specific rendering dispatches, this method explicitly computes geometric intersections in world or local space, allowing for precise depth sorting and visibility testing.

### [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md)
Splat Raycasting Dispatch is the core computational kernel that performs ray-splat intersection tests for Gaussian primitives, returning hit distances and primitive indices. It solves the problem of efficiently determining which splats intersect a given ray within a specified depth range, serving as the bridge between high-level scene management and low-level geometric intersection logic. It is distinguished from its siblings by its specific implementation as a batched, buffer-oriented dispatch function (often compiled to WebAssembly) that operates on pre-extracted subbuffers, whereas [Ray-Splat Intersection](6816-ray-splat-intersection.md) describes the general geometric algorithm and [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md) adapts the coordinate space for spherical projections.

## Background
The methods in this family originate from the rise of 3D Gaussian Splatting (3DGS) for real-time novel view synthesis [S1]. Standard 3DGS relies on screen-space projection and affine approximation, which fails for large FOV or panoramic cameras. SPaGS [S1] introduced plane-based intersection for spherical panoramas. Concurrently, general-purpose ray-splat intersection algorithms [S1, S2, unofficial] were developed for picking and hybrid rendering, utilizing BVHs and ellipsoid approximations. The dispatch model [S1, unofficial] emerged from the need to optimize these calculations in constrained environments like WebAssembly, using fixed-size buffers and lookup tables.

## Sources
[S1] Hahlbohm et al., "Spherical Panoramic Gaussian Splatting," 2025.
[S2] Unofficial documentation for Ray-Splat Intersection implementation.
