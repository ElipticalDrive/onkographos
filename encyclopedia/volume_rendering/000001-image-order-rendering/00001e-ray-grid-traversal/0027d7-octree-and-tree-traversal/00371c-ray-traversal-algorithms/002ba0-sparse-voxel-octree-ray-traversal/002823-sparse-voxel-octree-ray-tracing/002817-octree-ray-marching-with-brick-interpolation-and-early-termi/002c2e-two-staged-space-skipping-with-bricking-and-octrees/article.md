# Two-staged Space-Skipping with Bricking and Octrees

Also called: Bricking with octree refinement, Two-stage space-skipping

## Overview
This method balances GPU bottlenecks by dividing space-skipping into two stages: coarse division using bricking to address bus and texture cache limitations, and fine division using octrees within bricks to address the rasterization bottleneck [S1, unofficial]. It differs from Tong et al.'s bricking technique by storing unmodified scalar values in textures and using a fragment shader for post-interpolative RGB$\alpha$ lookup, avoiding time-consuming texture regeneration when the transfer function changes [S1, unofficial].

## How it works
**Inputs:** 3D scalar volume data, transfer function, and camera/view parameters [G].
**Outputs:** 2D rendered image [G].

**Data Structures:**
1. **Bricks:** 3D texture chunks of size $b \times b \times b$ voxels [S1, unofficial].
   * Content: Unmodified scalar voxel values [S1, unofficial].
   * Overlap: $n$ voxels in every direction to support interpolation [S1, unofficial].
   * Memory Overhead: $\approx (3 \cdot n / b) \cdot 100\%$ [S1, unofficial].
2. **Octrees:** Stored within each brick [S1, unofficial].
   * Structure: Hierarchical subdivision of the brick volume [S1, unofficial].
   * Content: Empty/non-empty status of sub-volumes to enable fine-grained skipping [S1, unofficial].

**Parameters:**
* **Brick size ($b$):** Must be equal to or smaller than the available texture memory size [S1, unofficial].
* **Overlap ($n$):**
  * Nearest neighbor: $n=0$ [S1, unofficial].
  * Trilinear interpolation: $n=1$ voxel in every direction [S1, unofficial].
  * Pre-integrated rendering or on-the-fly gradient calculation: $n$ increased by another voxel in every direction [S1, unofficial].
* **Texture Memory Management:** If volume exceeds texture memory, bricks are loaded sequentially from main memory into texture memory during rendering [S1, unofficial].

**Steps:**
1. **Brick Partitioning:** Divide the volume data set into chunks called bricks [S1, unofficial]. Ensure adjacent bricks overlap by $n$ voxels to satisfy the interpolation kernel requirements [S1, unofficial].
2. **Octree Construction:** Construct an octree within each brick to identify empty regions at a finer granularity than the brick level [S1, unofficial].
3. **Texture Loading:** Load bricks into video memory as 3D textures [S1, unofficial]. If the total data exceeds available texture memory, swap bricks between main memory and texture memory as needed [S1, unofficial]. Note: Many graphics cards require 3D texture sizes to be powers of 2; handle non-divisible dimensions by adding partially empty bricks or using smaller rest-bricks [S1, unofficial].
4. **Ray Traversal (Stage 1 - Coarse):** Traverse the volume using the brick structure [S1, unofficial]. Skip entire bricks that are empty or do not intersect the view frustum [S1, unofficial]. This addresses bus and texture cache bottlenecks [S1, unofficial].
5. **Early Ray Termination (Stage 1):** Apply early ray termination to each brick to alleviate fragment shader load, particularly beneficial for highly-filled data sets [S1, unofficial].
6. **Ray Traversal (Stage 2 - Fine):** Within active bricks, use the octree to skip empty sub-volumes [S1, unofficial]. This addresses the rasterization bottleneck by reducing the number of pixels accessed [S1, unofficial].
7. **Rendering:** For each sample point, lookup the scalar value from the brick texture [S1, unofficial]. Use a fragment shader program to post-interpolatively lookup RGB$\alpha$ values based on the current transfer function [S1, unofficial].

**Pitfalls:**
* **Triangle Throughput Bottleneck:** Space-skipping techniques may increase the amount of triangles considerably [S1, unofficial]. If the triangle count becomes too high, it becomes a limiting factor for frame rate, potentially overloading the triangle throughput bottleneck when the rasterization bottleneck is lifted [S1, unofficial].
* **Bottleneck Balance:** The two stages (bricking and octree) must be balanced because lifting one bottleneck may overload another (e.g., rasterization vs. triangle throughput) [S1, unofficial].
* **Precision/Format Limitations:** Hardware texture compression (S3TC, FXT1, DXT1, VTC) is limited to lossy 8-bit RGB($\alpha$) compression, making it unsuitable for 12- or 16-bit scalar medical data [S1, unofficial].

## Operating characteristics
* **Complexity:** Not given in the sources.
* **Performance:** Addresses bus and texture cache bottlenecks via bricking, and rasterization bottlenecks via octrees [S1, unofficial].
* **Strengths:** Handles data sets exceeding texture memory; allows runtime transfer function changes without texture regeneration; reduces fragment shader load via early ray termination [S1, unofficial].
* **Limitations:** Risk of triangle throughput bottleneck if space-skipping increases triangle count too much; texture compression is unsuitable for 12- or 16-bit scalar medical data [S1, unofficial].
* **When to use:** When rendering large scalar volumes on GPUs with limited texture memory and when transfer function flexibility is required [S1, unofficial].
* **When not to use:** When triangle throughput is the primary bottleneck and space-skipping would significantly increase triangle count [S1, unofficial].

## Background
* **Origin:** Described in a 2006 publication [S1, unofficial].
* **History:** Builds on Eckhardt's bricking technique [Eck98] and Tong et al.'s bricking technique [TWTT99]. The method analyzes bottlenecks in the graphics pipeline (bus, texture cache, rasterization, triangle throughput) to justify the two-stage approach [S1, unofficial].
* **Usage:** Used in GPU volume rendering, particularly for medical data where scalar values are 12- or 16-bit and transfer functions need frequent adjustment [S1, unofficial].

## Sources
[S1] Optimizing GPU Volume Rendering, Danny Ruijters, https://dannyruijters.nl/docs/OptimizingGPUVolumeRendering_final.pdf, fetched 2026-10-26 (unofficial)
