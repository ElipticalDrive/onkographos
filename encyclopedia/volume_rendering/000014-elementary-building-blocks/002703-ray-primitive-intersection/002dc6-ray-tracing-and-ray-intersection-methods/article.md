# Ray Tracing and Ray Intersection Methods

Also called: Ray Casting, Ray Tracing (broad sense), Ray-Primitive Intersection

## Overview
Ray tracing is a rendering model that determines the color of a pixel by computing the first intersection between a camera ray and scene geometry, handling shadows, reflections, and refractions via secondary rays [S1, unofficial][S2, unofficial]. It differs from simple ray casting by the inclusion of these secondary rays for global illumination effects [S2, unofficial]. This family encompasses the specific algorithms used to compute these intersections for various primitives (boxes, triangles, spheres, implicit surfaces) and acceleration structures.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values. It solves the problem of finding the precise intersection of a ray with a volume's isovalue surface without requiring explicit geometric primitives, serving as a high-accuracy alternative to marching cubes. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) which uses linear interpolation, this method uses recursive bisection to converge on the root, and unlike [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md) which may use analytical derivatives, it relies solely on sampled scalar values.

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Computes the exact intersection between a ray and the isosurface defined by a threshold value within a single voxel of a scalar grid. It solves the cubic polynomial derived from the trilinear interpolation of the eight voxel corner values to find the precise intersection distance $t$. Unlike iterative ray-marching methods, which require multiple sampling steps and suffer from precision issues near the surface, this method provides a direct closed-form or iterative root solution, offering superior performance for silhouette and secondary rays. It builds on [Ray-primitive intersection](769-ray-primitive-intersection.md) by replacing linear bounding-box tests with an exact algebraic solution for the implicit surface within the primitive.

### [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md)
A rendering technique that combines explicit voxel selection with implicit ray-surface intersection to produce high-quality, crack-free isosurfaces from rectilinear volumes. It solves the problem of surface detail loss and artifacts common in fixed-step sample-based methods by restricting ray traversal to active voxels and performing exact implicit intersections within them. Unlike sibling methods such as [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) which may extract polygons, or [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) which operates on uniform grids, this method builds a Bounding Volume Hierarchy (BVH) over logical voxels (such as octants in AMR data) and uses an implicit intersection routine, allowing for efficient rendering of adaptive mesh refinement (AMR) data.

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
A single-kernel iso-surface ray casting method that uses a min/max octree built from macro-cells to accelerate ray traversal. It sets itself apart from its siblings by projecting macro-cell boundaries onto the image plane to create a hexagonal footprint, allowing local rays to skip redundant octree traversal steps. Unlike [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) or [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md), it relies on spatial subdivision rather than iterative root-finding for the initial traversal, though it may use Newton-Raphson for precise intersection within the cell.

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
A strategy to improve SIMD utilization by distributing work for a single ray to many threads when rays have high workload or when standard packetization is inefficient. It solves the problem of low occupancy in warp-level ray-triangle intersection on sparse grids with many primitives per leaf. It sets itself apart from standard packetization by dynamically switching between warp-level parallelism and intra-ray parallelism based on workload thresholds.

### [Ray Tracing](8334-ray-tracing.md)
Ray tracing is a rendering model that captures shadows, reflection, refraction, and diffuse surface shading by placing an image plane in the scene and sending rays through each pixel. It differs from simple ray casting by the additional tracing of secondary rays for shadows and refractions.

### Packet Ray List Merging
An optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list. This exploits spatial coherence where adjacent pixels share common visible blocks, reducing the number of lists generated and processed [S2, unofficial]. Phase I traverses the BONO tree as in the basic scheme but creates lists for each packet of rays (2x2 adjacent pixels) instead of individual pixels. If a group of pixels overlaps with the projection of the current node, the group's list is processed. The upper bound $k$ on the list size for grouped pixels is increased compared to single pixels; best performance is achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S2, unofficial].

### Geometry Integrator
A direct illumination integrator for scene geometry that uses a hardcoded two-sided diffuse BSDF with red reflectance [0.8, 0.2, 0.2] [S3, unofficial]. It performs ray intersection against the scene (`itx : SurfaceInteraction3f = scene.ray_intersect(ray)`), adjusts the normal direction to point against the ray direction, loads the BSDF, and checks visibility using a shadow ray intersected against the scene. Direct illumination is accumulated if both primary and secondary rays are valid [S3, unofficial].

## Background
Ray tracing was introduced as a surface rendering technique to determine the first object intersected by a camera ray by analytically computing intersection distances for simple geometries [S1, unofficial]. It is one of the most efficient techniques for rendering opaque objects when geometry is simple enough for analytical solutions [S1, unofficial]. When geometry is too complex for analytical intersection, ray-marching or other approximation methods are required [S1, unofficial]. The technique fails when object geometry is too complex for analytical intersection solutions [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S3] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
