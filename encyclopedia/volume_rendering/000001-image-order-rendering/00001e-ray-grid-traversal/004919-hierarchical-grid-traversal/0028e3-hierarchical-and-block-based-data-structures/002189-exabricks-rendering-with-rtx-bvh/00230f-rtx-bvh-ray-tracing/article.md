# RTX BVH Ray Tracing
Also called: Hardware-accelerated ray marching

## Overview
RTX BVH Ray Tracing is a method for volume rendering that utilizes hardware-accelerated Bounding Volume Hierarchy (BVH) traversal to iterate through data regions, performing volume integration within each region while skipping empty space [S1]. It is distinguished from sibling methods by its reliance on NVIDIA RTX hardware acceleration for both traversal and shading, enabling interactive transfer function and iso-surface editing alongside adaptive sampling [S1]. Unlike [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md) which may use software-based logic for region boundaries, or [Sparse Brick Set Rendering](9909-sparse-brick-set-rendering.md) which focuses on sparse data structures, this method explicitly leverages the RTX ray tracing cores for efficient space skipping and adaptive sample density control [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Adaptive Block-Refined (ABR) data organized into regions; an RTX BVH built over these regions; transfer functions; isosurface definitions [S1].
*   **Output:** Rendered image with direct volume rendering, surface shading from implicit isosurfaces, and/or polygonal surfaces [S1].

**Data Structures:**
*   **Regions:** Spatial partitions of the ABR data. Each region stores a filter indicating which bricks can influence the region, allowing for fast basis reconstruction without costly cell location kernels [S1].
*   **RTX BVH:** A bounding volume hierarchy built over the regions. The BVH nodes contain bounding volumes enclosing the regions [S1].

**Steps:**
1.  **Build BVH:** Construct an RTX BVH over the resulting regions derived from the ABR data [S1].
2.  **Ray Generation:** Generate rays for each pixel in the image grid [S1].
3.  **Hardware-Accelerated Traversal:** Use the RTX hardware to traverse the BVH. This step performs space skipping by ignoring regions that do not intersect the ray or contain no data [S1].
4.  **Adaptive Sampling:** During traversal, adapt the sampling frequency to the finest level cell size while keeping total sample counts tractable [S1].
5.  **Volume Integration:** For intersected regions, perform volume integration using the stored basis reconstruction and transfer functions [S1].
6.  **Shading:** Apply gradient-based volume shading and surface shading from implicit isosurfaces or polygonal surfaces defined throughout the volume [S1].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### GRTX
GRTX is a concurrent work proposing a hardware extension to accelerate 3DGRT-style repeated BVH traversals [S2]. It differs from standard RTX BVH ray tracing by specifically targeting the acceleration of repeated traversal patterns found in Gaussian Ray Tracing [S2]. It proposes using icosahedron meshes with instancing akin to icosahedrons inside oriented bounding boxes to optimize this process [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Supports interactive high-quality direct volume rendering with smooth AMR reconstruction filters and gradient-based volume shading [S1]. Enables interactive transfer function and iso-surface editing [S1]. Guarantees that sampling frequency adapts to the finest level cell size while keeping total sample counts tractable [S1].
*   **Limitations:** Not given in the sources.
*   **When to Use:** When interactive rendering of structured AMR data with complex transfer functions and isosurfaces is required, and hardware-accelerated ray tracing is available [S1].
*   **When Not to Use:** Not given in the sources.

## Background
The method was introduced in the context of ray tracing structured Adaptive Mesh Refinement (AMR) data using ExaBricks [S1]. It builds on the ExaBricks framework to provide hardware-accelerated traversal for volume rendering [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
