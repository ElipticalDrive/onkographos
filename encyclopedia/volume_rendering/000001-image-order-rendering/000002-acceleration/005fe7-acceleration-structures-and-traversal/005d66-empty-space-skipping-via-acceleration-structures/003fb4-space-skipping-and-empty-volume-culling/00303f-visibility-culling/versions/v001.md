# Visibility Culling

## Overview
Visibility culling in volume rendering identifies and skips the processing of data regions or rays that do not contribute to the final image, thereby reducing computational load and memory bandwidth usage. It differs from its siblings by focusing on the geometric or scalar-based determination of "visible" vs. "invisible" primitives (cells, voxels, or rays) before expensive integration or rasterization occurs, whereas methods like [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) operate during the ray-marching process itself.
## Shared principles
The core mechanism of visibility culling is the evaluation of a binary or probabilistic condition to decide whether a data unit (e.g., a voxel, cell, or ray segment) intersects with the view frustum and the transfer function's opacity range. If the condition evaluates to false, the unit is discarded without performing the volume rendering integral. Common mathematical constructs include bounding volume hierarchies (AABBs) for spatial culling and lookup tables mapping scalar value ranges to opacity existence for attribute culling.
## Subtopics

### [Probabilistic Culling](2833-probabilistic-culling.md)
See [Probabilistic Culling](2833-probabilistic-culling.md).

### [Visibility Feedback with Screen Space Subdivision](3439-visibility-feedback-with-screen-space-subdivision.md)
See [Visibility Feedback with Screen Space Subdivision](3439-visibility-feedback-with-screen-space-subdivision.md).

### Screen Regions
#### definition
An optimization that divides image space into equal-sized squares of $n \times n$ pixels to reduce the overhead associated with per-pixel visibility testing [S1, unofficial]. During rasterization, when a screen region boundary is crossed, the system checks if the entered region is already fully covered by previously rendered pixels; if so, rasterization skips to the next boundary [S1, unofficial].
#### number
Screen regions save approximately 3 percent of overall rendering time on average [S1, unofficial]. Additionally, terminating the rendering process as soon as all pixels are covered (early end of computation) saves another 5 percent of overall rendering time on average [S1, unofficial].

### Visibility Culling via Scalar Range Lookup Table
#### definition
A GPU-based visibility test that uses a $2 \times D$ Boolean lookup table to cull cells that do not intersect the transfer function's opacity range, thereby reducing the load on the rendering stage [S2].
#### step
1. The geometry shader computes the minimum and maximum scalar values among the 8 corner values of a cell [S2].
2. These two values are used as indices to fetch a visibility value from the lookup table [S2].
3. If the cell is deemed visible, the geometry shader instantiates its 6 faces; otherwise, the cell is discarded [S2].
4. Back faces are culled by the standard OpenGL pipeline, and front faces are rasterized [S2].
#### parameter
**Visibility Lookup Table**: A $2 \times D$ table where $D$ is the number of scalar value bins. For each pair of values $s_1$ and $s_2$, the table holds 1 if there exists at least one opacity value which is non-zero for scalar values in the range $[s_1, s_2]$, otherwise it holds 0 [S2].
#### number
On a PC with two Xeon E5345 processors, 4GB memory, and a GeForce 8800 GTX (768MB), using the 'meteorite' dataset (2,377,878 cells):
- Removing the look-up table drops performance to 0.76 fps [S2].
- Adding the look-up table increases the framerate to ~3.48 fps [S2].
- Adding the hybrid CPU-GPU mesh traversal mechanism to the look-up table optimization increases the framerate to 3.62 fps [S2].

### Ray Filtering
#### definition
A technique that filters rays based on whether they intersect with an alpha mask or an Axis-Aligned Bounding Box (AABB), reducing computation for rays that do not pass through the scene [S3, unofficial].
#### step
1. Split rays into chunks for processing [S3, unofficial].
2. For each chunk, if `bbox_only` is False, sample points along the rays and check if any point has alpha > 0 in the alpha mask [S3, unofficial].
3. If `bbox_only` is True, filter based on AABB intersection only [S3, unofficial].
#### parameter
**bbox_only**: A boolean flag. If True, filtering is based on AABB intersection. If False, filtering is based on alpha mask density [S3, unofficial].

## Background
Visibility culling techniques have been developed to address the high computational cost of volume rendering, particularly for large datasets and real-time applications. The concept of using screen regions to optimize rasterization overhead was introduced in the context of cell-based first-hit ray casting [S1, unofficial]. The use of scalar range lookup tables for GPU-based culling was proposed for high-quality, semi-analytical volume rendering of Adaptive Mesh Refinement (AMR) data [S2]. Ray filtering based on alpha masks and bounding boxes is utilized in modern neural rendering pipelines, such as collaborative neural radiance fields [S3, unofficial].
## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
