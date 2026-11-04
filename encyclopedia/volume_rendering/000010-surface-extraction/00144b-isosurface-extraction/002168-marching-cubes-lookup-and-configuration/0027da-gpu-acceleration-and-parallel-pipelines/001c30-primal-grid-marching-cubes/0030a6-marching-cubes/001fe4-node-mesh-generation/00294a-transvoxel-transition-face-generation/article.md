# Transvoxel Transition Face Generation

Also called: LOD Transition, Transvoxel, Transvoxel Transition

## Overview
Transvoxel Transition Face Generation is a method for creating manifold mesh geometry at the boundary between different levels of detail (LOD) in a voxel grid, specifically bridging the gap between a coarse LOD (LOD > 0) and the finest LOD (LOD 0) [S2, unofficial]. It differs from standard Marching Cubes by sampling a larger, non-uniform voxel block (the "transition cell") and computing midpoints to ensure connectivity, addressing the holes that typically arise when merging meshes of varying resolution [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D voxel grid (`float[,,] voxels`), the current Level of Detail (`int LOD`), the voxel scale (`float voxelScale`), and mesh data containers (`Mesh.MeshData`).
*   **Outputs:** A list of vertices and indices forming the transition face, stored in the provided mesh data structure.

**Data Structures:**
*   `vertexSharingIndexes`: A 3D integer array `int[firstSize + 1, secondSize + 1, 7]` used to track vertex reuse at the transition boundary. It is initialized to `-1` for all elements. The third dimension (size 7) corresponds to specific vertex positions or types within the transition cell structure [S2, unofficial].
*   `sample_positions`: A `vec4[13]` array storing corner, edge midpoint, and center positions for the transition block [S1, unofficial].
*   `MarchingCubesTables`: Contains lookup tables for transition cell classes and vertex data. The bit mapping is non-standard compared to classic Marching Cubes (e.g., corner 3 maps to `0x80`, corner 4 to `0x100`) [S2, unofficial].

**Parameters:**
*   `LOD`: The level of detail. Transition logic only executes if `LOD > 0` [S2, unofficial].
*   `step`: Calculated as `1 << LOD` [S2, unofficial].
*   `transitionCellSize`: A float controlling the size of the transition cell relative to the step. Values below 1 are recommended, with 0.5 being optimal [S2, unofficial].
*   `transvoxel_cell_multiplier`: An integer multiplier used to compute `transvoxel_size = 0.1f * transvoxel_cell_multiplier` [S1, unofficial].
*   `isoLevel`: The isovalue for surface extraction, typically `0.0f` in the reference implementation [S2, unofficial].

**Steps:**
1.  **Initialization:** Calculate `step = 1 << LOD`. If `LOD == 0`, return immediately as no transition face is needed for the finest level [S2, unofficial].
2.  **Face Iteration:** Iterate over the six faces of the voxel volume (Positive/Negative X, Y, Z). For each face, call `CreateTransitionFace` with the appropriate `startPosition`, `axisTriplet`, `faceDirection`, and target `meshData` array index [S2, unofficial].
3.  **Vertex Tracking:** Initialize `vertexSharingIndexes` with `-1` for all elements [S2, unofficial].
4.  **Grid Loop:** Loop through the 2D grid of the transition face. Start at `startPosition`. Increment the first axis by `step`. If the first axis exceeds `maxBounds`, reset it to `startPosition` and increment the second axis by `step`. Break if the second axis exceeds `maxBounds` [S2, unofficial].
5.  **Corner Sampling:** For each cube position, retrieve the 13 corner values using `SetTransitionCubeCornersValues`. This function samples the voxel grid at specific offsets (`0`, `halfStep`, `step`) along the two active axes, keeping the fixed axis constant. Corners 9-12 are reused from corners 0, 2, 6, and 8 respectively [S2, unofficial].
6.  **Cube Index Calculation:** Calculate the transition cube index using `CalculateTransitionCubeIndex`. This function compares corner values against `isoLevel` and sets bits in the index according to the non-standard mapping [S2, unofficial].
7.  **Lookup:** Look up the transition cell class and data from `MarchingCubesTables`. The class code determines winding direction (bit `0x80`) and the specific vertex configuration (masked with `0x7F`) [S2, unofficial].
8.  **Vertex Generation/Reuse:** For each vertex in the cell configuration:
    *   Extract `vertexCode` from `MarchingCubesTables.TransitionVertexData`.
    *   Calculate `vFirst` and `vSecond` indices based on the owner cube position and `step`.
    *   Check `vertexSharingIndexes[vFirst, vSecond, vertexIndexToReuse]`.
    *   If the sharing index is `-1` or the direction nibble is `4` (create but not own), create a new vertex by interpolating the edge using `InterpolateTransitionEdge`. Store the new vertex index in `vertexSharingIndexes` if the direction nibble is `8` (create and own) [S2, unofficial].
    *   If the sharing index is not `-1` and direction nibble is not `4`, reuse the existing vertex index from `vertexSharingIndexes` [S2, unofficial].
9.  **Triangle Generation:** Generate triangles from the cell data. If `reverseWinding` XOR `faceReverseWinding` is true, add vertices in order `(i, i+1, i+2)`. Otherwise, add them in swapped order `(i, i+2, i+1)` to correct winding [S2, unofficial].

**Vertex Interpolation Formula:**
The position of a vertex on an edge is calculated as:
$$ P = P_{base} + Lerp(P_a, P_b, t) $$
Where:
*   $P_{base}$ is the cube position scaled by `voxelScale`.
*   $P_a$ and $P_b$ are the scaled corner offsets for vertices $a$ and $b$.
*   $t = \frac{0 - corners[a]}{corners[b] - corners[a]}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CreateTransitionFace(voxels, voxelScale, step, startPosition, axisTriplet, faceDirection, meshData, transitionCellSize, faceReverseWinding):
    vertices = []
    triangles = []
    
    xSize = voxels.GetLength(0)
    ySize = voxels.GetLength(1)
    zSize = voxels.GetLength(2)
    
    firstSize = axisTriplet.First if axisTriplet.First in [X, Y, Z] else 0 # Simplified logic for pseudocode
    secondSize = axisTriplet.Second if axisTriplet.Second in [X, Y, Z] else 0
    
    # Initialize vertex sharing table
    vertexSharingIndexes = [[[ -1 for _ in range(7)] for _ in range(secondSize + 1)] for _ in range(firstSize + 1)]
    
    maxBounds = [xSize - 1, ySize - 1, zSize - 1]
    current = list(startPosition)
    
    while True:
        cubePosition = list(current)
        cubeCorners = [0.0] * 13
        transitionCornerOffset = [Vector3(0,0,0)] * 13
        
        # Sample corner values
        SetTransitionCubeCornersValues(cubeCorners, voxels, cubePosition, step, axisTriplet)
        
        # Calculate corner offsets for the transition cell geometry
        SetTransitionCornerOffset(transitionCornerOffset, transitionCellSize, axisTriplet, faceDirection)
        
        # Process the cube
        ProcessTransitionCube(cubePosition, cubeCorners, 0.0, voxelScale, step, 
                              vertices, triangles, vertexSharingIndexes, 
                              transitionCornerOffset, axisTriplet, faceReverseWinding)
        
        # Move to next cell in 2D grid
        first_axis_idx = axisTriplet.First
        second_axis_idx = axisTriplet.Second
        
        current[first_axis_idx] += step
        
        if current[first_axis_idx] >= maxBounds[first_axis_idx]:
            current[first_axis_idx] = startPosition[first_axis_idx]
            current[second_axis_idx] += step
            
            if current[second_axis_idx] >= maxBounds[second_axis_idx]:
                break
                
    SetMeshData(meshData, vertices, triangles)

def ProcessTransitionCube(cubePosition, corners, isoLevel, scale, step, vertices, triangles, vertexSharingIndexes, transitionCornerOffset, axisTriplet, faceReverseWinding):
    cubeIndex = CalculateTransitionCubeIndex(corners, isoLevel)
    
    transitionCellClassCode = MarchingCubesTables.transitionCellClass[cubeIndex]
    reverseWinding = (transitionCellClassCode & 0x80) != 0
    cellData = MarchingCubesTables.transitionCellData[transitionCellClassCode & 0x7F]
    
    vertexIndexes = []
    
    for i in range(cellData.GetVertexCount()):
        vertexCode = MarchingCubesTables.TransitionVertexData[cubeIndex][i]
        
        ownerCubePosition = GetOwnerTransitionCubePosition(cubePosition, vertexCode, step, axisTriplet)
        vertexIndexToReuse = (vertexCode & 0x0F00) >> 8
        vFirst = ownerCubePosition[axisTriplet.First] // step + 1
        vSecond = ownerCubePosition[axisTriplet.Second] // step + 1
        directionNibble = (vertexCode >> 12) & 0xF
        
        if vertexSharingIndexes[vFirst][vSecond][vertexIndexToReuse] == -1 or directionNibble == 4:
            firstVertex = vertexCode & 0x000F
            secondVertex = (vertexCode & 0x00F0) >> 4
            newVertex = InterpolateTransitionEdge(cubePosition, scale, step, firstVertex, secondVertex, corners, transitionCornerOffset)
            vertices.append(newVertex)
            newVertexIndex = len(vertices) - 1
            
            if directionNibble == 8:
                vertexSharingIndexes[vFirst][vSecond][vertexIndexToReuse] = newVertexIndex
                
            vertexIndexes.append(newVertexIndex)
        else:
            vertexIndexes.append(vertexSharingIndexes[vFirst][vSecond][vertexIndexToReuse])
            
    # Generate triangles
    for i in range(0, len(vertexIndexes), 3):
        if (reverseWinding ^ faceReverseWinding):
            triangles.append(vertexIndexes[i])
            triangles.append(vertexIndexes[i+1])
            triangles.append(vertexIndexes[i+2])
        else:
            triangles.append(vertexIndexes[i])
            triangles.append(vertexIndexes[i+2])
            triangles.append(vertexIndexes[i+1])

def InterpolateTransitionEdge(cubePosition, scale, step, firstVertex, secondVertex, corners, transitionCornerOffset):
    # Pseudocode for interpolation based on formula: P = P_base + Lerp(P_a, P_b, t)
    # t = (0 - corners[a]) / (corners[b] - corners[a])
    # Note: Exact mapping of firstVertex/secondVertex to corners[a/b] depends on specific table implementation
    # Assuming standard edge interpolation logic for transition cells
    val_a = corners[firstVertex]
    val_b = corners[secondVertex]
    
    if val_b == val_a:
        return Vector3(0,0,0) # Guard against division by zero
        
    t = (0.0 - val_a) / (val_b - val_a)
    
    # Base position scaled
    base_pos = Vector3(cubePosition[0] * scale, cubePosition[1] * scale, cubePosition[2] * scale)
    
    # Offsets from transitionCornerOffset
    offset_a = transitionCornerOffset[firstVertex]
    offset_b = transitionCornerOffset[secondVertex]
    
    # Lerp offsets
    lerp_offset = offset_a + t * (offset_b - offset_a)
    
    return base_pos + lerp_offset
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the surface area of the transition boundary, proportional to the number of transition cells. Each cell requires constant-time lookup and interpolation.
*   **Memory Complexity:** Proportional to the number of vertices in the transition mesh. The `vertexSharingIndexes` array adds overhead proportional to the boundary dimensions.
*   **Performance:** The implementation uses `UInt16` for indices, limiting the total vertex count to 65,535. Exceeding this requires switching to `UInt32` and `uint` [S2, unofficial].
*   **Strengths:** Ensures manifold connectivity between LOD levels, preventing visual holes.
*   **Limitations:** Requires careful handling of winding order across different faces, often necessitating manual correction flags (`faceReverseWinding`) [S2, unofficial]. The transition cell size parameter (`transitionCellSize`) affects visual quality and may require shader support to pull in vertices of the main mesh [S2, unofficial].

## Background
The Transvoxel algorithm was partially implemented in the context of real-time volume rendering with geometry shaders, allowing for simple LOD activation and surface shifting to reduce holes between different LODs [S1, unofficial]. The specific transition face generation method described here is derived from a Unity-based implementation [S2, unofficial].

## Sources
[S1] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
[S2] Verdant2202/TransvoxelUnityImplementation, https://github.com/Verdant2202/TransvoxelUnityImplementation, fetched 2026-10-27 (unofficial)
