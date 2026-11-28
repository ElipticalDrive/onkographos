# Polyhedral Primitives
## Overview
This family computes the exact signed distance from a query point to the surface of convex polyhedral primitives [S1, unofficial]. It sets itself apart from [SDF Primitives](1731-sdf-primitives.md) by handling non-smooth vertices and edges with exact Euclidean distance calculations rather than smooth blends, and from [Cone Signed Distance Function](2594-cone-signed-distance-function.md) by supporting arbitrary planar faces rather than rotational symmetry [S1, unofficial].
## Shared principles
Builds on [Basic Primitive Definitions](5346-basic-primitive-definitions.md) [G].
## Subtopics

### [Octahedron Signed Distance Function](2339-octahedron-signed-distance-function.md)
Computes the exact signed distance from a point to a regular octahedron centered at the origin with size parameter $s$. It is distinguished from its siblings by its specific symmetry group and face geometry, using coordinate permutation to handle its six faces efficiently. Choose this primitive when high symmetry and axis-aligned orientation are required.

### [Hexagonal Prism Signed Distance Function](2348-hexagonal-prism-signed-distance-function.md)
Computes the signed distance from a query point to a hexagonal prism aligned with the Z-axis, defined by its cross-sectional radius and half-height. It differs from the [Triangular Prism SDF](2865-triangular-prism-signed-distance-function.md) by utilizing a 6-fold rotational symmetry to reflect the query point into a fundamental domain. Choose this primitive for extruded hexagonal geometry.

### [Triangular Prism Signed Distance Function](2865-triangular-prism-signed-distance-function.md)
Computes the signed distance from a query point to a triangular prism aligned with the Z-axis, where the cross-section is an equilateral triangle. It differs from other polyhedral primitives by using a specific angular projection for the triangular base rather than a generic polygon clipping approach. Choose this primitive for right-prisms with triangular cross-sections.

### [Dodecahedron Modulo](3737-dodecahedron-modulo.md)
Computes the signed distance to a regular dodecahedron centered at the origin by folding 3D space into a single fundamental region using a sequence of reflections and rotations, then calculating the distance to the nearest face within that region. It differs from sibling primitives by utilizing a complex, multi-step symmetry folding process specific to the dodecahedron's icosahedral symmetry group. Choose this primitive for exact dodecahedral SDFs without tessellation.
#### Pyramid Signed Distance Function
**Definition**: Computes the signed distance function for a pyramid with height $h$ [S1, unofficial].
**Steps**: Calculate $m^2 = h^2 + 0.25$ [S1, unofficial]. Take the absolute value of the XZ components of $p$ [S1, unofficial]. Swap X and Z if $Z > X$ to fold the pyramid into a fundamental domain [S1, unofficial]. Subtract 0.5 from X and Z [S1, unofficial]. Calculate $q$ as a transformed vector [S1, unofficial]. Calculate $s$ as the positive part of $-q.x$ [S1, unofficial]. Calculate $t$ as the clamp of $(q.y - 0.5 p.z) / (m^2 + 0.25)$ to $[0, 1]$ [S1, unofficial]. Calculate $a$ and $b$ as squared distances to edges [S1, unofficial]. Calculate $b$ as squared distance to the other edge [S1, unofficial]. Calculate $d^2$ as the minimum of $a$ and $b$, or 0 if inside [S1, unofficial]. Return the final signed distance [S1, unofficial].
#### Rhombus Signed Distance Function
**Definition**: Computes the signed distance function for a rhombus with semi-axes $s$ and height $h$ [S1, unofficial].
**Steps**: Calculate $f$ using the dot product of $s$ and $s - 2 p.xz$, clamped to $[-1, 1]$ [S1, unofficial]. Calculate $q.x$ as the length of the vector from the rhombus edge minus the radius, multiplied by the sign [S1, unofficial]. Return the signed distance using the min/max/length formula for box-like shapes [S1, unofficial].

## Background
The methods are implemented in open-source ray tracing repositories [S1, unofficial].
## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
