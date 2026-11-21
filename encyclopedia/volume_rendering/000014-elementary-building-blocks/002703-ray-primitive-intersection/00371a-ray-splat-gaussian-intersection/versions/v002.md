# Ray-Splat/Gaussian Intersection

Also called: Ray-Gaussian Intersection, Ray-Ellipsoid Intersection

## Overview
This family defines the geometric primitives and intersection algorithms required to determine the interaction between a viewing ray and 3D Gaussian splats, which are the fundamental primitives in Gaussian Splatting rendering. It solves the problem of efficiently identifying which splats contribute to a pixel and computing their depth and opacity for alpha blending. It is distinguished from sibling methods by its focus on ellipsoidal (Gaussian) volumes rather than polyhedral or spherical primitives, often leveraging hierarchical structures like BVHs or specific plane-intersection techniques for numerical stability.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model.

**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model.

**Parallel Execution**
Computations are mapped to parallel threads. Each thread typically processes one or more pixels or samples. Memory access patterns are optimized for coalescing. Synchronization is used to manage data dependencies between threads.

## Subtopics

### [Ray-Splat Intersection](6816-ray-splat-intersection.md)
Defines the core geometric traversal and intersection logic using a BVH, supporting both approximate spherical and true ellipsoid intersection tests.

### [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md)
Specializes in equirectangular panoramas by using a numerically stable plane-intersection approach to avoid covariance matrix inversion.

### [Splat Raycasting Dispatch](6924-splat-raycasting-dispatch.md)
Focuses on the computational kernel and buffer management for batched ray-splat intersection tests, optimized for specific hardware backends.

### Ray Marching with RBF Intersection
This variant intersects Radial Basis Function (RBF) density representations with slices perpendicular to the view direction. The slices are rendered from far to near using alpha blending (`GL_ONE` and `GL_SRC_ALPHA`). For each pixel, density $D(x)$ and source radiance $L_x$ are retrieved from 3D textures. The RGB output is set to $D(x)L_x$, and the alpha channel is set to the differential transmittance of the slice, computed as $\exp(-\sigma_t D(x) \Delta x)$ [S1].

## Background
The concept of intersecting rays with Gaussian volumes in the context of rendering inhomogeneous media was explored in early real-time rendering research. Specifically, Gradient-based Interpolation and Sampling for Real-time Rendering of Inhomogeneous, Single-scattering Media introduced ray marching techniques using RBFs for density fields [S1]. While modern 3D Gaussian Splatting (3DGS) popularized the use of 3D Gaussians as explicit scene primitives, the underlying intersection logic for these ellipsoidal volumes has evolved to support real-time rasterization and raycasting variants.

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
