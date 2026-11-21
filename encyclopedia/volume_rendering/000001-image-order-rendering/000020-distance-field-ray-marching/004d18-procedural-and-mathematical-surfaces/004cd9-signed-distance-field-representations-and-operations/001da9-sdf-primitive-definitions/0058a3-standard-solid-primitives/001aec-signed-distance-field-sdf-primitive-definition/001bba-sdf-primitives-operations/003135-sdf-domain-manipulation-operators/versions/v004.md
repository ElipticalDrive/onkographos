# SDF Domain Manipulation Operators

Also called: Domain Manipulation Operators, SDF Modifiers, SDF domain transformations, Shape Transformations, Space Partitioning, pSomething operators

## Overview
SDF Domain Manipulation Operators are a family of functions that modify the input query point $\mathbf{p}$ before evaluating a base Signed Distance Function (SDF) primitive. By warping the coordinate space, these operators create complex geometric structures such as periodic repetitions, reflections, twists, and elongations without altering the underlying primitive definition [S1, unofficial][S3, unofficial]. Unlike [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) which combine multiple SDFs by modifying the output distance value, domain operators modify the input coordinate $\mathbf{p}$ [1730-sdf-domain-manipulation-operators.md]. They are distinct from [SDF Rounding](1829-sdf-rounding.md) which adjusts the distance scalar, and [SDF Primitives](1731-sdf-primitives.md) which define base shapes [S1, unofficial][S2, unofficial].

## Shared principles
All domain manipulation operators follow a common mechanism: they accept a query point $\mathbf{p}$ (and often additional parameters) and return a transformed point $\mathbf{p}'$ (or modify $\mathbf{p}$ in place) which is then passed to the base SDF evaluation.

**Convention**:
- The input position $\mathbf{p}$ is typically the first argument and is modified in place (`inout`) [S1, unofficial][S3, unofficial].
- For operators acting on subsets of dimensions, specific components are supplied (e.g., `<p.x>` or `<p.zx>`) [S1, unofficial].
- Cell index 0 implies $\mathbf{p}$ is unchanged and cells are centered on the origin, so objects do not need to be moved to fit [S1, unofficial].
- Many operators return a cell index or identifier for optional use, such as a random seed for noise variation [S1, unofficial][S3, unofficial].

**Mathematical Basis**:
The general form is $d_{result}(\mathbf{p}) = d_{base}(T(\mathbf{p}))$, where $T$ is the transformation function. The transformation $T$ partitions space into cells or warps the metric to preserve volume and shape continuity [S1, unofficial].

## Subtopics

### [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md)
[CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) is a technique for constructing complex geometric shapes from simple SDF primitives using boolean logic. It maps set-theoretic operations (Union, Intersection, Difference) directly to arithmetic operators on scalar distance values, allowing the entire scene to be represented as a single composite function. Unlike domain manipulation operators which modify the input coordinate $\mathbf{p}$ [1730-sdf-domain-manipulation-operators.md], CSG operations modify the output distance value, combining multiple independent SDFs into one. This method is distinct from polygonal CSG because it operates on continuous fields, enabling smooth blending and immediate integration with ray marching.

### [SDF Rounding](1829-sdf-rounding.md)
[SDF Rounding](1829-sdf-rounding.md) is a geometric transformation that rounds the edges of a shape by a specified radius. It operates by subtracting a constant distance value from the SDF result, distinguishing it from [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) which modify the coordinate space, and [SDF Primitives](1731-sdf-primitives.md) which define base shapes.

### [Elongation](2440-elongation.md)
[Elongation](2440-elongation.md) is a domain manipulation operator for Signed Distance Functions (SDFs) that modifies the query point to create elongated or tapered geometric structures, such as skeletal branches or organic limbs. It addresses the geometric distortion inherent in standard SDF primitives by stretching the distance field along a specific axis. Unlike [SDF Rounding](1829-sdf-rounding.md) which smooths corners, or [Finite Repetition](2869-finite-repetition.md) which tiles space, elongation warps the metric to preserve volume and shape continuity in tapered objects.

### [Finite Repetition](2869-finite-repetition.md)
[Finite Repetition](2869-finite-repetition.md) creates a bounded grid of repeated instances of a base Signed Distance Function (SDF) within a specified limit. Unlike infinite tiling operators, it restricts the replication count to a finite number of cells in each dimension, effectively carving out a "window" into an infinite periodic structure. It is chosen when a localized cluster of objects is required without the computational cost or visual artifacts of infinite repetition extending to infinity.

### Boolean Subtraction with Domain Repetition
Creating complex geometry by subtracting domain-repeated smaller objects from a non-repeated larger object (e.g., domain repeated cubes subtracted from a larger sphere) [S4, unofficial]. This combines domain repetition with subtracting distance fields from each other to create complex objects [S4, unofficial].

### Symmetry Y
Reflects an SDF across the XZ plane [S5, unofficial]. The step is to take the absolute value of the Y component of $\mathbf{p}$ [S5, unofficial].

### Symmetry X
Reflects an SDF across the YZ plane [S5, unofficial]. The step is to take the absolute value of the X component of $\mathbf{p}$ [S5, unofficial].

### Symmetry Z
Reflects an SDF across the XY plane [S5, unofficial]. The step is to take the absolute value of the Z component of $\mathbf{p}$ [S5, unofficial].

## Background
The concept of manipulating the domain of a function to generate geometry is foundational in procedural modeling and ray marching. The specific "pSomething" (pMod, pMirror, pR) naming convention and implementation details are popularized by Inigo Quilez's work on distance functions [S1, unofficial][S3, unofficial]. These operators allow for the creation of intricate patterns like grids, spirals, and twisted structures by simply transforming the input coordinates before the distance calculation. The technique is widely used in real-time graphics, shader coding, and procedural generation due to its computational efficiency and lack of need for explicit mesh data.

## Sources

[S1] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S2] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S3] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S4] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S5] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
