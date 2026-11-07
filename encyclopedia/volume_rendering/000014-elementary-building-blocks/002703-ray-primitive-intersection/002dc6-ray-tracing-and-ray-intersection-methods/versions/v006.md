# Ray Tracing and Ray Intersection Methods

Also called: Ray Casting, Ray-Primitive Intersection, Ray Tracing (General)

## Overview
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and computing their interactions with scene geometry to determine color, shadows, reflections, and refractions [S1, unofficial]. It is distinguished from simple ray casting by its native support for secondary rays, specifically those required for shadow determination and specular reflections/refractions, thereby capturing global illumination effects beyond direct lighting [S2, unofficial]. The core problem it solves is finding the closest intersection between a ray and scene primitives, which serves as the foundation for shading and light transport calculations.

## Shared principles
This family builds on the geometric definition of rays and the parallel execution model defined in [Ray-primitive intersection](769-ray-primitive-intersection.md).

## Subtopics

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values. It solves the problem of locating the precise intersection of a ray with a specific density threshold (isovalue) within a volumetric dataset. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md), which analytically solves for the intersection with linearly interpolated planes, this method uses a numerical bisection approach. It differs from [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md) by relying on discrete texture sampling rather than implicit function evaluation or hierarchical acceleration structures. See [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md).

### [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md)
Trilinear Isosurface Intersection detects if a ray crosses an isosurface defined by a threshold value within a hexahedral voxel of a scalar grid. It solves the problem of finding the exact parametric distance $t$ where the trilinearly interpolated scalar field equals the isovalue, converting the intersection test into a cubic polynomial root-finding problem. Unlike [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) which relies on iterative stepping, or [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md) which may use approximations, this method provides a direct algebraic or iterative solution for the root within a single cell, offering superior performance for silhouette and secondary rays by avoiding iterative step reduction near the surface. See [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md).

### [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md)
Hybrid Implicit Isosurface Ray Tracing is a rendering technique that combines explicit voxel selection with implicit ray-primitive intersection to render high-quality, crack-free isosurfaces from rectilinear volumes. It solves the problem of surface detail loss and artifacts inherent in fixed-step sample-based isosurface rendering, particularly in regions with high spatial refinement. Unlike sibling methods such as [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) which rely on iterative root-finding within cells, or [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) which computes exact intersection points analytically, this method builds a Bounding Volume Hierarchy (BVH) over active voxels and performs implicit intersections within each voxel without generating polygonal geometry. See [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md).

### [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md)
Neubauer Iso-surface Ray Casting is a single-kernel iso-surface ray casting method that accelerates traversal by projecting macro-cell boundaries onto the image plane to create hexagonal footprints, thereby reducing redundant octree traversal steps for pixels covered by the same spatial region. It distinguishes itself from sibling methods like Binary Search IsoSurface Extraction by using a hierarchical spatial index (min/max octree) combined with rasterization-based culling rather than pure geometric root-finding or packetization. See [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md).

### [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md)
A strategy to improve SIMD utilization by distributing work for a single ray to many threads when rays have high workload or when standard packetization is inefficient. It addresses the loss of parallel efficiency in sparse grids with many primitives per leaf, where standard warp-level packetization fails due to divergent traversal paths. It sets itself apart by dynamically switching between standard warp-level intersection and a shared-memory-based parallel processing mode based on workload thresholds. See [Hybrid Packetization Strategy](7606-hybrid-packetization-strategy.md).

### [Ray Tracing](8334-ray-tracing.md)
Ray tracing is a rendering technique that forms an image by tracing the path of light rays through an image plane and computing their interactions with scene geometry to determine color, shadows, reflections, and refractions. It is distinguished from simple ray casting by its native support for secondary rays, specifically those required for shadow determination and specular reflections/refractions, thereby capturing global illumination effects beyond direct lighting. See [Ray Tracing](8334-ray-tracing.md).

### Packet Ray List Merging
An optimization where lists of adjacent pixels (typically 2x2 groups) are combined into a single list. This exploits spatial coherence where adjacent pixels share common visible blocks, reducing the number of lists generated and processed [S2, unofficial]. Phase I traverses the BONO tree as in the basic scheme but creates lists for each packet of rays (2x2 adjacent pixels) instead of individual pixels. If a group of pixels overlaps with the projection of the current node, the group's list is processed. The upper bound $k$ on the list size for grouped pixels is increased compared to single pixels; best performance is achieved when $k$ is around 20 for packets, compared to 12 for single pixels [S2, unofficial].

### Geometry Integrator
A direct illumination integrator for scene geometry that uses a hardcoded two-sided diffuse BSDF with red reflectance [0.8, 0.2, 0.2] [S3, unofficial]. It performs ray intersection against the scene (`itx : SurfaceInteraction3f = scene.ray_intersect(ray)`), adjusts the normal direction to point against the ray direction, loads the BSDF, checks visibility using a shadow ray intersected against the scene, and accumulates direct illumination if both primary and secondary rays are valid [S3, unofficial].

## Background
Ray tracing originated as a method for surface rendering, where the goal is to find which object the ray will intersect first [S1, unofficial]. Early techniques distinguished between ray-casting, which computes intersection points analytically for simple geometries, and ray-marching, which is used when geometry is too complex for analytical solutions but point-in-object tests are possible [S1, unofficial]. Sphere-marching was introduced as an optimized cousin of ray-marching for objects definable by Signed Distance Functions (SDFs) [S1, unofficial]. Modern implementations integrate these intersection tests into broader frameworks (like OSPRay or Embree) to handle complex volumes and implicit surfaces [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive High Resolution Isosurface Ray Tracing on Multi-Core ... - UMD, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
[S3] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
