# Texture Mapping in Volume Graphics
Also called: Photo mapping
## Overview
Texture mapping in volume graphics integrates surface appearance data directly into the volumetric representation during the voxelization stage, rather than applying it as a post-processing step to geometric primitives. This approach allows the texture color to be stored within each voxel in the volume buffer, making the mapping viewpoint-independent and eliminating the need for repeated texture lookups during rendering. Unlike surface-based methods where complexity scales with object geometry, this method embeds texture attributes into the voxel data structure itself, enabling efficient exploration of inner structures.
## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset (voxel grid) and texture data (e.g., orthogonal photographs or scalar fields defining color/opacity).
- Output: A volume buffer where each voxel contains both geometric/opacity data and mapped texture color.

**Data Structures:**
- Volume Buffer: A 3D array where each element (voxel) stores a composite value. The exact bit-packing is not specified in the sources, but it must hold the mapped texture color alongside the voxel's intrinsic properties (such as density or opacity) [S1, book].

**Steps:**
1. **Voxelization:** Convert the continuous object representation into a discrete voxel grid.
2. **Texture Projection (Photo Mapping):** For photo mapping specifically, project six orthogonal photographs of the real object back onto the voxelized object. This establishes the mapping between the external surface appearance and the internal voxel coordinates [S1, book].
3. **Storage:** Store the resulting texture color directly into each voxel in the volume buffer during the voxelization stage. This step binds the texture to the voxel data permanently [S1, book].
4. **Rendering:** Render the volume using standard volume rendering techniques (e.g., ray casting or slicing). Since the texture is already stored in the voxels, no additional texture mapping calculations are required during the rendering pass [S1, book].

**Formulas:**
Not given in the sources. The opacity field example mentions $1-r$ where $r$ is the radius, but this defines geometry, not the texture mapping algorithm itself [S1, book].

**Parameters:**
- Number of orthogonal photographs: Six [S1, book].

**Edge Cases and Pitfalls:**
- The method assumes the texture can be accurately projected onto the voxelized geometry.
- Since texture is stored per-voxel, memory usage increases compared to storing only scalar fields, but rendering performance is not degraded by the texture complexity [S1, book].

## Subtopics

### [Four-level Volume-Loading Pipeline with Min-Max Trimming](23741-four-level-volume-loading-pipeline-with-min-max-trimming.md)
This method accelerates volume rendering on hardware with limited texture memory by preprocessing large volume datasets to exclude empty voxels before they enter the rendering pipeline.

## Operating characteristics
**Time and Memory Complexity:**
- Preprocessing (Voxelization + Mapping): Complexity is proportional to the voxelization process. The sources state that texture mapping complexity is proportional to object complexity in surface rendering, but in volume graphics, it is performed once during voxelization [S1, book].
- Rendering: Rendering performance is not degraded by the texture mapping because the mapping is pre-computed and stored. The texture lookup is effectively a memory read from the volume buffer [S1, book].

**Strengths and Limitations:**
- Strengths: Viewpoint independence; texture need not be repeated during rendering; efficient for applications like voxel-based flight simulators and CAD systems where the object geometry and texture are static [S1, book].
- Limitations: Requires storing texture data within the volume buffer, increasing memory footprint. Not suitable for dynamic textures or real-time texture changes without re-voxelization.

**When to Use:**
- Use when the object's texture is static and the inner structures need to be explored with consistent appearance.
- Use in voxel-based flight simulators and CAD systems [S1, book].

## Background
**Origin:**
The method is described in the context of Volume Visualization and Volume Graphics, highlighting the distinction between surface and volumetric representation. It is attributed to the general field of volume graphics research [S1, book].

**History and Development:**
The technique is presented as a fundamental difference between surface and volume rendering pipelines. It is noted that volumetric representation can reveal inner structures, which surface representation cannot, and texture mapping is adapted to fit this volumetric nature [S1, book].

**Applications:**
- Voxel-based flight simulators [S1, book].
- CAD systems [S1, book].

## Sources
[S1] Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
