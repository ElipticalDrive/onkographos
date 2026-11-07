# Visibility Culling

## Overview
Visibility culling in volume rendering identifies and skips the processing of data regions or rays that do not contribute to the final image, thereby reducing computational load and memory bandwidth usage [G]. It differs from its siblings by focusing on the geometric or scalar-based determination of "visible" vs. "invisible" primitives (cells, voxels, or rays) before expensive integration or rasterization occurs, whereas methods like [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) operate during the ray-marching process itself [G].
## Shared principles
(empty)
## Subtopics

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
See [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md).

### [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md)
See [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md).

### [Ray Filtering](2681-ray-filtering.md)
See [Ray Filtering](2681-ray-filtering.md).

### [Probabilistic Culling](2833-probabilistic-culling.md)
See [Probabilistic Culling](2833-probabilistic-culling.md).

### [Adaptive Multi-Step Empty Voxel Skipping](6289-adaptive-multi-step-empty-voxel-skipping.md)
See [Adaptive Multi-Step Empty Voxel Skipping](6289-adaptive-multi-step-empty-voxel-skipping.md).

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

### 24-Region View-Dependent Visibility Preprocessing
#### definition
A preprocessing technique that decomposes the domain of viewing directions into 24 regions on a directional cube to extract a lower number of potentially visible voxels compared to the standard 6-region decomposition, improving extraction rate and rendering speed [S3, unofficial].
#### step
1. Decompose the viewing direction domain into 24 regions based on the signs of the $x$ and $y$ components, assuming the principal component of the viewing direction is the $z$ coordinate [S3, unofficial].
2. Perform preprocessing for all 24 regions. During rendering, select the appropriate precomputed data structure based on the current viewing direction [S3, unofficial].
#### formula
The visibility function $v(i, j, k)$ for the case where $x$ and $y$ components are positive is defined as: $v(i, j, k) = 1$ if $k = 0$ or $\exists l, m \text{ s.t. } i - 1 \leq l \leq i, j - 1 \leq m \leq j, v(l, m, k - 1) = 1, b(l, m, k - 1) = 0$; otherwise 0. This checks only four voxels that could hide the current voxel [S3, unofficial].
#### number
On a low resolution dataset, decomposing into 24 regions extracts 1.8% of voxels, compared to 3.4% for 6 regions. On a high resolution volume, it extracts 0.9% compared to 1.8% for 6 regions [S3, unofficial].

### Frustum Culling
#### step
Call the C++/CUDA function `_C.mark_visible` with positions, viewmatrix, projmatrix, near_n, and far_n to determine visibility [S4, unofficial].
#### layout
Return value: A boolean tensor indicating which points are visible within the frustum defined by viewmatrix, projmatrix, near_n, and far_n [S4, unofficial].

## Background
The concept of using screen regions to optimize rasterization overhead was introduced in the context of cell-based first-hit ray casting [S1, unofficial]. The use of scalar range lookup tables for GPU-based culling was proposed for high-quality, semi-analytical volume rendering of Adaptive Mesh Refinement (AMR) data [S2].

## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S3] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S4] brownvc/gftorf, https://github.com/brownvc/gftorf, fetched 2026-10-27 (unofficial)
