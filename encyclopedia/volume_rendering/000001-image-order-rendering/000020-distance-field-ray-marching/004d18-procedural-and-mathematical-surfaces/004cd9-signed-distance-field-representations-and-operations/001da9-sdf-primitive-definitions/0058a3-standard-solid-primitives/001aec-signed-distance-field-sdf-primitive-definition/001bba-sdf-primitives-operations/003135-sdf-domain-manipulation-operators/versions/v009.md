# SDF Domain Manipulation Operators

Also called: Domain Manipulation Operators, SDF Modifiers, SDF domain transformations, Shape Transformations, Space Partitioning, pSomething operators

## Overview
SDF Domain Manipulation Operators modify the input position vector $\mathbf{p}$ before it is passed to a base Signed Distance Function (SDF) primitive, effectively transforming the coordinate space to create periodic, mirrored, reflected, or stretched structures [S1, S3, unofficial]. They solve the problem of generating complex geometric patterns (such as grids, twists, or elongated shapes) from simple primitives without altering the primitive's internal distance calculation logic. They differ from [SDF Rounding](1829-sdf-rounding.md) and [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) by operating on the input position vector $\mathbf{p}$ (domain transformation) rather than on the scalar distance output (range transformation) [S1, S3, unofficial].

## Shared principles
All domain manipulation operators follow a common evaluation pattern:
1.  **Input:** A query point $\mathbf{p} \in \mathbb{R}^3$ and a base SDF function $d_{base}(\mathbf{p})$.
2.  **Transformation:** The operator modifies $\mathbf{p}$ (either in-place or via a new variable $\mathbf{p}'$) using arithmetic, trigonometric, or logical operations to map the query point into a transformed coordinate system.
3.  **Evaluation:** The base SDF is evaluated at the transformed point: $d_{result} = d_{base}(\mathbf{p}')$.
4.  **Output:** The resulting signed distance is returned. Some operators also return a cell index or identifier for use as a random seed or material ID [S1, S3, unofficial].

**Convention:**
*   The input position $\mathbf{p}$ is often passed as an `inout` argument, modified in place [S1, S3, unofficial].
*   For operators acting on subsets of dimensions, specific components are supplied (e.g., `<p.x>` or `<p.zx>`) [S1, unofficial].
*   Cell index 0 implies $\mathbf{p}$ is unchanged and cells are centered on the origin, so objects do not need to be manually translated to fit [S1, unofficial].

## Subtopics

### [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md)
CSG Operations combine multiple SDFs into a single implicit function using Boolean logic (Union, Intersection, Difference). It differs from this family by operating on scalar field values rather than transforming the input position vector $\mathbf{p}$. It differs from [Constructive Solid Geometry (CSG) on Level Sets](1627-constructive-solid-geometry-csg-on-level-sets.md) by providing specific implementation strategies for discrete voxel buffers and classification look-up tables, whereas this method focuses on the direct arithmetic composition of scalar fields for ray marching. See [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md).

### [SDF Rounding](1829-sdf-rounding.md)
SDF Rounding modifies the SDF by subtracting a constant scalar value from the computed distance, effectively rounding hard edges and corners. It operates directly on the scalar distance field output, unlike domain operators which transform $\mathbf{p}$. See [SDF Rounding](1829-sdf-rounding.md).

### [Elongation](2440-elongation.md)
Elongation stretches a primitive along a specific axis or vector by modifying the query position $\mathbf{p}$ before evaluation. It sets itself apart by focusing on linear stretching rather than edge smoothing or tiling. See [Elongation](2440-elongation.md).

### [Finite Repetition](2869-finite-repetition.md)
Finite Repetition creates a bounded, periodic tiling of a base SDF within a specified limit, preventing infinite replication. It applies a simple clamping operation to the cell index, making it computationally cheaper than infinite repetition variants. See [Finite Repetition](2869-finite-repetition.md).

### Boolean Subtraction with Domain Repetition
This technique creates complex geometry by subtracting domain-repeated smaller objects from a non-repeated larger object (e.g., domain repeated cubes subtracted from a larger sphere) [S4, unofficial]. It combines domain repetition with subtracting distance fields from each other to create complex objects [S4, unofficial].

### Symmetry Y
Reflects an SDF across the XZ plane by taking the absolute value of the Y component of $\mathbf{p}$ [S5, unofficial].

### Symmetry X
Reflects an SDF across the YZ plane by taking the absolute value of the X component of $\mathbf{p}$ [S5, unofficial].

### Symmetry Z
Reflects an SDF across the XY plane by taking the absolute value of the Z component of $\mathbf{p}$ [S5, unofficial].

## Background
The concept of manipulating the domain of a function to generate geometric patterns is fundamental to procedural modeling and ray marching. The specific "pMod" and "pMirror" style operators, often used in GLSL shaders for real-time rendering, were popularized in the context of signed distance field ray marching for creating intricate procedural scenes [S1, S3, unofficial]. The term "Domain Manipulation Operators" encompasses a family of functions that partition space into cells and return a cell index for optional use as a random seed [S1, S3, unofficial]. These operators allow for the creation of periodic, mirrored, or reflected spatial structures without explicit mesh data [S1, S3, unofficial].

## Sources

[S1] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S2] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S3] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S4] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S5] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
