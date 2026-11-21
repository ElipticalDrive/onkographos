# Ray-Volume and Bounding Box Intersection

Also called: Ray-Volume Intersection, Ray-Box Intersection, Slab Method, Face Distance Method

## Overview
This family provides the geometric primitives for determining if a viewing ray intersects a volume's spatial bounds and calculating the valid sampling interval $[t_{near}, t_{far}]$ along the ray. It solves the problem of culling rays that miss the volume and establishing the start and end points for volume traversal, ensuring density or color sampling occurs only within the volume's physical bounds [S1, S11, S13, S24, S40, unofficial]. Unlike [Ray-box intersection via OpenGL Geometry Shaders](1768-ray-box-intersection-via-opengl-geometry-shaders.md) which rasterizes the box to derive intersection data, this family performs analytical calculations [S3, unofficial].

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].
**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].
**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Ray-box intersection](776-ray-box-intersection.md)
The Ray-box intersection method computes the parametric entry ($t_{near}$) and exit ($t_{far}$) distances of a ray intersecting an Axis-Aligned Bounding Box (AABB). It sets itself apart from [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md) by explicitly providing the scalar interval $[t_{min}, t_{max}]$ required for subsequent volume sampling or traversal, rather than just a boolean hit test [G].

### [Ray-Volume Intersection (Face Distance Method)](2381-ray-volume-intersection-face-distance-method.md)
The Ray-Volume Intersection (Face Distance Method) is a two-step procedure to determine if a viewing ray intersects the data volume and to find the entry point for volume traversal. It calculates distances from the ray origin to each of the six faces of the volume geometry, divides by the corresponding ray direction component, and selects the smallest valid distance as the nearest impact point.

### [Ray Filtering](2681-ray-filtering.md)
Filters rays based on whether they intersect with the alpha mask or the AABB bounding box, reducing computation for rays that do not pass through the scene.

#### Clipping Cell-Level Slice to Active Rays
- **Definition**: An optimization to reduce cell primitives in a slice by intersecting individual rays with the world-space bounding box of the current u,v slice [S1, unofficial].
- **Step**: Ignore rays that have already successfully hit a cell (inactive rays), even if they intersect the slice bounding box [S1, unofficial].

#### Procedural Ray/Box Intersection
- **Definition**: A method for determining ray entry and exit within the volume bounding box entirely within the fragment shader. The ray is defined by the camera position and the volume entry position [S2, unofficial].
- **Parameter**: Requires an exit criterion to determine when the ray has left the volume [S2, unofficial].
- **Pitfall**: The entire computational load for intersection and marching is placed on the fragment shader, which is a disadvantage compared to rasterization-based setup [S2, unofficial].

## Background
The slab method for AABB intersection is a standard technique in ray tracing, often attributed to Amanatides and Woo [S8, S15, S31, S50, S53, S60, unofficial]. The face distance method appears in early interactive volume rendering literature for determining ray-volume entry points [S1, unofficial]. Procedural ray-box intersection in fragment shaders was explored to simplify rendering pipelines by handling intersection logic entirely on the GPU without rasterization setup [S2, unofficial].

## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
