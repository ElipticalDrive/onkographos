# Water Face Mesh Generation
Also called: addWaterFace
## Overview
Generates mesh vertices for water blocks using the same geometric layout as terrain faces but omits texture layer and block light to maintain a uniform vertex format while reducing shader workload [S1, unofficial]. It solves the problem of efficiently rendering dynamic fluid surfaces by simplifying vertex data, unlike static terrain models which carry full lighting and texture metadata [S1, unofficial].
## How it works
**Inputs and outputs:**
- Input: Block face geometry data (local coordinates, normals, corner indices) derived from the voxel grid.
- Output: A packed vertex buffer for a single face, containing position, normal index, corner index, sky light level, and a water flag.

**Data structures:**
The method uses the identical `faceData[6][18]` layout as `addFace` [S1, unofficial]. This array stores the 6 vertices of the face, with each vertex represented by 18 floats (or equivalent packed fields) defining position, normal, and other attributes.

**Steps:**
1. Compute local face coordinates using the identical `faceData[6][18]` layout as `addFace` [S1, unofficial].
2. Pack vertex data with `texLayer` hardcoded to 0 and `blockLightLevel` omitted [S1, unofficial].
3. Store only the following attributes for each vertex:
   - Position (3 floats)
   - Normal index (1 float/integer)
   - Corner index (1 float/integer)
   - Sky light level (1 float/integer)
   - Water flag (1 float/integer)

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def addWaterFace(faceData, numVertices):
    """
    Generates mesh vertices for water blocks.
    faceData: list of 6 vertices, each with 18 components (same layout as addFace).
    numVertices: number of vertices in the face (typically 6).
    Returns: Packed vertex data for GPU consumption.
    """
    packedVertices = []
    for i in range(numVertices):
        # Extract position, normal index, corner index, sky light from faceData
        # faceData[i] has 18 components; we assume standard layout from addFace
        # For simplicity, assume faceData[i] contains:
        # [pos_x, pos_y, pos_z, normal_idx, corner_idx, sky_light, ...rest...]
        
        pos_x = faceData[i][0]
        pos_y = faceData[i][1]
        pos_z = faceData[i][2]
        normal_idx = faceData[i][3]
        corner_idx = faceData[i][4]
        sky_light = faceData[i][5]
        
        # Pack vertex data
        # texLayer hardcoded to 0
        tex_layer = 0
        # blockLightLevel omitted
        water_flag = 1  # Indicates this is a water block
        
        packedVertex = [
            pos_x, pos_y, pos_z,
            normal_idx,
            corner_idx,
            sky_light,
            water_flag,
            tex_layer
        ]
        packedVertices.append(packedVertex)
    
    return packedVertices
```

**Parameters:**
- `texLayer`: Hardcoded to 0 [S1, unofficial].
- `waterFlag`: Indicates water block presence [S1, unofficial].

**Edge cases and pitfalls:**
- Precision: Ensure that position and normal indices are accurately preserved to avoid visual artifacts [S1, unofficial].
- Boundaries: Handle edge cases where water blocks are adjacent to non-water blocks to maintain consistent mesh topology [S1, unofficial].
- Degenerate input: If `faceData` is empty or malformed, the method should return an empty vertex list [S1, unofficial].

## Subtopics

### [VoxelBlockyModelFluid](12770-voxelblockymodelfluid.md)
`VoxelBlockyModelFluid` is a procedural model for rendering flowing water with multiple levels, designed to mimic the visual style of games like Minecraft. It solves the problem of efficiently rendering dynamic fluid surfaces without the memory overhead of precomputing variations for every possible fluid height combination. Unlike static terrain models, it calculates vertex displacements on the fly during mesh generation to raise top corners to match the highest neighbor fluid of the same type.

## Operating characteristics
- Time complexity: Linear with respect to the number of vertices in the face, as it processes each vertex independently [S1, unofficial].
- Memory complexity: Reduced compared to terrain faces due to the omission of `blockLightLevel` and `texLayer` [S1, unofficial].
- Measured performance: Not specified in the sources [S1, unofficial].
- Scaling: Scales well with the number of water blocks, as the method is applied per face [S1, unofficial].
- Strengths: Simplified vertex format reduces shader workload and memory usage [S1, unofficial].
- Limitations: Lacks detailed lighting and texture information, which may affect visual fidelity in complex scenes [S1, unofficial].
- When to use: Ideal for rendering dynamic water surfaces in voxel-based games where performance is critical [S1, unofficial].
- When not to use: Not suitable for static terrain or non-water blocks that require full lighting and texture data [S1, unofficial].

## Background
Origin: The method is part of the `ft_minecraft` project, an unofficial implementation of Minecraft-like voxel rendering [S1, unofficial]. It was developed to optimize the rendering of water blocks by simplifying vertex data [S1, unofficial].
History and development: The method builds on the geometric layout of terrain faces but adapts it for water blocks by omitting unnecessary attributes [S1, unofficial].
Where it is used: Primarily in voxel-based games and simulations that require efficient rendering of dynamic fluid surfaces [S1, unofficial].

## Sources
[S1] ldominiq/ft_minecraft, https://github.com/ldominiq/ft_minecraft, fetched 2026-10-27 (unofficial)
