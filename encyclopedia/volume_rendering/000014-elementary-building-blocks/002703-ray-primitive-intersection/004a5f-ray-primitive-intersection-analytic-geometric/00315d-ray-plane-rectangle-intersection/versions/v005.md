# Ray-Plane/Rectangle Intersection
## Overview
This family computes the intersection of a ray with planar primitives, ranging from infinite planes to bounded rectangles and squares [G]. It solves the problem of determining entry and exit points for volume rendering traversal or hit testing for surface rendering [G]. It is distinguished from curved primitives (like [Sphere Intersection](1620-sphere-intersection.md)) by relying on linear algebra and plane equations rather than quadratic surface definitions, and from general mesh intersection by operating on simple, analytically defined patches [G].

## Shared principles
See [Ray-Primitive Intersection (Analytic/Geometric)](12732-ray-primitive-intersection-analytic-geometric.md).

## Subtopics

### [Ray-Sphere Intersection](3539-ray-sphere-intersection.md)
Computes entry and exit distances for spherical volumes using a closed-form quadratic solution [G]. It is chosen when bounding spherical primitives or sampling within spherical volumes, avoiding the barycentric checks of triangles or iterative root-finding of implicit surfaces [G].

### [Ray-Cylinder Intersection](5014-ray-cylinder-intersection.md)
Computes entry and exit distances for cylinders by decomposing them into planar faces and a curved lateral surface solved via quadratic equations [G]. It is chosen for cylindrical primitives where bounding box approximations are insufficient [G].

### [Infinite Plane Ray Intersection](5021-infinite-plane-ray-intersection.md)
Computes intersection distance and point for unbounded planes, rejecting if parallel or outside segment bounds [G]. It is chosen for ground planes, sky domes, or clipping planes kept separate from scene BVHs [G].

### [Axis-Aligned Rectangle Hit Test](5279-axis-aligned-rectangle-hit-test.md)
Determines hits on finite, axis-aligned rectangular patches by solving a fixed-axis plane equation and checking two free coordinates against bounds [G]. It is chosen for fast, branch-minimized hit testing on planar surfaces aligned with coordinate axes [G].

### [Plane-Cube Intersection](11468-plane-cube-intersection.md)
Computes the volume of the irregular polyhedron formed by a plane intersecting a cube, constructing an ordered list of face vertices for volume calculation [G]. It is chosen in computational astrophysics for determining partial cell ionization or burning front coverage [G].

### Ray-Square Intersection
#### definition
Ray-squared plane intersection test that computes intersection with a square primitive defined by corner, v1, v2, and normal [G].
#### step
Compute intersection point: p = ray.o + ray.d * tHit [G].
Check if point is inside square: planar_hitpt_vector = p - corner, alpha = dot(w, cross(planar_hitpt_vector, v2)), beta = dot(w, cross(v1, planar_hitpt_vector)), where w = cross(v1, v2) / dot(cross(v1, v2), cross(v1, v2)) [G]. Return false if !(alpha > 0.0 && alpha < 1.0) || !(beta > 0.0 && beta < 1.0) [G].

## Background
The analytical intersection of rays with planes is a foundational technique in computer graphics, dating back to the early days of ray tracing [G]. The infinite plane test is derived directly from the plane equation $\mathbf{N} \cdot (\mathbf{P} - \mathbf{P}_0) = 0$ [G]. Bounded variants, such as axis-aligned rectangles and squares, add spatial constraints to this basic test [G].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
