# SDF Domain Manipulation Operators

Also called: Domain Manipulation Operators, SDF Modifiers, SDF domain transformations, Shape Transformations, Space Partitioning, pSomething operators

## Overview
SDF Domain Manipulation Operators are a family of geometric transformations that modify the input position vector $\mathbf{p}$ before it is evaluated by a base Signed Distance Function (SDF) primitive [S1, S3, unofficial]. By partitioning space into cells or applying continuous transformations to the coordinate space, these operators create periodic, mirrored, reflected, or stretched structures without altering the underlying distance metric's continuity [S1, S3, unofficial]. They set themselves apart from value-domain operators (like [SDF Rounding](1829-sdf-rounding.md)) by acting on the spatial coordinates $\mathbf{p}$ rather than the distance output $d$, and from [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) by composing single primitives into complex patterns rather than combining multiple distinct shapes via Boolean logic.

## Shared principles
All domain manipulation operators in this family share a common evaluation pattern: they accept a query point $\mathbf{p}$ and return a transformed point $\mathbf{p}'$ (and optionally a cell index or symmetry identifier) which is then passed to the base SDF.

**Convention**:
- The input position $\mathbf{p}$ is typically passed as an `inout` argument, modified in place [S1, S3, unofficial].
- For operators acting on subsets of dimensions, specific components are supplied (e.g., `<p.x>` or `<p.zx>`) [S1, unofficial].
- Cell index 0 implies $\mathbf{p}$ is unchanged and cells are centered on the origin, meaning objects do not need to be manually translated to fit the grid [S1, S3, unofficial].

**General Transformation Logic**:
Most operators follow this structure:
1.  **Partitioning**: Determine the cell or region containing $\mathbf{p}$ using division and floor/round operations.
2.  **Mapping**: Map $\mathbf{p}$ to a canonical region (e.g., $[-size/2, size/2]$) using modulo arithmetic or reflection.
3.  **Symmetry/Reflection**: Apply absolute values or sign flips to mirror the geometry across axes [S1, S3, unofficial].
4.  **Evaluation**: The transformed point is used to query the base SDF.

**Mathematical Basis**:
The core mechanism relies on the property that $d_{transformed}(\mathbf{p}) = d_{base}(T(\mathbf{p}))$, where $T$ is the domain transformation [S3, unofficial]. This preserves the Lipschitz-1 property of the SDF if $T$ is an isometry (rotation, reflection, translation) or a uniform scaling [S3, unofficial]. Non-isometric transformations (like non-uniform scaling or twisting) may distort the distance metric, requiring smaller step sizes during ray marching to avoid artifacts [S4, unofficial].

## Subtopics

### [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md)
CSG Operations combine multiple SDFs into a single implicit function using Boolean operations (Union, Intersection, Difference). Unlike domain manipulation which transforms coordinates, CSG operates on the distance values themselves using min/max functions. It sets itself apart from [Stack-Based SDF Editor](1696-stack-based-sdf-editor.md) by focusing on spatial composition of volume data (voxels or classification maps) rather than a procedural stack, and from [CSG Operations on Level Sets](1627-constructive-solid-geometry-csg-on-level-sets.md) by providing concrete implementation strategies for discrete volume representations. Choose this when you need to combine distinct geometric primitives into a single complex shape.

### [SDF Rounding](1829-sdf-rounding.md)
SDF Rounding modifies a primitive by subtracting a constant radius value from its distance output, effectively rounding off hard edges and corners. Unlike [Elongation](2440-elongation.md) which stretches geometry, or [CSG Operations on Implicit Functions](1329-csg-operations-on-implicit-functions.md) which combine shapes, rounding alters the metric of a single primitive to create smooth, filleted surfaces. It is a specific instance of SDF Domain Manipulation Operators that operates on the distance value rather than the input position vector. Choose this when you need to soften the features of a base primitive.

### [Elongation](2440-elongation.md)
Elongation is a domain manipulation operator that stretches a primitive along a specific axis or direction. It solves the problem of creating extended geometric structures (such as beams, tubes, or stretched spheres) from compact base primitives without altering the underlying distance metric's continuity. It sets itself apart from siblings like [SDF Rounding](1829-sdf-rounding.md) by modifying the spatial metric to increase length, whereas rounding modifies the surface geometry to soften corners. Choose this when you need to extend a shape along a vector.

### [Finite Repetition](2869-finite-repetition.md)
Finite Repetition creates a bounded array of an object by repeating a base SDF within a specified limit. Unlike infinite tiling operators, it restricts the replication count to a specific range, preventing geometry from extending to infinity [G]. It differs from [Infinite Repetition](1730-sdf-domain-manipulation-operators.md) by clamping the cell index, and from [Mirrored Repetition](1730-sdf-domain-manipulation-operators.md) by not reflecting the object orientation [G]. Choose this when you need a limited grid of repeated objects.

### Boolean Subtraction with Domain Repetition
Creating complex geometry by subtracting domain-repeated smaller objects from a non-repeated larger object (e.g., domain repeated cubes subtracted from a larger sphere) [S4, unofficial]. Combine domain repetition with subtracting distance fields from each other to create complex objects [S4, unofficial].

### Symmetry Y
Reflects an SDF across the XZ plane [S5, unofficial]. Take the absolute value of the Y component of $\mathbf{p}$ [S5, unofficial].

### Symmetry X
Reflects an SDF across the YZ plane [S5, unofficial]. Take the absolute value of the X component of $\mathbf{p}$ [S5, unofficial].

### Symmetry Z
Reflects an SDF across the XY plane [S5, unofficial]. Take the absolute value of the Z component of $\mathbf{p}$ [S5, unofficial].

## Background
The concept of manipulating the domain of a function to generate geometric patterns originates from procedural modeling and ray marching techniques popularized in the early 2010s [S4, unofficial]. The "pSomething" naming convention (e.g., `pMod`, `pMirror`) was established in community-driven shader code repositories, particularly those focusing on real-time ray marching [S1, S3, unofficial]. These operators allow for the creation of complex scenes (like forests, grids, or symmetrical structures) with minimal computational overhead compared to explicit mesh rendering [S4, unofficial]. The distinction between domain manipulation (transforming $\mathbf{p}$) and value manipulation (transforming $d$) is fundamental to SDF composition, enabling artists and programmers to layer effects like repetition, symmetry, and boolean operations [S1, S3, S4, unofficial].

## Sources

[S1] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S2] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S3] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S4] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S5] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
