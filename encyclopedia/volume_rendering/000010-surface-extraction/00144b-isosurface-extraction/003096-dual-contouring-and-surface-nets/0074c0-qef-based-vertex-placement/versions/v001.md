# QEF-based vertex placement

Also called: Quadratic Error Function vertex placement, QEF minimization

## Overview
QEF-based vertex placement determines the position of a single dual vertex inside each grid cell intersecting an isosurface by minimizing a quadratic error function derived from Hermite data (intersection points and surface normals) . It sets itself apart from sibling methods like [Surface Nets](921-surface-nets.md) by optimizing for feature preservation rather than simple averaging, and from [Dual Contouring](778-dual-contouring.md) variants that use brute-force or topological heuristics by providing a unified mathematical framework for vertex optimization .

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid, where each cell contains scalar values or occupancy states at its vertices or centers [G]. They generate a mesh by placing a single vertex inside each grid cell intersecting the isosurface, rather than on cell edges [G]. The vertex position is determined either by optimizing a quadratic error function using Hermite data (intersection points and normals) or by averaging edge intersection points [G].

The core mechanism is the minimization of the Quadratic Error Function (QEF). For a cell $c$, let $E_c$ be the set of active edges where the scalar field changes sign. For each edge $e \in E_c$, let $p_e$ be the intersection point and $n_e$ be the unit normal vector at that point. The QEF is defined as:

$$ E(v) = \sum_{e \in E_c} (n_e \cdot (v - p_e))^2 $$

This can be rewritten in matrix form as $\|Av - b\|^2$, where $A$ is a matrix with rows $n_e^T$ and $b$ is a vector with entries $n_e \cdot p_e$. The optimal vertex position $v_c$ is the solution to the linear system:

$$ A^T A v_c = A^T b $$

Let $M = A^T A$ and $rhs = A^T b$. The matrix $M$ is a $3 \times 3$ symmetric positive semi-definite matrix. The solution is found by solving $M v_c = rhs$. If $M$ is singular or ill-conditioned (e.g., normals are collinear), regularization is applied. Common regularization includes adding a small multiple of the identity matrix to $M$ or using Singular Value Decomposition (SVD) with truncation of small singular values . The resulting vertex is typically clamped to the cell boundaries to ensure it remains within the voxel .

## Subtopics

### [Dual Contouring](778-dual-contouring.md)
Uses QEF minimization to place vertices, preserving sharp features by constructing a dual graph. Choose when feature preservation is critical.

### [Dual Contouring QEF Solver](1424-dual-contouring-qef-solver.md)
Implements QEF solving via regularized SVD with deterministic topology generation. Choose for robust, watertight manifold output.

### [Classical Dual Contouring with Estimated Normals](5802-classical-dual-contouring-with-estimated-normals.md)
Uses SDF gradients for normals instead of Hermite data. Choose when only SDF input is available and sharp features are less critical.

## Background
Originated in the context of dual contouring for isosurface extraction, QEF-based placement was introduced to solve the problem of sharp feature preservation in marching cubes . It allows for the exact reconstruction of creases and apexes by placing vertices directly on sharp features via QEF minimization .

## Sources
