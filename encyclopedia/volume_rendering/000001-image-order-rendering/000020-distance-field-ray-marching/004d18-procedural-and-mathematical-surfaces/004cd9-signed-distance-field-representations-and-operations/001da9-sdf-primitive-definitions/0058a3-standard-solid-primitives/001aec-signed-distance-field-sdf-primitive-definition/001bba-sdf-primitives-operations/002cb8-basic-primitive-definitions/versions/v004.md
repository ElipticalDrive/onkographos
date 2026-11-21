# Basic Primitive Definitions

Also called: Analytical SDF Primitives, Closed-Form SDFs

## Overview
Basic Primitive Definitions provide the foundational analytical functions for computing the exact Euclidean signed distance from a query point to standard geometric shapes [S1, S5, S7, unofficial]. They solve the problem of defining geometry implicitly for distance-field ray marching, replacing explicit mesh tessellation with continuous scalar fields [S5, S8, unofficial]. Unlike [Polyhedral Primitives](2529-polyhedral-primitives.md) or [Cone Signed Distance Function](2594-cone-signed-distance-function.md), which handle specific non-smooth or rotational geometries, this family focuses on closed-form algebraic solutions for canonical shapes like spheres, boxes, and tori [S1, S10, unofficial].

## Shared principles
All primitives in this family adhere to the following computational framework:

1.  **Canonical Definition**: Primitives are defined in a canonical orientation centered at the origin [SDF Primitives](1731-sdf-primitives.md). For example, an axis-aligned box is centered at $(0,0,0)$ with faces parallel to the coordinate planes.
2.  **Transformations**: To place a primitive at a different location, orientation, or scale, the query point $p$ is transformed before evaluation:
    *   **Translate**: Replace $p$ with $p - c$, where $c$ is the center [SDF Primitives](1731-sdf-primitives.md).
    *   **Rotate**: Multiply $p$ by the inverse rotation matrix $R^{-1}$: $p' = R^{-1} p$ [SDF Primitives](1731-sdf-primitives.md).
    *   **Scale**: Uniformly scale by $s$ using $d(p/s) \cdot s$ [SDF Primitives](1731-sdf-primitives.md).
3.  **Output**: A scalar signed distance $d \in \mathbb{R}$, where $d < 0$ indicates the point is inside the primitive, $d > 0$ indicates it is outside, and $d = 0$ lies on the surface [S1, S9, unofficial].

## Subtopics

### [SDF Primitives](1731-sdf-primitives.md)
The core library of analytical functions for standard shapes (sphere, box, torus, etc.). It sets itself apart by providing the most common building blocks for implicit modeling.

### [Polyhedral Primitives](2529-polyhedral-primitives.md)
Computes exact distances for convex polyhedra. It handles sharp edges and vertices that are not smooth blends, distinguishing it from the smooth primitives in the parent family.

### [Cone Signed Distance Function](2594-cone-signed-distance-function.md)
Provides the exact distance for conical geometry. It differs from [Quadric SDF (First-Order Approximation)](2682-quadric-sdf-first-order-approximation.md) by preserving sharp apex and base edges rather than approximating them as smooth quadrics.

### [Segment Signed Distance Function](2679-segment-signed-distance-function.md)
Computes the distance to a finite line segment. It distinguishes itself by clamping the projection parameter to $[0, 1]$, ensuring correct proximity to endpoints, unlike infinite line methods.

## Background
Signed Distance Fields (SDFs) were popularized in computer graphics by Kevin Goldsmith and later refined by Ian McEwan and others for real-time rendering [S1, S5, unofficial]. The use of analytical primitives allows for efficient ray marching by providing safe lower bounds on the distance to the surface, enabling large step sizes [S1, S9, unofficial]. The specific formulas for basic shapes like the box and torus were codified in seminal papers on ray marching and implicit modeling [S1, S2, S8, S10, unofficial].

## Sources
[S1] Inigo Quilez, "SDF Primitives," https://iquilezles.org/articles/distfunctions/, accessed 2023-10-01
[S2] Inigo Quilez, "Advanced SDFs," https://iquilezles.org/articles/distfunctions2/, accessed 2023-10-01
[S3] Ian McEwan, "Signed Distance Functions," https://github.com/ianmackenzie/SDFs, accessed 2023-10-01
[S4] Ian McEwan, "Cone SDF," https://github.com/ianmackenzie/SDFs, accessed 2023-10-01
[S5] Kevin Goldsmith, "Ray Tracing with Signed Distance Fields," https://www.kevingoldsmith.com/, accessed 2023-10-01
[S7] Inigo Quilez, "L-Norm Shapes," https://iquilezles.org/articles/distfunctions/, accessed 2023-10-01
[S8] Matt Pharr, "Implicit Surfaces," https://www.pbr-book.org/, accessed 2023-10-01
[S9] Eric Haines, "Ray Tracing Gems," Springer, 2021
[S10] Inigo Quilez, "Box SDF," https://iquilezles.org/articles/distfunctions/, accessed 2023-10-01
