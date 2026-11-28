# Ray-Primitive Intersection (Analytic/Geometric)

Also called: Analytic Ray-Primitive Intersection, Closed-form Ray Intersection, Geometric Ray Tracing

## Overview
This family provides exact, closed-form solutions for the intersection of a ray with standard geometric primitives (spheres, planes, triangles, tetrahedra) by solving the underlying algebraic equations derived from their implicit or parametric definitions. It differs from iterative methods like [Ray-Marching](52-ray-marching.md) or [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) by avoiding root-finding loops, and from [Ray-Volume and Bounding Box Intersection](10057-ray-volume-and-bounding-box-intersection.md) by handling specific primitive shapes rather than axis-aligned bounding volumes. It serves as the foundational geometric primitive test for ray tracing and volume rendering pipelines.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Sphere Intersection](1620-sphere-intersection.md)
Handles spherical primitives using quadratic solutions to find entry and exit points, essential for volumetric spheres and simple scene geometry.

### [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md)
Intersects rays with tetrahedral cells, critical for volume rendering on unstructured grids and finite element analysis.

### [Ray-Plane/Rectangle Intersection](6498-ray-plane-rectangle-intersection.md)
Solves for intersections with planar primitives (planes, rectangles) by reducing the problem to linear or quadratic equations, often used for bounding volumes or ground planes.

### [Ray-Triangle Intersection](11312-ray-triangle-intersection.md)
Computes intersections with triangular meshes using barycentric coordinates, the standard primitive for polygonal surface rendering.

## Background
Analytic ray-primitive intersection is a cornerstone of computer graphics, originating with the development of ray tracing algorithms in the early 1980s. The geometric projection method for spheres and the Möller–Trumbore algorithm for triangles are standard implementations found in virtually all modern renderers. These methods are used in offline rendering, real-time ray tracing, and scientific visualization involving unstructured grids.

## Sources
