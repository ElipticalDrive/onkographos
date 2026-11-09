# Ray Traversal and Coherency

Also called: First Depth Ray Traversal

## Overview
This family accelerates isosurface extraction by optimizing the order and grouping of ray-volume interactions to minimize redundant traversal and intersection tests. It differs from naive uniform marching by using hierarchical acceleration structures (octrees, tiled grids) and spatial coherence (ray packets) to skip empty space and process multiple rays simultaneously.

## Shared principles
The family shares the core mechanism of [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md): constructing a ray per pixel and finding the first point where $f(\mathbf{p}) \ge \tau$ or a sign change occurs [S3, unofficial][S1, unofficial]. Upon detection, the intersection is refined (e.g., linear interpolation) and shaded using $\nabla f$ [S3, unofficial][S1, unofficial]. The key distinction is the **traversal strategy**: instead of marching voxel-by-voxel, rays are accelerated by:
1.  **Hierarchical Culling:** Using min-max bounds in octrees or tiled grids to skip entire sub-volumes where $\tau \notin [min, max]$ [S1, unofficial][S2].
2.  **Coherence Exploitation:** Grouping rays into packets (e.g., 2x2) to traverse the acceleration structure together, leveraging spatial locality [S1, unofficial].
3.  **List-Based Restriction:** Pre-computing a short, front-to-back list of candidate blocks/pixels for each ray, restricting the search space [S1, unofficial].

## Subtopics

### [Cell-Based First-Hit Ray Casting](1039-cell-based-first-hit-ray-casting.md)
A technique using a min-max octree of macro-cells ($n^3$ cells, $n \in [3,10]$). It projects trimmed macro-cells to the screen, rasterizes them, and casts local rays only at uncolored pixels. It reduces ray path lengths via early termination but suffers at low thresholds where "peep holes" delay termination.

### [Hybrid Strategy for Interactive High Resolution Isosurface Ray Tracing](3697-hybrid-strategy-for-interactive-high-resolution-isosurface-r.md)
Combines object-order projection with image-order ray casting on multi-core CPUs. Phase I identifies visible, isosurface-intersecting 3D tiles; Phase II dynamically allocates ray casting tasks to cores based on these candidate lists. It solves load-balancing issues of static partitioning but requires careful memory hierarchy management.

### [Interactive High Resolution Isosurface Ray Tracing](4093-interactive-high-resolution-isosurface-ray-tracing.md)
Uses a Branch-On-Need Octree (BONO) to build front-to-back block lists per pixel, capped at size $k$ (typically $\le 20$). Phase I generates lists; Phase II casts rays through them. If the list fills, a resume point is stored for later traversal. It avoids SIMD dependency by using algorithmic list merging, maintaining coherence on complex data.

### [Coherent Multiresolution Isosurface Ray Tracing](4095-coherent-multiresolution-isosurface-ray-tracing.md)
Groups rays into 2x2 packets to traverse a multiresolution octree. It uses an LOD cut depth to switch between fine voxels and coarser cells, maintaining coherence even in large volumes. SSE optimizations accelerate intersection tests within leaf nodes.

### First Depth Ray Traversal
A ray traversal scheme that extracts iso-surfaces, introduced by Tuy & Tuy in 1984 [S1, unofficial].

## Background
The "First Depth" or "First Intensity" traversal strategy was introduced by Tuy and Tuy in 1984 to extract isosurfaces [S1, unofficial]. This approach contrasts with accumulation-based methods (like Levoy’s 1988 compositing) by stopping at the first surface hit [S1, unofficial]. Modern implementations leverage octrees and ray coherence to make this interactive on large datasets [S1, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
