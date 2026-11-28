# RTX BVH Ray Tracing
Also called: Hardware-accelerated ray marching

## Overview
RTX BVH Ray Tracing is a volume rendering method that uses a Bounding Volume Hierarchy (BVH) built over data regions to enable hardware-accelerated ray marching, space skipping, and adaptive sampling [S1]. It is distinguished from sibling methods by its direct reliance on RTX hardware acceleration for traversal and shading, which supports interactive transfer function and iso-surface editing alongside adaptive sample density control [S1]. Unlike [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md) which may use software-based logic for region boundaries, or [Sparse Brick Set Rendering](9909-sparse-brick-set-rendering.md) which focuses on sparse data structures, this method explicitly leverages RTX ray tracing cores for efficient space skipping and adaptive sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Adaptive Block-Refined (ABR) data organized into regions; an RTX BVH built over these regions; transfer functions; isosurface definitions [S1].
*   **Output:** Rendered image with direct volume rendering, surface shading from implicit isosurfaces, and/or polygonal surfaces [S1].

**Data Structures:**
*   **Regions:** Spatial partitions of the ABR data. Each region stores a filter indicating which bricks can influence the region, allowing for fast basis reconstruction without costly cell location kernels [S1].
*   **RTX BVH:** A bounding volume hierarchy built over the regions [S1].

**Steps:**
1.  **Build BVH:** Construct an RTX BVH over the resulting regions derived from the ABR data [S1].
2.  **Hardware-Accelerated Traversal:** Use RTX hardware to traverse the BVH [S1]. This step performs space skipping during traversal [S1].
3.  **Adaptive Sampling:** During traversal, adapt the sampling frequency to the finest level cell size while keeping total sample counts tractable [S1].
4.  **Volume Integration:** For intersected regions, perform volume integration using the stored basis reconstruction and transfer functions [S1].
5.  **Shading:** Apply gradient-based volume shading and surface shading from implicit isosurfaces or polygonal surfaces defined throughout the volume [S1].

**Formulas:**
Not given in the sources. [S1]

**Parameters:**
Not given in the sources. [S1]

**Edge Cases and Pitfalls:**
Not given in the sources. [S1]

## Subtopics

### GRTX
GRTX is a concurrent work proposing a hardware extension to accelerate 3DGRT-style repeated BVH traversals [S2]. It differs from standard RTX BVH ray tracing by specifically targeting the acceleration of repeated traversal patterns found in Gaussian Ray Tracing [S2]. It proposes using icosahedron meshes with instancing akin to icosahedrons inside oriented bounding boxes to optimize this process [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Measured Performance:** Not given in the sources. [S1]
*   **Strengths:** Supports interactive high-quality direct volume rendering with smooth AMR reconstruction filters and gradient-based volume shading [S1]. Enables interactive transfer function and iso-surface editing [S1]. Guarantees that sampling frequency adapts to the finest level cell size while keeping total sample counts tractable [S1].
*   **Limitations:** Not given in the sources. [S1]
*   **When to Use:** When interactive rendering of structured AMR data with complex transfer functions and isosurfaces is required, and hardware-accelerated ray tracing is available [S1].
*   **When Not to Use:** Not given in the sources. [S1]

## Background
The method was introduced in the context of ray tracing structured Adaptive Mesh Refinement (AMR) data [S1]. It builds on the ExaBricks framework to provide hardware-accelerated traversal for volume rendering [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
