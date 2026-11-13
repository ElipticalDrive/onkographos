# Water Face Mesh Generation
Also called: addWaterFace
## Overview
Generates mesh vertices for water blocks using the same geometric layout as terrain faces but omits texture layer and block light to maintain a uniform vertex format while reducing shader workload [S1, unofficial]. It builds on [Block Face Mesh Generation](10913-block-face-mesh-generation.md) by specializing the vertex packing for fluid blocks.
## How it works
**Inputs and outputs:**
- Input: Local face coordinates and neighbor block data required to determine visibility and lighting.
- Output: A packed vertex buffer for a single face, containing position, normal index, corner index, sky light level, and a water flag.

**Data structures:**
The method uses the identical `faceData[6][18]` layout as `addFace` [S1, unofficial]. This array stores the 6 vertices of a quad face, with each vertex represented by 18 values (likely floats or packed integers depending on the specific implementation details not fully explicit in the abstract, but structurally consistent with the terrain mesher).

**Steps:**
1. Compute local face coordinates using the identical `faceData[6][18]` layout as `addFace` [S1, unofficial].
2. Pack vertex data with `texLayer` hardcoded to 0 [S1, unofficial].
3. Omit `blockLightLevel` from the vertex attributes [S1, unofficial].
4. Store only the following attributes for each vertex:
   - Position
   - Normal index
   - Corner index
   - Sky light level
   - Water flag

**Formulas and Parameters:**
- `texLayer = 0` [S1, unofficial].
- `blockLightLevel` is excluded [S1, unofficial].

**Edge cases and pitfalls:**
- The uniform vertex format is critical for shader efficiency; deviating from this layout may require shader modifications [S1, unofficial].
- Precision issues are not explicitly detailed, but the omission of block light implies water blocks are assumed to be lit only by sky light or global illumination in this context [S1, unofficial].

## Subtopics
## Operating characteristics
**Time and memory complexity:**
Not given in the sources.

**Measured performance:**
Not given in the sources.

**Strengths and limitations:**
- Strength: Reduces shader workload by simplifying vertex attributes [S1, unofficial].
- Limitation: Assumes water blocks do not require block light or specific texture layers [S1, unofficial].

**When to use it and when not:**
- Use for water blocks in voxel engines that follow the Minecraft-like rendering model [S1, unofficial].
- Do not use for blocks requiring dynamic block lighting or multi-layer texturing [S1, unofficial].

## Background
Origin: This method is part of the `ft_minecraft` project, an unofficial implementation of Minecraft-like rendering [S1, unofficial]. It was developed to optimize the rendering of fluid blocks by reducing the data passed to the GPU.

## Sources
[S1] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
