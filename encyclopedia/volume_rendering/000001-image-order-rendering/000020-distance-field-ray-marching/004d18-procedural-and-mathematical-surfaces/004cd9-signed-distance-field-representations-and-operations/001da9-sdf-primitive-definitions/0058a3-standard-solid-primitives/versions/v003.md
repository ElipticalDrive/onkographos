# Standard Solid Primitives


## Overview
Standard Solid Primitives provide exact, closed-form Signed Distance Functions (SDFs) for canonical geometric shapes, enabling efficient distance-field ray marching without explicit mesh tessellation [890-signed-distance-field-sdf-primitive-definition.md]. Unlike [Distance to Natural Quadrics](1559-distance-to-natural-quadrics.md), which focuses on the algebraic derivation of quadric surfaces, this family emphasizes the practical implementation of exact distance bounds for spheres, boxes, tori, and their combinations, guaranteeing that sphere tracing steps do not overshoot surfaces [890-signed-distance-field-sdf-primitive-definition.md].

## Shared principles
The family operates on the principle that a scene is defined by a scalar field $f(\mathbf{p})$ where the zero-level set $f(\mathbf{p}) = 0$ represents the surface [7277-implicit-surface-representation.md]. For sphere tracing to be valid, the function must satisfy the Lipschitz condition with constant 1, ensuring $|f(\mathbf{p}_1) - f(\mathbf{p}_2)| \le \|\mathbf{p}_1 - \mathbf{p}_2\|$ [890-signed-distance-field-sdf-primitive-definition.md]. This guarantees that the value $f(\mathbf{p})$ is a safe lower bound on the distance to the surface, allowing the ray marcher to step forward by exactly $f(\mathbf{p})$ without tunneling [890-signed-distance-field-sdf-primitive-definition.md].

Composition is achieved through Boolean operations on the distance values:
*   **Union**: $f_{union}(\mathbf{p}) = \min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [890-signed-distance-field-sdf-primitive-definition.md].
*   **Intersection**: $f_{intersect}(\mathbf{p}) = \max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [890-signed-distance-field-sdf-primitive-definition.md].
*   **Subtraction**: $f_{sub}(\mathbf{p}) = \max(-f_1(\mathbf{p}), f_2(\mathbf{p}))$ [890-signed-distance-field-sdf-primitive-definition.md].

Transformations are applied by mapping the query point $\mathbf{p}$ into the local coordinate space of the primitive using the inverse transformation matrix $M^{-1}$ before evaluation [890-signed-distance-field-sdf-primitive-definition.md].

## Subtopics

### [Signed Distance Field (SDF) Primitive Definition](890-signed-distance-field-sdf-primitive-definition.md)
A method for representing 3D geometry using analytical scalar functions that return the shortest Euclidean distance from a query point $\mathbf{p}$ to the surface of a canonical shape, with a sign indicating whether the point is inside (negative) or outside (positive) the shape. It solves the problem of defining complex scene geometry without explicit mesh tessellation, replacing discrete triangle lists with continuous, closed-form algebraic solutions for primitives like spheres, boxes, and tori. Unlike [Distance to Natural Quadrics](1559-distance-to-natural-quadrics.md), which focuses on the geometric properties of conic sections, SDF primitives provide exact distance bounds required for safe sphere tracing in ray marching.

### [Distance to Natural Quadrics](1559-distance-to-natural-quadrics.md)
Provides closed-form Signed Distance Functions (SDFs) for standard solid modeling primitives, including the plane, sphere, cylinder, cone, and torus. These functions compute the exact Euclidean distance from a query point to the surface of the primitive, enabling valid sphere tracing in volume rendering and constructive solid geometry (CSG). Unlike general implicit surface solvers, these methods offer high efficiency and analytical precision, serving as the foundational building blocks for complex scene composition.

### Cylindrical SDF Primitives
A technique for rendering implicit geometry defined by SDFs using ray marching, specifically for cap cylinder shapes, often involving nested cylinder calculations for complex lighting or structural details [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
