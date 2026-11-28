# Primitive Geometry and Distribution

Also called: Primitive Representation Family

## Overview
This family defines the geometric primitives used to represent 3D scenes in differentiable splatting methods, extending the standard 3D Gaussian ellipsoid to alternative shapes, distributions, and dimensionalities. It solves the limitations of volumetric Gaussian mixtures, such as ambiguous geometry, poor mesh extraction, and inefficient representation of sharp edges or opaque surfaces, by introducing planar disks, fat-tailed distributions, generalized exponential functions, and polyhedral volumes [S1, S2, S3]. It differs from sibling methods by varying the fundamental density function or geometric support of the primitive while maintaining the core splatting pipeline of projection and alpha compositing.

## Shared principles
All child methods share the core requirement of defining a 3D primitive with a position, orientation, scale, and appearance attributes [G]. The standard 3D Gaussian primitive is defined by a mean $\boldsymbol{\mu}$ for position, a covariance $\Sigma$ for shape and orientation, and appearance attributes like Spherical Harmonics coefficients and opacity $\alpha$ [G]. Parameterized as $\Sigma^w = R S S^T R^T$ in world space, where $R$ is rotation and $S$ is scaling [G]. The rendering pipeline projects these 3D primitives to 2D screen space [G]. The covariance transforms via the view matrix $W$ and projection Jacobian $J$: $\Sigma = J W \Sigma^w W^T J^T$ [G]. The integral of the 3D density along the view direction yields the 2D splat density [G].

## Subtopics

### [Direct Gaussian Splatting](3555-direct-gaussian-splatting.md)
Direct Gaussian Splatting (dGS) is a method for N-dimensional splatting that replaces the covariance-derived conditioning step of standard N-DGS with explicit per-primitive parameters, removing the need for per-primitive matrix inversions and regression products during rendering. It solves the computational bottleneck of recovering conditional mean, opacity, and covariance from a joint distribution by directly parameterizing the opacity precision and position regression operator. Unlike N-DGS, which couples position and opacity through shared covariance blocks requiring expensive linear algebra, dGS separates these effects using a Cholesky precision factor for opacity and a factorized query-to-position regression operator with learnable per-dimension coupling. It differs from sibling methods like [3D Student's t Distribution Splatting](4352-3d-student-s-t-distribution-splatting.md), [Generalized Exponential Splatting](4684-generalized-exponential-splatting.md), and [2D Gaussian Splatting](6068-2d-gaussian-splatting.md) by explicitly parameterizing the conditioning step rather than deriving it from a joint covariance matrix. See [Direct Gaussian Splatting](3555-direct-gaussian-splatting.md) for implementation details.

### [3D Student's t Distribution Splatting](4352-3d-student-s-t-distribution-splatting.md)
3D Student's t Distribution Splatting (SSS) is a neural rendering method that replaces the 3D Gaussian primitives of 3D Gaussian Splatting (3DGS) with flexible Student's t distributions and extends the splatting scheme to include both positive (splatting) and negative (scooping) density components. It solves the expressivity limitations of Gaussian mixtures by utilizing fat-tailed distributions and non-monotonic density modeling, achieving higher parameter efficiency (reducing component counts by up to 82%) while maintaining or improving rendering quality. SSS replaces monotonic Gaussian mixtures with Student's t distributions and employs Stochastic Gradient Hamiltonian Monte Carlo (SGHMC) to prevent the degree of freedom parameter $\nu$ from collapsing to local minima. See [3D Student's t Distribution Splatting](4352-3d-student-s-t-distribution-splatting.md) for implementation details.

### [Generalized Exponential Splatting](4684-generalized-exponential-splatting.md)
Generalized Exponential Splatting (GES) is a 3D scene representation method that replaces the Gaussian basis functions of 3D Gaussian Splatting with Generalized Exponential Functions (GEF). It introduces a learnable shape parameter $\beta$ for each primitive to better model signals with sharp edges and high-frequency content, reducing the number of particles required and thus memory footprint while maintaining competitive rendering quality. Unlike standard Gaussian Splatting, which assumes low-pass characteristics, GES allows primitives to adapt their shape to fit high-bandwidth signals like squares and triangles more efficiently. See [Generalized Exponential Splatting](4684-generalized-exponential-splatting.md) for implementation details.

### [2D Gaussian Splatting](6068-2d-gaussian-splatting.md)
2D Gaussian Splatting (2DGS) is a surface reconstruction technique that replaces the volumetric 3D Gaussian ellipsoids of 3D Gaussian Splatting with oriented, zero-thickness 2D disks (surfels). It solves the problem of ambiguous geometry and poor mesh extraction inherent in volumetric representations by aligning primitives with local surface tangent planes, thereby improving multi-view depth consistency and enabling high-resolution normal map recovery. Unlike its sibling [Direct Gaussian Splatting](3555-direct-gaussian-splatting.md), which retains volumetric blobs, 2DGS uses planar primitives to provide unambiguous normals and accurate surface intersection. See [2D Gaussian Splatting](6068-2d-gaussian-splatting.md) for implementation details.

#### LinPrim Octahedron Representation
- **Definition**: A scene representation using transparent polyhedra (octahedra) bounded by triangular faces with homogeneous density, optimized for differentiable volumetric rendering [S1].
- **Steps**: Restrict vertex positions to lie on the coordinate axes relative to the center to prevent degenerate cases and non-triangular faces [S1]. Enforce symmetry by choosing a single distance to describe opposing corners, so each pair of opposite vertices shares the same distance from the center [S1]. Optimize a rotation quaternion for each primitive to allow arbitrary orientation in space [S1].
- **Parameters**: Octahedra memory requirement is 11 floats per primitive (omitting color), comprising center, rotation, corner distances, and opacity [S1]. Spherical Harmonics (SH) coefficients are 48 floats per primitive (Degree 2), stored to capture view-dependent appearance [S1].

#### High-Opacity Gaussians
- **Definition**: A modification where regular capped Gaussian primitives are converted to high-opacity Gaussians to boost modeling of opaque surfaces with fewer primitives [S2].
- **Steps**: Replace the opacity activation function with `abs()` and clamp blending weights to 1 from above during rendering [S2].
- **Parameters**: Activation switch point is the midpoint of training, specifically 15K iterations [S2].

#### Volumetric Shading System (VSS)
- **Definition**: A method that represents opaque objects with Gaussian density distribution to make the visibility function differentiable, suitable for generative pose estimation [S3].
- **Pitfalls**: VSS is hard to handle geometry with high frequency such as sharp corners, and the rendered image is blurry compared with traditional rendering methods [S3].

## Background
The concept of using explicit geometric primitives for differentiable rendering has evolved from neural radiance fields to explicit point-based representations. The LinPrim method, introduced in "LinPrim: Linear Primitives for Differentiable Volumetric Rendering" (2025), proposed transparent polyhedra like octahedra to replace Gaussian blobs, aiming for real-time volumetric rendering with linear primitives [S1]. The High-Opacity Gaussian modification, described in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" (2024), optimized the standard 3DGS pipeline by converting primitives to high-opacity states to better model opaque surfaces [S2]. The Volumetric Shading System (VSS), referenced in "Jrender: An efficient differentiable rendering library based on Jittor" (2023), was an early approach using Gaussian density for visibility functions, though noted for blurring high-frequency geometry [S3].

## Sources
[S1] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
[S2] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
[S3] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
