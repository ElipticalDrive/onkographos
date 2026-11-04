# Polyhedral Primitives
Also called: Polyhedral SDFs, Polyhedron Signed Distance Functions

## Overview
This family provides exact signed distance functions for convex polyhedra, prisms, and pyramids, enabling precise ray marching [S1, S2, unofficial]. It distinguishes itself from [SDF Primitives](1731-sdf-primitives.md) by focusing on multi-faceted shapes with planar faces rather than simple quadratic forms or domain manipulations [G]. It provides closed-form analytic solutions that exploit geometric symmetries to reduce 3D distance queries to simpler 2D or 1D projections [G].

## Shared principles
All polyhedral SDFs in this family share a common computational structure based on symmetry reduction and extrusion [S1, S2, unofficial].
1. **Symmetry Folding**: Most polyhedra possess reflectional symmetries across coordinate planes. The first step is typically computing the absolute value of the query point components ($q = |p|$) [S1, S2, unofficial]. This folds the 3D space into a fundamental octant (or sector), reducing the number of cases the algorithm must handle [S1, S2, unofficial]. For shapes with rotational symmetry (like prisms), additional reflections or rotations map the point into a smaller fundamental domain (e.g., a 30° sector for hexagons) [S1, S2, unofficial].
2. **Separation of Axes**: For prisms, the distance calculation separates the axial component (usually Z) from the cross-sectional component (XY) [S1, S2, unofficial]. The total distance is computed by combining the distance to the infinite prism ($d_{xy}$) and the distance to the caps ($d_z$) using the extruded shape formula: $d = \min(\max(d_{xy}, d_z), 0) + \text{length}(\max(d_{xy}, d_z, 0))$ [S1, S2, unofficial].
3. **Fundamental Domain Projection**: For non-prismatic polyhedra (octahedra, pyramids), the algorithm projects the folded point onto the closest face, edge, or vertex [S1, S2, unofficial]. This often involves comparing the point's position against linear inequalities defining the faces to determine which region of the Voronoi cell the point lies in [S1, S2, unofficial].
4. **Branch-Free Optimization**: Where possible, implementations use `max`, `min`, and `clamp` operations to replace conditional branches, ensuring consistent performance on GPU architectures [G].

## Subtopics

### [Octahedron Signed Distance Function](2339-octahedron-signed-distance-function.md)
Computes the exact signed distance to a regular octahedron. It is distinguished by its specific symmetry group and face geometry, offering a distinct aesthetic for procedural geometry. Choose this when a smooth, faceted sphere-like shape with 8 triangular faces is required.

### [Hexagonal Prism Signed Distance Function](2348-hexagonal-prism-signed-distance-function.md)
Computes the signed Euclidean distance to a hexagonal prism. It differs from the triangular prism by utilizing six-fold rotational symmetry to fold the query point into a fundamental sector, reducing the distance calculation to a simpler 2D case. Choose this for hexagonal columns or honeycomb structures.

### [Triangular Prism Signed Distance Function](2865-triangular-prism-signed-distance-function.md)
Computes the signed distance to an equilateral triangular prism. It exploits the symmetry of the equilateral base to provide a closed-form, branch-free solution using absolute values and max operations, avoiding iterative projection or complex geometric decomposition [G]. Choose this for tent-like structures or triangular beams.

### Pyramid Signed Distance Function
Computes the signed distance function for a pyramid with height $h$ [S1, unofficial]. It folds the base into a fundamental domain by swapping X and Z if $Z > X$, then transforms the coordinates to project the point onto the pyramid's faces and edges [S1, unofficial]. It calculates distances to the side faces and the base using a combination of linear projections and clamping, returning the minimum squared distance adjusted for the pyramid's geometry [S1, unofficial].

### Rhombus Signed Distance Function
Computes the signed distance function for a rhombus with semi-axes $s$ and height $h$ [S1, unofficial]. It uses a dot product to determine the position relative to the rhombus edges, clamping the result to $[-1, 1]$ [S1, unofficial]. It then constructs a 2D vector representing the distance to the edge and the height, combining them using the standard extruded shape SDF pattern to return the final signed distance [S1, unofficial].

### [Dodecahedron Modulo](3737-dodecahedron-modulo.md)
Transforms a point $p$ to fold space into a dodecahedron cell using a sequence of reflections and rotations. Not given in the sources.

## Background
Signed distance functions for polyhedra are a staple of procedural geometry in computer graphics, popularized by articles such as those by Inigo Quilez [S1, unofficial]. The techniques rely on analytical geometry to derive closed-form solutions for the Euclidean distance to convex polytopes [G]. These methods are widely used in ray marching for real-time rendering, allowing for the creation of complex, smooth, and infinitely detailed geometric structures without the overhead of mesh data structures [G]. The specific implementations for octahedra, prisms, and pyramids are derived from the geometric properties of these shapes, particularly their symmetries and face normals [S1, S2, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
