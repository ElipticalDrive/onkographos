# Basic Primitive Definitions

Also called: Analytical SDF Primitives, Basic Signed Distance Functions

## Overview
Basic Primitive Definitions provide the foundational analytical Signed Distance Functions (SDFs) for simple geometric shapes, serving as the atomic building blocks for distance-field ray marching [G]. They compute the exact Euclidean distance from a query point to the surface of a shape, guaranteeing no underestimation to prevent ray overshooting [G]. Unlike [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) which transform space or [SDF Rounding](1829-sdf-rounding.md) which modifies surfaces, primitives define the base geometry itself [G].

## Shared principles
All basic primitives share a common computational framework [SDF Primitives](1731-sdf-primitives.md):
1. **Canonical Definition**: Primitives are defined centered at the origin with specific orientations (e.g., sphere at $(0,0,0)$, cylinder along Y-axis) [S7].
2. **Transformations**: To place a primitive at a different location, orientation, or scale:
   * **Translate**: Replace query point $p$ with $p - c$, where $c$ is the center [S7].
   * **Rotate**: Multiply $p$ by the inverse rotation matrix $R^{-1}$ before evaluation: $p' = R^{-1} p$ [S7].
   * **Scale**: Uniformly scale by $s$ using $d(p/s) \cdot s$ [S7].
3. **Output**: A scalar signed distance $d \in \mathbb{R}$, where $d < 0$ indicates inside and $d > 0$ indicates outside [S7].

## Subtopics

### [SDF Primitives](1731-sdf-primitives.md)
Analytical SDFs for basic shapes (sphere, box, torus, cylinder, capsule, plane, rhombic triacontahedron). Defines the base geometry using exact Euclidean distance formulas [G].

### [Polyhedral Primitives](2529-polyhedral-primitives.md)
Exact SDFs for convex polyhedra (prisms, pyramids). Uses symmetry folding and separation of axes to reduce 3D queries to simpler projections [G].

### [Cone Signed Distance Function](2594-cone-signed-distance-function.md)
Exact SDF for cones, including base cap and apex. Reduces 3D distance to a 2D problem via rotational symmetry, preserving sharp edges unlike quadric approximations [G].

### [Segment Signed Distance Function](2679-segment-signed-distance-function.md)
SDF for line segments. Computes distance to the closest point on a segment defined by endpoints, handling boundaries explicitly [G].

## Background
The concept of using analytical distance functions for rendering was popularized by Inigo Quilez in his online tutorials, providing efficient, branch-free distance estimators for ray marching [G]. These primitives form the core of modern SDF-based rendering, enabling complex scene construction through boolean operations and domain manipulation [G].

## Sources
[S1] Inigo Quilez, "SDF Primitives," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
[S2] Inigo Quilez, "Cone," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
[S3] Inigo Quilez, "Segment," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
[S4] Inigo Quilez, "Distance to Line Segment," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
[S5] Inigo Quilez, "Torus," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
[S6] Inigo Quilez, "Box," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
[S7] Inigo Quilez, "General Transformations," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
[S8] Inigo Quilez, "Rhombic Triacontahedron," https://iquilezles.org/articles/distfunctions/, fetched 2024-05-20
