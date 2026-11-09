# Ray-Plane/Rectangle Intersection
Also called: Ray-Plane Intersection, Ray-Rectangle Intersection, Axis-Aligned Rectangle Hit Test

## Overview
This family computes the geometric intersection between a viewing ray and planar primitives, ranging from infinite planes to bounded rectangles and squares. It solves the problem of determining entry and exit points or surface normals for flat geometry, which is fundamental for clipping, bounding, and shading in volume rendering. Unlike [Sphere Intersection](1620-sphere-intersection.md) which uses quadratic solving for curved surfaces, or [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) which handles general finite triangles, this family typically relies on linear plane equations or axis-aligned projections, offering lower computational cost for aligned primitives.

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
Computes the intersection of a ray with an infinite plane defined by a point and a normal, typically used as a ground plane separate from the scene's BVH for ray intersection testing. It returns the intersection distance and fills intersection details, or rejects if the ray is parallel to the plane or falls outside the valid ray segment. Unlike [Ray-box intersection](776-ray-box-intersection.md) which handles bounded volumes, this method handles an unbounded surface, and unlike [General Ray-Triangle Intersection (Plücker Coordinates)](909-general-ray-triangle-intersection-pl-cker-coordinates.md) which handles finite triangles, this handles the entire plane.

### Ray-Square Intersection
#### definition
- [S1, unofficial] Ray-squared plane intersection test that computes intersection with a square primitive defined by corner, v1, v2, and normal.
#### step
- [S1, unofficial] Compute intersection point: p = ray.o + ray.d * tHit.
- [S1, unofficial] Check if point is inside square: planar_hitpt_vector = p - corner, alpha = dot(w, cross(planar_hitpt_vector, v2)), beta = dot(w, cross(v1, planar_hitpt_vector)), where w = cross(v1, v2) / dot(cross(v1, v2), cross(v1, v2)). Return false if !(alpha > 0.0 && alpha < 1.0) || !(beta > 0.0 && beta < 1.0).

### Axis-Aligned Rectangle Hit Test
#### step
- [S2, unofficial] Compute intersection parameter `t` by solving the plane equation: `t = (rect.K - r.Origin.fixed_axis) / r.Direction.fixed_axis`. Check if `t` is within `[tMin, tMax]`.
- [S2, unofficial] Compute the other two coordinates at `t`: e.g., for XYRect, `x = r.Origin.X + t*r.Direction.X` and `y = r.Origin.Y + t*r.Direction.Y`. Check if `x` and `y` are within `[X0, X1]` and `[Y0, Y1]`.
- [S2, unofficial] On hit, set `rec.T = t`, `rec.P = r.At(t)`, `rec.Mat = rect.Mat`. Compute outward normal (e.g., `{0,0,1}` for XYRect) and call `rec.SetFaceNormal(r, outwardNormal)` to orient it correctly.
#### layout
- [S2, unofficial] Rectangles are defined by structs `XYRect`, `XZRect`, `YZRect`. Each contains `X0, X1, Y0, Y1` (bounds on two axes), `K` (constant coordinate on the third axis), and `Mat Material`.
#### pitfall
- [S2, unofficial] Division by zero occurs if the ray direction component along the fixed axis is zero. The code relies on `t` becoming `NaN` or `Inf`, which typically fails the `tMin/tMax` bounds check, but explicit zero-checks are omitted.

## Background
The analytical solution for ray-plane intersection is a foundational technique in computer graphics, dating back to the early days of ray tracing in the 1980s. It is widely used in volume rendering for clipping rays against bounding volumes, defining scene boundaries, and computing surface normals for shading. The axis-aligned rectangle variant is common in educational ray tracers and simple scene descriptions due to its computational efficiency.

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
