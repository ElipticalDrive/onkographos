# GigaVoxels

Also called: Dynamic Sparse Octree, Ray-Guided Streaming

## Overview
GigaVoxels is a real-time voxel-based rendering pipeline for efficient exploration of large and detailed scenes, using a sparse voxel octree data structure, GPU caching, and ray-guided streaming to load only needed data at the needed resolution [S1, unofficial][S2, unofficial]. It differs from its parent [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md) by introducing a hierarchical mipmap-like structure and a centralized cache manager that streams data on-demand based on ray traversal requests, rather than relying solely on a static traversal stack [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Large volumetric datasets (e.g., 3D scans, hypertextured meshes, fractals) stored on mass storage or in a hierarchical format [S1, unofficial][S2, unofficial].
- **Output:** A rendered image with anti-aliased, filtered volume visualization [S1, unofficial][S2, unofficial].

**Data Structures:**
The system uses a unified data structure combining geometry and texture [S1, unofficial]:
1.  **Sparse Voxel Mipmap Pyramid:** A hierarchy built by iteratively downsampling and averaging neighbors. This allows reading from a mipmap level to deliver the integrated value of a small volume, approximating the cone integral defined by the eye and pixel extent [S2, unofficial].
2.  **Generalized N3-tree (Octree):** Used for space partitioning and empty space compaction. It is a "node pointer based" structure that links bricks of voxels [S1, unofficial].
3.  **Bricks of Voxels:** Constant-sized memory blocks storing opacity, color, and normal. These are linked by octree nodes [S1, unofficial].
4.  **Data Cache:** Implemented as a 3D texture for reading, bound with surfaces for writing data. It uses an LRU (Least Recently Used) mechanism to maintain temporal coherence [S1, unofficial].

**Pipeline Steps:**
1.  **Voxel RayTracer:** Emits data usage requests based on the current view and occlusion information. It performs hierarchical volume ray-casting, cone tracing, and brick marching [S1, unofficial].
2.  **Central Memory/Cache Manager:** Handles structure updates and loads data from mass storage or GPU cache in response to requests. It supports two loading modes:
    -   Level by level (from coarse to fine) for speed trade-off [S1, unofficial].
    -   Directly max level of resolution for quality first [S1, unofficial].
3.  **Renderer:** Performs the actual rendering using the cached data [S1, unofficial].

**Procedural Data Generation (Hypertextures):**
To enrich datasets, procedural methods are used. For noise optimization, the system mimics LOD by reusing noise computation at previous resolution levels. Only one octave of noise is computed per level of resolution and stored in the GPU cache. The sum is reproduced as:
$$ F(\text{level } N) = \text{currentNoise}(\text{level } N) + \text{previousNoise}(\text{level } N-1) $$
This replaces "noise computation" with "data fetch in cache" [S1, unofficial].

**Node Subdivision on GPU:**
A kernel uses 1 block per node and 1 thread per child node.
-   **Input:** Localization info (LOD depth, Children spatial index pos) and address in node cache.
-   **Process:** Test if the child node is inside the object (e.g., analytically for a sphere).
-   **Output:** Write nodes in cache [S1, unofficial].

**Bricks of Voxels Production on GPU:**
A kernel uses 1 block per brick.
-   **Input:** Localization info (LOD depth, spatial index pos) and address in data cache.
-   **Process:** User populates voxels.
-   **Output:** Write voxels in cache [S1, unofficial].

**Ray Casting Shader:**
The custom shader kernel samples data using `brickSampler`. It retrieves color (channel 0) and accumulates color if opacity > 0.
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for ShaderKernel::run
def run(brickSampler, coneAperture, accumulatedColor):
    # Retrieve first channel element: color
    color = brickSampler.getValue(0, coneAperture)
    
    # Accumulate color if opacity is non-zero
    if color.w > 0.0f:
        accumulatedColor = accumulatedColor + (1.0f - accumulatedColor.w) * color
        
    return accumulatedColor
```
The shader provides entry points for customization:
-   `getConeAperture()`: Modify cone aperture [S1, unofficial].
-   `getStopCriterion()`: e.g., `accumulatedColor.w >= 0.98f` to stop ray marching [S1, unofficial].
-   `getDescentCriterion()`: Used during "visit" traversal [S1, unofficial].

**OpenGL Interoperability:**
To mix triangles and voxels, opaque objects are rendered first to the depth buffer to skip occluded parts. GigaVoxels registers textures, render buffers, and PBOs with optimized flags (`cudaGraphicsRegisterFlagsReadOnly`, `cudaGraphicsRegisterFlagsWriteDiscard`) [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Aliasing:** Appears because pixel value should be defined by integration along a cone, not just ray accumulation. Mipmapping addresses this by smoothing results and allowing larger step sizes proportional to distance [S2, unofficial].
-   **Sparse-Twice Data:** Scenes often consist of dense clusters in free space. The system exploits this via compaction, fast traversal of empty space, and avoiding occluded regions [S2, unofficial].

## Subtopics

### [Distributed Multi-Level Out-Of-Core Volume Rendering](6238-distributed-multi-level-out-of-core-volume-rendering.md)
Distributed Multi-Level Out-Of-Core Volume Rendering is a distributed GPU volume rendering system that uses a multi-level hierarchy of caches and data streaming to render volumes larger than GPU memory, maintaining a 1:1 pixel-to-voxel ratio for interactive exploration. It differs from its parent [GigaVoxels](3057-gigavoxels.md) by operating on a distributed-memory cluster with explicit data management across CPU and GPU memory, rather than a single-node ray-guided streaming approach. It also differs from Vol-a-Tile by not replicating data within a region-of-interest on every node, and from Volume Visualizer by distributing data rather than replicating the entire dataset.

### Dense 3D Texture Voxelization
An alternative to GigaVoxels where the entire scene (empty space and all) is encoded into a very large mip-mapped 3D texture [S3, unofficial]. The octree is implicit in the mipmap levels of the 3D texture. Rays traverse the mip-map chain instead of an octree. Rays index into the voxel data directly because the 3D texture is a world space representation of the scene. Quadrilinear interpolation is automatically achieved with `texture()` calls because of the mip map chain. Bricks are spatially aligned in the scene rather than existing in an amorphous memory pool, so redundant neighbor voxels do not need to be stored. The filtering stage (mip map generation) is straightforward when there are no redundant neighbor voxels. However, dense 3D textures are impractical because of how much memory they consume, especially the empty space that must be backed by physical memory [S3, unofficial].

## Operating characteristics
-   **Time and Memory Complexity:** The system achieves real-time to interactive rates on volumes exceeding GPU memory capacities by far, thanks to efficient streaming based on a ray-casting solution. The volume is only used at the resolution needed to produce the final image, providing memory and speed gains [S2, unofficial].
-   **Measured Performance:** 20-90 fps on current generation hardware, respecting limited GPU memory budget [S2, unofficial].
-   **Scaling:** The system allows a trade-off between quality and performance, exploiting temporal coherence [S2, unofficial].
-   **Strengths:** Inherently anti-aliased rendering, smooth transitions to different subdivision levels, and ability to handle semi-transparent objects [S1, unofficial][S2, unofficial].
-   **Limitations:** Mipmapping increases memory consumption and makes tree updates more challenging [S2, unofficial].
-   **When to use:** For exploring large and detailed scenes (e.g., 3D scans, hypertextured meshes, fractals) where real-time interaction is required [S2, unofficial].
-   **When not to use:** When the scene is dense and sparse, or when the GPU memory budget is extremely constrained and streaming overhead is prohibitive [S2, unofficial].

## Background
GigaVoxels was developed by Sylvain Lefebvre (INRIA Sophia-Antipolis) and Elmar Eisemann (MPI Informatik / Saarland University) [S2, unofficial]. The approach was developed in parallel to the work of Gobbetti et al. [GMAG08] [S2, unofficial]. It was presented at the I3D 2009 conference proceedings [S2, unofficial]. The method builds on the structure from [LHN05a] and [LKS*06], combining an adaptive version of the former with regular constant-sized memory blocks of the latter [S2, unofficial].

## Sources
[S1] GigaVoxels, Real-time Voxel-based Library to Render Large and Detailed Objects, https://inria.hal.science/hal-00808121v1, fetched 2026-10-26 (unofficial)
[S2] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S3] http://realtimevoxels.blogspot.fr/2012/09/introduction-to-gigavoxels-and-sparse.html, http://realtimevoxels.blogspot.fr/2012/09/introduction-to-gigavoxels-and-sparse.html, fetched 2026-10-28 (unofficial)
