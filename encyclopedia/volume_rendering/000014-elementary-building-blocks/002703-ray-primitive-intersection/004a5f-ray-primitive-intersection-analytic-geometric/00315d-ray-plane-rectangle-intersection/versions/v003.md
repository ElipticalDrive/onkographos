# Ray-Plane/Rectangle Intersection
Also called: Ray-Plane Intersection, Ray-Rectangle Intersection

## Overview
This family computes the geometric intersection between a viewing ray and planar primitives, ranging from infinite planes to bounded rectangles and squares. It solves the problem of determining entry and exit points for volume rendering traversal or shading calculations on flat surfaces. Unlike [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) which relies on barycentric coordinate checks for triangular domains, or [Ray-Box Intersection](776-ray-box-intersection.md) which uses slab intersection for axis-aligned volumes, this family focuses on planar geometry defined by normals and points, often serving as the foundational test for more complex primitives like cylinders or cubes.

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
Computes the intersection of a ray with an infinite plane defined by a point and a normal, returning the intersection distance and filling an intersection structure, or rejecting if the ray is parallel or outside the ray segment. It is typically used as a ground plane kept separate from the scene's BVH for ray intersection testing.

### [Axis-Aligned Rectangle Hit Test](5279-axis-aligned-rectangle-hit-test.md)
Not given in the sources.

### [Plane-Cube Intersection](11468-plane-cube-intersection.md)
The Plane-Cube Intersection method computes the geometric intersection between an infinite plane and a cube, resulting in an inscribed irregular polygon with 3 to 6 vertices. It is primarily used to calculate the volume of the portion of the cube sliced by the plane, a computation relevant in computational astrophysics for determining partial cell coverage (e.g., by burning fronts) and in solid modeling. Unlike standard ray-primitive intersection which returns a single point or segment, this method constructs the full cross-sectional polygon to enable volume integration.

### Ray-Square Intersection
#### definition
- [S1, unofficial] Ray-squared plane intersection test that computes intersection with a square primitive defined by corner, v1, v2, and normal.
#### step
- [S1, unofficial] Compute intersection point: p = ray.o + ray.d * tHit.
- [S1, unofficial] Check if point is inside square: planar_hitpt_vector = p - corner, alpha = dot(w, cross(planar_hitpt_vector, v2)), beta = dot(w, cross(v1, planar_hitpt_vector)), where w = cross(v1, v2) / dot(cross(v1, v2), cross(v1, v2)). Return false if !(alpha > 0.0 && alpha < 1.0) || !(beta > 0.0 && beta < 1.0).

## Background
Origin: The analytical solution for ray-plane intersection is a standard technique in computer graphics, foundational to ray tracing algorithms introduced by Whitted (1980) and later detailed in standard texts like [PBRT](https://www.pbr-book.org/). The specific implementation details for volumetric path tracing using these primitives are documented in [S1, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
