# Ray Tracing and Ray Intersection Methods

Also called: Ray Casting, Whitted-Style Ray Tracing, Ray-Primitive Intersection

## Overview
Ray tracing is a rendering technique that determines the color of a pixel by computing the intersection of a viewing ray with scene geometry, handling shadows, reflections, and refractions through recursive secondary rays [S1, unofficial]. It distinguishes itself from simple ray casting by natively supporting these secondary rays spawned at intersection points, rather than just primary viewing rays [S2, unofficial]. The method solves the problem of finding the first visible object along a ray by analytically computing intersection distances for simple geometries [S1, unofficial].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Approximates isosurface position by iteratively halving the interval based on scalar field values. Chosen when higher accuracy than linear interpolation is needed without the overhead of dense ray marching.

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Computes the exact intersection distance $t$ within a single hexahedral voxel by solving the cubic polynomial derived from trilinear interpolation. Chosen for superior performance on silhouette and secondary rays compared to iterative methods.

### [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md)
Combines explicit voxel selection (BVH) with implicit ray-primitive intersection to render crack-free isosurfaces from rectilinear volumes. Chosen to avoid surface detail loss and artifacts common in sample-based methods.

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
Uses a min/max octree to accelerate ray traversal by skipping empty regions. Chosen for its hierarchical spatial subdivision and boundary rasterization culling, distinct from direct scalar evaluation.

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
Improves SIMD utilization on GPUs for sparse acceleration structures with high primitive density by dynamically switching between standard warp-level intersection and parallel processing of a single ray across multiple threads. Chosen when standard packetization fails due to lost computational coherence.

### [Ray Tracing](8334-ray-tracing.md)
Classic Whitted-style ray tracing capturing shadows, reflection, and refraction via recursive light paths. Chosen as the general framework for physically based rendering of opaque objects.

### Packet Ray List Merging
An optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list to exploit spatial coherence [S2, unofficial]. Phase I traverses the BONO tree creating lists for each packet of rays; if a group overlaps with the current node's projection, the group's list is processed [S2, unofficial]. The upper bound $k$ on list size is increased compared to single pixels, with best performance achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S2, unofficial].

### Geometry Integrator
A direct illumination integrator for scene geometry using a hardcoded two-sided diffuse BSDF with red reflectance $[0.8, 0.2, 0.2]$ [S3, unofficial]. It performs ray intersection `itx : SurfaceInteraction3f = scene.ray_intersect(ray)`, adjusts the normal to point against the ray direction, loads the BSDF, checks visibility via a shadow ray, and accumulates direct illumination if both primary and secondary rays are valid [S3, unofficial].

## Background
Ray tracing was introduced as a surface rendering technique to determine the first object intersected by a camera ray [S1, unofficial]. The goal is to compute the distance between the camera and the intersection point for each object to find the first intersection [S1, unofficial]. It fails when object geometry is too complex for analytical intersection solutions [S1, unofficial]. The method is foundational for rendering opaque objects and serves as the basis for more complex volume rendering techniques that require precise surface intersection [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S3] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
