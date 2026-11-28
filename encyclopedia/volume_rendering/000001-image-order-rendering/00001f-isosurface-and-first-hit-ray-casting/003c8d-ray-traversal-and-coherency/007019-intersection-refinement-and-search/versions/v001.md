# Intersection Refinement and Search

Also called: Bisection Refinement, Newton Refinement, Binary Search Surface Tracing

## Overview
This family of methods renders implicit isosurfaces directly from volumetric data by detecting the first ray-surface intersection and refining its position to sub-voxel accuracy, thereby eliminating the "staircasing" artifacts inherent in fixed-step ray marching . It differs from [Algebraic Surface Raycasting](1601-algebraic-surface-raycasting.md) by relying on numerical root-finding rather than analytic polynomial solving, and from [Direct Surface Rendering with Intersection Buffer](2211-direct-surface-rendering-with-intersection-buffer.md) by computing the intersection per-pixel in image order rather than via a global buffer .

## Shared principles
All methods in this family follow a two-phase traversal strategy:
1.  **Coarse Detection:** The ray is marched with a fixed step size $h$ (or brick-aligned steps) until a sign change in the scalar field $g(t) = f(\mathbf{o} + t\mathbf{d}) - \tau$ is detected, or the field value crosses the threshold $\tau$ .
2.  **Refinement:** The interval $[t_a, t_b]$ bracketing the intersection is refined to reduce discretization error . The hit position is computed as $\mathbf{p}_{hit} = \mathbf{o} + t_{hit}\mathbf{d}$ [G].
3.  **Shading:** Surface normals are computed via gradients $\nabla f$ (analytic, finite differences, or SDF-specific) and used in local lighting models (Phong/Blinn-Phong) .

## Subtopics

### [Volumetric Isosurface Rendering with Intersection Refinement](958-volumetric-isosurface-rendering-with-intersection-refinement.md)
Uses bisection or linear interpolation to refine the hit position after coarse ray marching. Choose this for general scalar volumes where analytic gradients are unavailable. It is the standard approach for basic isosurface raycasting.

### [Robust Iso-surface Rendering via Bisection and Newton Refinement](1309-robust-iso-surface-rendering-via-bisection-and-newton-refine.md)
Combines bisection with Newton's method for sub-pixel accuracy, using Lipschitz constants for robustness. Choose this when high precision is required or when the scalar field is defined by a function with known derivatives.

### [GVDB Raycasting with Binary Search Surface Tracing](2189-gvdb-raycasting-with-binary-search-surface-tracing.md)
Integrates binary search with GVDB sparse voxel traversal, using SDF gradients for shading. Choose this for sparse datasets where brick-based traversal is necessary to maintain performance.

## Background
The technique originated as a solution to the aliasing artifacts of early ray marching . It is widely used in medical visualization and scientific rendering where explicit mesh extraction is undesirable .

