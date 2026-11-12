# Ray-Plane/Rectangle Intersection
Also called: Planar Primitive Intersection, Ray-Plane Tests

## Overview
This family computes the intersection of a ray with planar primitives, ranging from infinite planes to bounded rectangles and squares. It solves the problem of determining entry and exit points for volume rendering traversal or hit testing for surface rendering. It is distinguished from curved primitives (like [Sphere Intersection](1620-sphere-intersection.md)) by relying on linear algebra and plane equations rather than quadratic surface definitions, and from general mesh intersection by operating on simple, analytically defined patches.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Ray-Sphere Intersection](3539-ray-sphere-intersection.md)
Ray-sphere intersection computes the geometric points where a ray intersects the surface of a sphere, providing the entry and exit distances along the ray parameter $t$. It solves the problem of efficiently determining if and where a viewing ray hits a spherical primitive, which is a fundamental building block for volume rendering. Unlike [Ray-Triangle Intersection](909-general-ray-triangle-intersection-pl-cker-coordinates.md) which requires barycentric coordinate checks, or [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) which relies on iterative root-finding, this method uses a closed-form analytical solution to the quadratic equation derived from the sphere's implicit definition.

### [Ray-Cylinder Intersection](5014-ray-cylinder-intersection.md)
Computes the entry and exit distances for a ray intersecting a cylinder defined by `geomSize`, composed of front/back rectangles, top/bottom faces, and a lateral curved surface. It differs from [Ray-box intersection](776-ray-box-intersection.md) by handling the curved lateral surface via quadratic solving and cylindrical coordinate checks.

### [Infinite Plane Ray Intersection](5021-infinite-plane-ray-intersection.md)
Computes the intersection of a ray with an infinite plane defined by a point and a normal, returning the distance and intersection point or rejecting if parallel or outside the ray segment bounds. It is distinct from bounded primitives (like [Axis-Aligned Rectangle Hit Test](5279-axis-aligned-rectangle-hit-test.md)) by having no spatial extent limits, making it suitable for ground planes or sky domes in path tracers.

### [Axis-Aligned Rectangle Hit Test](5279-axis-aligned-rectangle-hit-test.md)
The Axis-Aligned Rectangle Hit Test determines whether a ray intersects a finite, axis-aligned rectangular surface patch. It solves the problem of intersecting a ray with a bounded planar primitive, returning the intersection point, distance, and surface normal if a hit occurs. It is distinguished from infinite plane intersection by its additional bounds checking on the two free axes, and from general rectangle intersection by its strict alignment with the coordinate axes, which allows for a simplified, branch-minimized implementation using a single fixed-axis formula.

### [Plane-Cube Intersection](11468-plane-cube-intersection.md)
This method computes the volume of the irregular polyhedron formed by the intersection of a plane and a cube. It is specifically designed for applications in computational astrophysics, such as determining the fraction of a cell that is partial ionized or covered by a burning front. It extends the approach of Salama & Kolb (2005) by explicitly constructing an ordered list of face vertices to calculate the volume of the resulting inscribed irregular polyhedron.

### Ray-Square Intersection
#### definition
- [S1, unofficial] Ray-squared plane intersection test that computes intersection with a square primitive defined by corner, v1, v2, and normal.
#### step
- [S1, unofficial] Compute intersection point: p = ray.o + ray.d * tHit.
- [S1, unofficial] Check if point is inside square: planar_hitpt_vector = p - corner, alpha = dot(w, cross(planar_hitpt_vector, v2)), beta = dot(w, cross(v1, planar_hitpt_vector)), where w = cross(v1, v2) / dot(cross(v1, v2), cross(v1, v2)). Return false if !(alpha > 0.0 && alpha < 1.0) || !(beta > 0.0 && beta < 1.0).

## Background
The analytical intersection of rays with planes is a foundational technique in computer graphics, dating back to the early days of ray tracing. The infinite plane test is derived directly from the plane equation $\mathbf{N} \cdot (\mathbf{P} - \mathbf{P}_0) = 0$. Bounded variants, such as axis-aligned rectangles and squares, add spatial constraints to this basic test. The specific implementation for volumetric path tracing using these primitives is documented in recent open-source implementations [S1, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
