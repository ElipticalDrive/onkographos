# Ray Tracing and Ray Intersection Methods

Also called: Ray Casting, Ray-Surface Intersection, Ray Tracing (Whitted-style)

## Overview
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and computing the color of each pixel based on interactions with scene geometry [S1, unofficial]. It solves the problem of accurately simulating global illumination effects such as shadows, specular reflections, and refractions, which are difficult to achieve with rasterization [S1, unofficial]. It is distinguished from simple ray casting by its native support for recursive secondary rays (reflections and refractions) and shadow testing, a model first proposed by Whitted [S2, unofficial].

## Shared principles
This family operates on the geometric definition of rays and the parallel execution model defined in [Ray-primitive intersection](769-ray-primitive-intersection.md). The core mechanism involves computing the intersection distance $t$ between a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ and a primitive. The closest positive $t$ determines the visible surface.

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Binary Search IsoSurface Extraction is a ray-primitive intersection method that approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values. It solves the problem of locating the precise intersection of a ray with a specific isovalue within a volumetric scalar field, offering higher accuracy than discrete marching cubes by refining the hit position through bisection. It distinguishes itself from sibling methods like [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) by using iterative refinement rather than analytical root-finding, and from [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md) by relying on discrete texture sampling rather than implicit function evaluation.

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Trilinear Isosurface Intersection computes the exact point where a ray crosses the zero-level set (isosurface) of a scalar field defined by trilinear interpolation within a single hexahedral voxel. It solves the cubic polynomial derived from substituting the ray equation into the trilinear interpolant to find the intersection distance $t$. Unlike iterative ray-marching methods, which step through the volume and approximate the surface, this method provides a direct analytical or semi-analytical solution, offering superior performance for silhouette and secondary rays where precision is critical. It builds on [Ray Tracing and Ray Intersection Methods](5617-ray-tracing-and-ray-intersection-methods.md) by providing the specific algebraic formulation for hexahedral cells.

### [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md)
Hybrid Implicit Isosurface Ray Tracing is a rendering technique that combines explicit voxel selection with implicit ray-primitive intersection to render high-quality, crack-free isosurfaces from rectilinear volumes. It solves the problem of surface detail loss and artifacts common in sample-based ray casting, particularly in regions with high spatial refinement where fixed step sizes are too large to capture fine features. Unlike pure implicit methods that traverse dense grids or explicit methods that generate polygonal meshes, this method builds a Bounding Volume Hierarchy (BVH) over a sparse list of "active voxels" and performs implicit ray-voxel intersections within those primitives.

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
Neubauer Iso-surface Ray Casting is a single-kernel iso-surface ray casting method that accelerates ray traversal by combining a min/max octree with projection-based rasterization. It sets itself apart from sibling methods like Binary Search IsoSurface Extraction by using spatial subdivision to skip empty regions rather than stepping through voxels, and from Trilinear Isosurface Intersection by rasterizing macro-cell boundaries to determine ray coverage.

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
The Hybrid Packetization Strategy is a ray-triangle intersection method designed to improve SIMD utilization on GPUs when rendering scenes with sparse acceleration structures that contain many primitives per leaf. It addresses the inefficiency of standard packetization, where assigning different rays to different threads causes a loss of coherence if some rays traverse full cells while others pass through empty ones. Unlike standard packetization which processes a fixed group of rays in parallel, this method dynamically distributes the workload of a single ray with high intersection candidates across multiple threads.

### [Ray Tracing](8334-ray-tracing.md)
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and computing the color of each pixel based on interactions with scene geometry. It solves the problem of accurately simulating global illumination effects such as shadows, specular reflections, and refractions, which are difficult to achieve with rasterization. It is distinguished from simple ray casting by its native support for recursive secondary rays (reflections and refractions) and shadow testing, a model first proposed by Whitted.

### Packet Ray List Merging
Packet Ray List Merging is an optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list. This exploits spatial coherence where adjacent pixels share common visible blocks, reducing the number of lists generated and processed [S2, unofficial]. Phase I traverses the BONO tree as in the basic scheme but creates lists for each packet of rays instead of individual pixels. If a group of pixels overlaps with the projection of the current node, the group's list is processed. The upper bound $k$ on the list size is increased compared to single pixels; best performance is achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S2, unofficial].

### Geometry Integrator
The Geometry Integrator is a direct illumination integrator for scene geometry that uses a hardcoded two-sided diffuse BSDF with red reflectance $[0.8, 0.2, 0.2]$ [S3, unofficial]. It performs ray intersection against the scene (`itx : SurfaceInteraction3f = scene.ray_intersect(ray)`), adjusts the normal direction to point against the ray direction, loads the BSDF, and checks visibility using a shadow ray intersected against the scene. Direct illumination is accumulated if both primary and secondary rays are valid [S3, unofficial].

## Background
Ray tracing was introduced as a surface rendering technique to determine the first object intersected by a camera ray by analytically computing intersection distances for simple geometries [S1, unofficial]. The method computes the distance between the camera and the intersection point with the ray of each object to find the first intersection [S1, unofficial]. It fails when object geometry is too complex for analytical intersection solutions, necessitating alternatives like ray-marching or ray-tracing with acceleration structures [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S3] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
