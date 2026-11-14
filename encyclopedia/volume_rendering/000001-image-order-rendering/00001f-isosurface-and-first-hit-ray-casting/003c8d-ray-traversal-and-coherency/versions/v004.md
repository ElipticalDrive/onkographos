# Ray Traversal and Coherency
## Overview
This family accelerates isosurface extraction by optimizing the order and grouping of ray-volume interactions to minimize redundant traversal and intersection tests [G]. It differs from naive uniform marching by using hierarchical acceleration structures (octrees, tiled grids) and spatial coherence (ray packets) to skip empty space and process multiple rays simultaneously [G].
## Shared principles
The family shares the core mechanism of [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md): constructing a ray per pixel and finding the first point where $f(\mathbf{p}) \ge \tau$ or a sign change occurs [G]. Upon detection, the intersection is refined (e.g., linear interpolation) and shaded using $\nabla f$ [G]. The key distinction is the traversal strategy: instead of marching voxel-by-voxel, rays are accelerated by hierarchical culling using min-max bounds in octrees or tiled grids to skip sub-volumes where $\tau \notin [\min, \max]$ [G], coherence exploitation by grouping rays into packets to traverse the acceleration structure together [G], and list-based restriction by pre-computing a short, front-to-back list of candidate blocks or pixels for each ray [G].
## Subtopics

### [Cell-Based First-Hit Ray Casting](1039-cell-based-first-hit-ray-casting.md)
A technique using a min-max octree of macro-cells ($n^3$ cells, $n \in [3,10]$) to accelerate iso-surface extraction [G]. It sets itself apart by projecting trimmed macro-cells to the screen and casting local rays only at uncolored pixels, avoiding empty space traversal [G]. Choose it when interactive threshold adjustment is needed without dedicated acceleration hardware, and when macro-cell culling effectively skips empty space [G].

### [Hybrid Strategy for Interactive High Resolution Isosurface Ray Tracing](3697-hybrid-strategy-for-interactive-high-resolution-isosurface-r.md)
A two-phase method combining object-order traversal with restricted ray casting on multi-core processors [G]. It sets itself apart by decoupling candidate block identification from ray casting execution, using dynamic task allocation to solve load-balancing and memory hierarchy overhead problems [G]. Choose it when rendering on multi-core CPUs and static screen partitioning causes uneven workloads [G].

### [Interactive High Resolution Isosurface Ray Tracing](4093-interactive-high-resolution-isosurface-ray-tracing.md)
A two-phase algorithm for rendering isosurfaces on multi-core processors using a BONO tree for hierarchical culling and list-based restriction [G]. It sets itself apart by maintaining a short, front-to-back ordered list of candidate blocks per pixel, achieving coherence and efficiency without SIMD instruction-level parallelism dependencies [G]. Choose it when high-resolution rendering is needed on multi-core CPUs without relying on dedicated hardware or surface fitting techniques [G].

### [Coherent Multiresolution Isosurface Ray Tracing](4095-coherent-multiresolution-isosurface-ray-tracing.md)
A method for interactive isosurface extraction from large volumetric datasets stored in octrees [G]. It sets itself apart by grouping neighboring rays into 2x2 packets to exploit spatial coherence during traversal and intersection, dynamically switching to coarser levels of detail (LOD) [G]. Choose it when rendering large volumes on commodity CPUs where spatial coherence can be maintained via ray packets and multiresolution structures [G].

### [First Depth Ray Traversal](8327-first-depth-ray-traversal.md)
A ray traversal method that extracts iso-surfaces by recording the first scalar intensity encountered along a viewing ray [G]. It sets itself apart from sibling methods like Maximum Intensity Projection (MIP) or Accumulate compositing by its immediate termination upon the first hit [G]. Choose it when isolating high-density or specific threshold structures without contribution from deeper volumes is required [G].

## Background
The "First Depth" or "First Intensity" traversal strategy was introduced by Tuy and Tuy in 1984 to extract isosurfaces [G]. This approach contrasts with accumulation-based methods (like Levoy’s 1988 compositing) by stopping at the first surface hit [G]. Modern implementations leverage octrees and ray coherence to make this interactive on large datasets [G].
## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
