# RTX BVH Ray Tracing
Also called: Hardware-accelerated ray marching
## Overview
RTX BVH Ray Tracing is a rendering method for Adaptive Block-Refined (ABR) data that uses hardware-accelerated RTX BVH traversal to iterate through regions, performing volume integration within each region and skipping empty space [S1]. It sets itself apart from sibling methods like [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md) by leveraging dedicated hardware ray-tracing cores (RT Cores) for acceleration structure traversal, enabling interactive transfer function and iso-surface editing [S1].

## How it works
**Inputs:**
*   A set of regions derived from Adaptive Block-Refined (ABR) data.
*   A mapping filter storing, for each region, which bricks can influence that region [S1].
*   Ray origin and direction (typically per-pixel).
*   Transfer function and isosurface definitions [S1].

**Outputs:**
*   Computed color and opacity for each ray, resulting in a rendered image [S1].

**Data Structures:**
*   **RTX BVH:** A Bounding Volume Hierarchy built over the regions. The exact internal layout (node size, bit-packing) is determined by the hardware acceleration structure format, but logically it consists of nodes containing bounding volumes and child indices [S1].
*   **Region Filter:** A data structure storing, for each region, a list of influencing bricks [S1].

**Steps:**
1.  **Build BVH:** Construct an RTX BVH over the resulting regions [S1].
2.  **Trace Ray:** For each ray, traverse the RTX BVH using hardware acceleration [S1].
3.  **Skip Space:** Use the BVH traversal to skip empty space (regions with no contributing data) [S1].
4.  **Sample and Integrate:** For regions hit by the ray, perform volume integration. Use the region filter to identify which bricks influence the current region, allowing for fast basis reconstruction without costly cell location kernels [S1].
5.  **Shade:** Apply gradient-based volume shading and surface shading from implicit isosurfaces and/or polygonal surfaces defined throughout the volume [S1].
6.  **Adaptive Sampling:** Adjust sampling frequency based on the finest level cell size to keep total sample counts tractable while maintaining quality [S1].

**Formulas:**
*   Not given in the sources.

**Parameters:**
*   Build quality: Not specified for RTX BVH in the sources.
*   Coherent flag: Not specified for RTX BVH in the sources.

**Edge Cases and Pitfalls:**
*   Precision: Not given in the sources.
*   Boundaries: Not given in the sources.
*   Degenerate input: Not given in the sources.

## Subtopics

### Embree Mesh Acceleration Structure
A wrapper around Embree (RTC) to build a triangle mesh acceleration structure and perform ray tracing (closest hit and any hit) [S2, unofficial]. It is used when software-based ray tracing is required or for comparison, rather than hardware RTX acceleration. It is chosen when hardware RTX is unavailable or when specific software ray tracing features are needed. It differs from RTX BVH by running on the CPU/GPU via Embree rather than dedicated RT Cores [S2, unofficial].

**Steps:**
1.  Initialize Embree device and scene. Set build quality to HIGH [S2, unofficial].
2.  Create triangle geometry [S2, unofficial].
3.  Allocate vertex buffer (`RTC_FORMAT_FLOAT3`) and index buffer (`RTC_FORMAT_UINT3`) [S2, unofficial].
4.  Copy data to buffers [S2, unofficial].
5.  Commit geometry and attach to scene [S2, unofficial].
6.  Commit scene [S2, unofficial].
7.  Trace closest hit for single ray: Use `rtcIntersect1` with coherent context [S2, unofficial].
8.  Trace any hit (occlusion) for single ray: Use `rtcOccluded1` with coherent context [S2, unofficial].
9.  Trace closest hit for multiple rays: Use `rtcIntersect1M` with coherent context flag determined by parameter [S2, unofficial].
10. Trace any hit for multiple rays: Use `rtcOccluded1M` with coherent context flag [S2, unofficial].

**Parameters:**
*   `coherent`: A boolean flag passed to trace functions. If true, uses `RTC_INTERSECT_CONTEXT_FLAG_COHERENT`. If false, uses the same flag in this implementation (potential bug in source, but documented as is) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Hardware-accelerated traversal is generally $O(\log N)$ for BVH traversal, where $N$ is the number of regions, but the constant factor is significantly reduced by hardware support [S1].
*   **Memory Complexity:** Depends on the size of the BVH and the region filter. Not given in the sources.
*   **Measured Performance:** Supports interactive high-quality direct volume rendering [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Hardware acceleration for fast traversal, space skipping, adaptive sampling, and support for interactive editing [S1].
*   **Limitations:** Requires hardware with RTX support. Not given in the sources.
*   **When to use:** When interactive performance is required for ABR data and hardware RTX is available [S1].
*   **When not to use:** When hardware RTX is unavailable or when software-only solutions are preferred [S2, unofficial].

## Background
Originated in the ExaBricks rendering method, presented by Wald et al. in 2020 [S1]. It is used for interactive high-quality direct volume rendering of structured AMR data [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
[S2] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
