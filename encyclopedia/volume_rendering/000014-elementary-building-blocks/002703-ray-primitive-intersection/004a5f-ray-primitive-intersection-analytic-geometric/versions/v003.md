# Ray-Primitive Intersection (Analytic/Geometric)


## Overview
This family computes exact intersection parameters between camera rays and standard geometric primitives by solving their implicit or parametric equations [G]. It sets itself apart from iterative approaches like [Ray-Marching](52-ray-marching.md) by avoiding root-finding loops, and from bounding volume methods like [Ray-Volume and Bounding Box Intersection](10057-ray-volume-and-bounding-box-intersection.md) by operating directly on specific primitive shapes [G].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Sphere Intersection](1620-sphere-intersection.md)
Intersects camera rays with spherical geometry to determine entry and exit points, then applies Beer's Law along the chord length for volumetric rendering.

### [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md)
Determines entry and exit points for rays traversing tetrahedral cells, exploiting the primitive's topology for early termination and shared edge computations in unstructured grids.

### [Ray-Object Intersection Test](3604-ray-object-intersection-test.md)
Returns the closest valid intersection distance or a failure indicator, serving as the core decision unit that filters non-intersections and self-intersections via epsilon thresholds.

### [Ray-Primitive and Segment Intersection](6230-ray-primitive-and-segment-intersection.md)
Optimizes multiple ray-primitive evaluations along a single ray by establishing a strict processing order based on bounding box distances to reduce computational load.

### [Ray-Plane/Rectangle Intersection](6498-ray-plane-rectangle-intersection.md)
Computes intersections with planar primitives using linear algebra and plane equations, distinguishing itself from curved primitives by avoiding quadratic surface definitions.

### [Ray-Triangle Intersection](11312-ray-triangle-intersection.md)
Computes intersections with triangular meshes using barycentric coordinate validation, explicitly testing against finite arbitrary surfaces rather than infinite planes.

### [Coordinate System and Parametric Intersection](11367-coordinate-system-and-parametric-intersection.md)
Provides geometric primitives for structured voxel grids and approximate intersections with complex volumetric structures, relying on angular decomposition and scan-plane approximations.

## Background
Not given in the sources.

## Sources
