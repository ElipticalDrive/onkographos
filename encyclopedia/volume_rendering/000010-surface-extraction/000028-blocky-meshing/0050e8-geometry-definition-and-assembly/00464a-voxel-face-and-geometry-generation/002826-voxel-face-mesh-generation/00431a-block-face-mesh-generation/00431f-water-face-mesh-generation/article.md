# Water Face Mesh Generation
Also called: addWaterFace
## Overview
Generates mesh vertices for water blocks using the same geometric layout as terrain faces but omits texture layer and block light to maintain a uniform vertex format while reducing shader workload [S1, unofficial].
## How it works
**Inputs and outputs:**
- Input: Block face geometry data (local coordinates, normals, corner indices) derived from the voxel grid [S1, unofficial].
- Output: A packed vertex buffer for a single face, containing position, normal index, corner index, sky light level, and a water flag [S1, unofficial].

**Data structures:**
The method uses the identical `faceData[6][18]` layout as `addFace` [S1, unofficial]. This array stores the 6 vertices of the face, with each vertex represented by 18 floats (or equivalent packed fields) defining position, normal, and other attributes [S1, unofficial].

**Steps:**
1. Compute local face coordinates using the identical `faceData[6][18]` layout as `addFace` [S1, unofficial].
2. Pack vertex data with `texLayer` hardcoded to 0 and `blockLightLevel` omitted [S1, unofficial].
3. Store only the following attributes for each vertex: position (3 floats), normal index (1 float/integer), corner index (1 float/integer), sky light level (1 float/integer), and water flag (1 float/integer) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def addWaterFace(faceData):
    """Generates mesh vertices for water blocks.
    faceData: list of 6 vertices, each with 18 components (same layout as addFace).
    Returns: Packed vertex data for GPU consumption."""
    packedVertices = []
    for i in range(6):
        pos = faceData[i][0:3]
        normal_idx = faceData[i][3]
        corner_idx = faceData[i][4]
        sky_light = faceData[i][5]
        water_flag = 1.0
        tex_layer = 0.0
        packedVertex = pos + [normal_idx, corner_idx, sky_light, water_flag, tex_layer]
        packedVertices.append(packedVertex)
    return packedVertices
```

**Parameters:**
- `texLayer`: Hardcoded to 0 [S1, unofficial].
- `waterFlag`: Indicates water block presence [S1, unofficial].

**Edge cases and pitfalls:**
Not given in the sources.

## Subtopics

### [VoxelBlockyModelFluid](12770-voxelblockymodelfluid.md)
`VoxelBlockyModelFluid` is a procedural model for rendering flowing water with multiple levels, designed to mimic the visual style of games like Minecraft. It solves the problem of efficiently rendering dynamic fluid surfaces without the memory overhead of precomputing variations for every possible fluid height combination. Unlike static terrain models, it calculates vertex displacements on the fly during mesh generation to raise top corners to match the highest neighbor fluid of the same type.

## Operating characteristics
Not given in the sources.

## Background
Origin: The method is part of the `ft_minecraft` project, an unofficial implementation of Minecraft-like voxel rendering [S1, unofficial].
History and development: The method builds on the geometric layout of terrain faces but adapts it for water blocks by omitting unnecessary attributes [S1, unofficial].
Where it is used: Not given in the sources.

## Sources
[S1] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
