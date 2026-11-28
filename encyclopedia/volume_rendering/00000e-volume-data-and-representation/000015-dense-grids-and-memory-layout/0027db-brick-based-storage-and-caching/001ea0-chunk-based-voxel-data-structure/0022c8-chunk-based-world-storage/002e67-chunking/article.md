# Chunking
Also called: Voxel Chunking
## Overview
Chunking manages large voxel worlds by dividing them into fixed-size chunks, trading faster meshing for more draw calls [S1, unofficial]. It builds on [Chunk-Based World Storage](2858-chunk-based-world-storage.md) by applying fixed-size partitioning to optimize rendering and memory management.
## How it works
**Inputs and Outputs:** Not given in the sources. [S1, unofficial]
**Data Structures:** Not given in the sources. [S1, unofficial]
**Steps:**
1. Partition the global voxel world into fixed-size, axis-aligned chunks [S1, unofficial].
2. Apply greedy meshing to each chunk by scanning across voxel slices and merging faces of adjacent voxels with the same value [S1, unofficial].
3. Render the resulting chunk meshes, accepting the tradeoff of faster meshing for a higher number of draw calls [S1, unofficial].
**Formulas:** Not given in the sources. [S1, unofficial]
**Parameters:**
- Minecraft-style: $16 \times 16 \times 256$, divided vertically [S1, unofficial].
- Demo engine: $32 \times 32 \times 32$ [S1, unofficial].
**Edge Cases and Pitfalls:**
- The technique inherently trades faster meshing for a higher number of draw calls [S1, unofficial].
## Subtopics

### Chunk-based Rendering (Minecraft Style)
A spatial partitioning strategy specific to voxel engines like Minecraft, using 2D chunks (planes) rather than 3D octrees [S2, unofficial]. It sets itself apart by leveraging simpler and more efficient X-Y plane visibility checks compared to 3D checks, as the horizontal plane typically contains more visible surfaces [S2, unofficial]. It is chosen when the view is primarily horizontal and 2D culling offers performance gains over 3D tree traversal [S2, unofficial].

## Operating characteristics
- **Trade-offs:** Faster meshing at the cost of more draw calls [S1, unofficial].
- **Time Complexity:** Not given in the sources. [S1, unofficial]
- **Memory Complexity:** Not given in the sources. [S1, unofficial]
- **Measured Performance:** Not given in the sources. [S1, unofficial]
- **Strengths:** Not given in the sources. [S1, unofficial]
- **Limitations:** Not given in the sources. [S1, unofficial]
- **When to Use:** Not given in the sources. [S1, unofficial]
- **When Not to Use:** Not given in the sources. [S1, unofficial]
## Background
Not given in the sources. [S1, unofficial]
## Sources
[S1] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S2] OpenGL voxel engine slow, https://stackoverflow.com/questions/4563693/opengl-voxel-engine-slow, fetched 2026-10-26 (unofficial)
