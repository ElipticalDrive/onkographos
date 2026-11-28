# Primitive Representation

Also called: Gaussian Primitives, Splatting Kernels

## Overview
Primitive Representation defines the geometric and photometric data structures used to approximate a 3D scene in Gaussian Splatting methods. It solves the problem of representing continuous geometry and appearance with discrete, differentiable primitives that can be efficiently rasterized. It sets itself apart from parent methods by explicitly defining the kernel shape (e.g., isotropic, half-space, or hybrid Bézier) and the associated attribute storage (e.g., opacity, normals, control points) that determine how light interacts with the primitive during rendering.

## Shared principles
All child methods share the core requirement of defining a 3D primitive with a position, orientation, scale, and appearance attributes. The standard 3D Gaussian primitive [S1, unofficial] is defined by:
- **Mean ($\boldsymbol{\mu}$):** 3D position vector.
- **Covariance ($\Sigma$):** Defines shape and orientation. Parameterized as $\Sigma^w = R S S^T R^T$ in world space, where $R$ is rotation and $S$ is scaling.
- **Appearance:** Typically Spherical Harmonics (SH) coefficients for color and a scalar opacity $\alpha$.

The rendering pipeline projects these 3D primitives to 2D screen space. The covariance transforms via the view matrix $W$ and projection Jacobian $J$: $\Sigma = J W \Sigma^w W^T J^T$. The integral of the 3D density along the view direction yields the 2D splat density.

## Subtopics

### [3D Half-Gaussian Splatting](9011-3d-half-gaussian-splatting.md)
[3D Half-Gaussian Splatting](9011-3d-half-gaussian-splatting.md) replaces the isotropic Gaussian with a half-space truncated Gaussian to resolve shape and color discontinuities. It adds a split plane normal $\mathbf{n}$ and dual opacities $\alpha_1, \alpha_2$. It is chosen when sharp boundaries are required without switching to surface-based representations.

### [Bézier Gaussian Triangle](11816-b-zier-gaussian-triangle.md)
[Bézier Gaussian Triangle](11816-b-zier-gaussian-triangle.md) combines Bézier triangle surfaces for geometry with 3D Gaussian sub-primitives for appearance. It tessellates curved surfaces into flat triangles to align with the pixel grid, enabling discontinuity-aware blending. It is chosen when explicit geometric boundaries are needed to prevent the blurring inherent in pure Gaussian splatting.

### Gaussian Surfels
Uses zero-thickness oriented surfels [S1, unofficial]. This variant represents primitives as planar disks with normal vectors, often used for surface extraction or meshing, differing from volumetric Gaussians by enforcing zero thickness.

## Background
The concept of Gaussian primitives for rendering originated with the foundational 3D Gaussian Splatting paper [S1, unofficial]. Subsequent variants like 3D-HGS and BG-Triangle emerged to address specific limitations such as Gibbs phenomena at boundaries and aliasing, expanding the primitive representation beyond the standard isotropic Gaussian.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
