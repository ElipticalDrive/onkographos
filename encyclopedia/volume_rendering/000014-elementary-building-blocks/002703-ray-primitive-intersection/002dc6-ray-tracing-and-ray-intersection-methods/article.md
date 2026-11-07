# Ray Tracing and Ray Intersection Methods

Also called: Ray Casting, Ray-Primitive Intersection, Ray Tracing

## Overview
Ray tracing determines the first object intersected by a camera ray by analytically computing intersection distances for simple geometries [S1, unofficial]. It solves the problem of visibility and shading in 3D scenes by finding the closest intersection point along each viewing ray [S1, unofficial]. Unlike [Ray-box intersection](776-ray-box-intersection.md) which handles axis-aligned bounding volumes, or [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) which uses distance fields, this family focuses on exact analytical or iterative root-finding for explicit primitives (triangles, spheres, planes) and isosurfaces. It differs from [Ray-marching](776-ray-box-intersection.md) (implied by context) by providing exact intersection points rather than approximate steps, provided the geometry is simple enough [S1, unofficial].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Approximates isosurface position via iterative bisection. Choose when scalar field sampling is available but analytical derivatives are difficult.

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Analytically solves for intersections with trilinearly interpolated cells. Choose for high-precision isosurfaces in regular grids.

### [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md)
Combines BVH traversal of active voxels with implicit intersection. Choose for crack-free rendering of large rectilinear volumes without polygonization.

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
Uses macro-cell projection and min/max octrees. Choose for accelerating traversal in sparse or hierarchical volumetric data.

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
Dynamically switches between warp-level and shared-memory packet processing. Choose for SIMD architectures with highly divergent or high-workload ray packets.

### Packet Ray List Merging
An optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list to exploit spatial coherence [S2, unofficial]. Phase I traverses the BONO tree, creating lists for each packet of rays instead of individual pixels; if a group overlaps with the current node's projection, the group's list is processed [S2, unofficial]. The upper bound $k$ on the list size is increased compared to single pixels, with best performance achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S2, unofficial].

### Ray Tracing
Entails ray casting but with the additional tracing of secondary rays for shadows and refractions [S3, unofficial]. First proposed by Whitted [S3, unofficial].

### Geometry Integrator
A direct illumination integrator for scene geometry that uses a hardcoded two-sided diffuse BSDF with red reflectance $[0.8, 0.2, 0.2]$ [S4, unofficial]. It performs ray intersection against the scene (`itx : SurfaceInteraction3f = scene.ray_intersect(ray)`), adjusts the normal direction to point against the ray direction (`itx.n = dr.select(dr.dot(itx.n, ray.d) < 0, itx.n, -itx.n)`), loads the BSDF, checks visibility using a shadow ray (`itx_shadow : mi.SurfaceInteraction3f = scene.ray_intersect(rayL)`), and accumulates direct illumination if both primary and secondary rays are valid (`L += dr.select(valid_ray, Li, black)`) [S4, unofficial].

## Background
Ray tracing was first proposed by Whitted [S3, unofficial]. Ray casting is one of the most efficient techniques for surface rendering, particularly when object geometry is simple enough for analytical intersection solutions [S1, unofficial]. When geometry is too complex for analytical solutions, ray-marching is used instead [S1, unofficial]. The method fails when object geometry is too complex for analytical intersection solutions [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S4] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
