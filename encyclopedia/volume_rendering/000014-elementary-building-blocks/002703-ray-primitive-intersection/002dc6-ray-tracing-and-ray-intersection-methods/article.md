# Ray Tracing and Ray Intersection Methods

Also called: Ray Casting, Ray-Surface Intersection

## Overview
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and calculating the color of each pixel based on the objects the rays intersect [S1, S2, unofficial]. It solves the problem of accurately simulating global illumination effects such as shadows, reflections, and refractions, which simpler rasterization or basic ray casting methods cannot natively support [S1, S2, unofficial]. It is distinguished from its siblings (e.g., [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)) by its explicit recursive handling of secondary rays for specular interactions rather than implicit surface extraction or fixed-step traversal [S1, S2, unofficial].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Binary Search IsoSurface Extraction is a ray-primitive intersection method that locates the intersection of a ray with an isosurface by iteratively halving the interval between two sample points where the scalar field values bracket the target isovalue. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) which solves for the exact intersection using linear interpolation, or [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md) which uses analytical root finding, this method relies on a fixed-depth recursive bisection. It builds on [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md) by providing a robust, grid-independent way to refine surface positions, often serving as a precursor to shading passes that utilize curvature data.

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Trilinear Isosurface Intersection computes the exact intersection between a ray and the isosurface defined by a threshold value within a single hexahedral voxel of a scalar grid. It solves the cubic polynomial derived from the trilinear interpolation of the eight voxel corner values to find the precise intersection distance $t$. Unlike iterative ray-marching methods, which require multiple sampling steps and suffer from precision issues near the surface, this method provides a direct closed-form or iterative root solution, offering superior performance for silhouette and secondary rays. It builds on [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md) by providing the specific algebraic formulation for trilinear fields.

### [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md)
Hybrid Implicit Isosurface Ray Tracing is a rendering technique that combines explicit voxel selection with implicit ray-surface intersection to render high-quality, crack-free isosurfaces from rectilinear volumes. It solves the problem of surface detail loss and artifacts common in sample-based methods by avoiding polygon extraction entirely; instead, it builds a Bounding Volume Hierarchy (BVH) over active voxels and performs implicit intersections within them. Unlike sibling methods such as [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) which rely on iterative root-finding within primitives, or [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) which may extract geometry, this method uses a pre-defined implicit intersection routine (based on Marmitt et al.) to determine the hit point directly within the voxel's trilinear field.

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
Neubauer Iso-surface Ray Casting is a single-kernel iso-surface ray casting method that uses a min/max octree built from macro-cells to accelerate ray traversal. It sets itself apart from sibling methods like Binary Search IsoSurface Extraction by using spatial subdivision and projection-based rasterization to skip empty space, rather than relying solely on iterative root-finding within dense grids.

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
The Hybrid Packetization Strategy is a ray-triangle intersection method designed to improve SIMD utilization on GPUs when standard packetization is inefficient due to sparse acceleration structures with high primitive density per leaf. It addresses the problem of lost coherence in warp-level computations, where some rays traverse full cells while others pass through empty ones, by dynamically distributing the workload of a single ray across multiple threads. Unlike standard packetization, which processes multiple rays in parallel, this method exploits the property that two-level grids contain many primitives per leaf by splitting a single high-workload ray's intersection tests among threads, thereby maintaining parallel efficiency.

### [Ray Tracing](8334-ray-tracing.md)
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and calculating the color of each pixel based on the objects the rays intersect. It solves the problem of accurately simulating global illumination effects such as shadows, reflections, and refractions, which simpler rasterization or basic ray casting methods cannot natively support. It is distinguished from its siblings (e.g., [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)) by its explicit recursive handling of secondary rays for specular interactions rather than implicit surface extraction or fixed-step traversal.

### Packet Ray List Merging
An optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list. This exploits spatial coherence where adjacent pixels share common visible blocks, reducing the number of lists generated and processed [S2, unofficial]. Phase I traverses the BONO tree as in the basic scheme but creates lists for each packet of rays (2x2 adjacent pixels) instead of individual pixels. If a group of pixels overlaps with the projection of the current node, the group's list is processed. The upper bound $k$ on the list size for grouped pixels is increased compared to single pixels; best performance is achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S2, unofficial].

### Geometry Integrator
A direct illumination integrator for scene geometry that uses a hardcoded two-sided diffuse BSDF with red reflectance $[0.8, 0.2, 0.2]$ [S3, unofficial]. It performs ray intersection against the scene (`itx : SurfaceInteraction3f = scene.ray_intersect(ray)`), adjusts the normal direction to point against the ray direction (`itx.n = dr.select(dr.dot(itx.n, ray.d) < 0, itx.n, -itx.n)`), and loads a hardcoded two-sided diffuse BSDF. It checks visibility using a shadow ray intersected against the scene (`itx_shadow : mi.SurfaceInteraction3f = scene.ray_intersect(rayL)`) and accumulates direct illumination if both primary and secondary rays are valid (`L += dr.select(valid_ray, Li, black)`) [S3, unofficial].

## Background
Ray tracing was introduced as a surface rendering technique that determines the first object intersected by a camera ray by analytically computing intersection distances for simple geometries [S1, unofficial]. The goal is always to find which object the ray will intersect first, requiring the computation of the distance between the camera and the intersection point with the ray of each object [S1, unofficial]. Ray-casting is one of the most efficient techniques when object geometry is simple enough for analytical intersection points [S1, unofficial]. When geometry is not simple, ray-marching can be used, marching along the ray by a fixed step size to check if points are inside or outside objects [S1, unofficial]. Sphere-marching is an optimized cousin of ray-marching that works when Signed Distance Functions (SDFs) can be defined [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S3] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
