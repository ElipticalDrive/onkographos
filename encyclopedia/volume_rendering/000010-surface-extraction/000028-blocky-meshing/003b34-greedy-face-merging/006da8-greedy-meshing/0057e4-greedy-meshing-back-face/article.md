# Greedy Meshing (Back Face)

Also called: BackGreedyMesh

## Overview
Greedy Meshing (Back Face) is a specialized surface extraction method that generates quads for the back-facing surfaces (negative Z direction) of a voxel chunk [S1, unofficial]. It extends the 2D greedy meshing algorithm to three dimensions by extracting 2D slices from the 3D voxel grid along the Z-axis and processing them to merge adjacent blocks of the same type into larger rectangles [S1, unofficial]. It is distinct from general greedy meshing [18808-greedy-meshing.md] because it operates exclusively on the back-facing boundary, requiring specific handling of neighbor chunks to determine visibility at the chunk's rear edge [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `blockArray1D`: A 1D `NativeArray<byte>` representing the voxel data of the current chunk [S1, unofficial].
    *   `blockArray1D_back`: A 1D `NativeArray<byte>` representing the voxel data of the neighbor chunk located behind the current chunk (in the negative Z direction) [S1, unofficial].
    *   `colors`: A `NativeArray<Color32>` mapping block IDs to vertex colors [S1, unofficial].
    *   `chunkWidth`, `chunkLength`, `chunkHeight`: Integer dimensions of the chunk [S1, unofficial].
    *   `blockSize`: Float size of a single voxel [S1, unofficial].
    *   `chunkToBack`: Boolean flag indicating if the chunk has a back neighbor [S1, unofficial].
*   **Outputs**:
    *   `vertices`: A `NativeList<GreedyVertex>` containing the generated quad vertices [S1, unofficial].
    *   `triangles`: A `NativeList<int>` containing the triangle indices [S1, unofficial].

**Data Structures**
The algorithm uses a temporary 1D array `blocks` of size `chunkWidth * chunkHeight` to store the processed visibility state for the current Z-slice [S1, unofficial]. The layout is row-major: `blocks[x + (chunkWidth * y)]` [S1, unofficial].

**Algorithm Steps**
1.  **Iterate Z-Slices**: Loop `z` from 0 to `chunkLength` [S1, unofficial].
    *   If `z == 0` and `chunkToBack` is false, skip this iteration (no back face to generate) [S1, unofficial].
2.  **Initialize Slice Visibility**: Create the temporary `blocks` array for the current `z`-slice [S1, unofficial].
    *   For each `x` in `0..chunkWidth-1` and `y` in `0..chunkHeight-1`:
        *   Calculate the 1D index in the neighbor chunk: `neighbor_idx = x + (chunkWidth * (z - 1)) + (chunkWidth * chunkLength * y)` [S1, unofficial].
        *   If `z > 0`:
            *   Check if the neighbor voxel exists: if `blockArray1D_back[neighbor_idx] > 0`, set `blocks[x + (chunkWidth * y)] = 0` (occluded) [S1, unofficial].
            *   Otherwise, copy the current chunk's block ID: `blocks[x + (chunkWidth * y)] = blockArray1D[x + (chunkWidth * z) + (chunkWidth * chunkLength * y)]` [S1, unofficial].
        *   If `z == 0`:
            *   Check the back neighbor at the end of the chunk: `neighbor_idx = x + (chunkWidth * (chunkLength - 1)) + (chunkWidth * chunkLength * y)` [S1, unofficial].
            *   If `blockArray1D_back[neighbor_idx] > 0`, set `blocks[x + (chunkWidth * y)] = 0` [S1, unofficial].
            *   Otherwise, copy the current block ID [S1, unofficial].
3.  **Greedy Merge (2D)**: Iterate `x` from 0 to `chunkWidth-1` and `y` from 0 to `chunkHeight-1` [S1, unofficial].
    *   Let `blockID = blocks[x + (chunkWidth * y)]` [S1, unofficial].
    *   If `blockID == 0`, continue (already consumed or empty) [S1, unofficial].
    *   **Calculate Height**:
        *   Initialize `height = 0`.
        *   While `y + height < chunkHeight` and `blocks[x + (chunkWidth * (y + height))] == blockID`:
            *   Increment `height`.
    *   **Calculate Width**:
        *   Initialize `width = 1`.
        *   Initialize `canExtend = true`.
        *   While `canExtend` and `x + width < chunkWidth`:
            *   Check if the entire vertical strip at `x + width` matches `blockID` for all `h` in `0..height-1` [S1, unofficial].
            *   Condition: `blocks[(x + width) + (chunkWidth * (y + h))] == blockID` for all `h`.
            *   If mismatch, set `canExtend = false` [S1, unofficial].
            *   If match, increment `width`.
    *   **Mark Consumed**:
        *   For `i` in `0..width-1` and `j` in `0..height-1`:
            *   Set `blocks[(x + i) + (chunkWidth * (y + j))] = 0` [S1, unofficial].
    *   **Generate Quad**:
        *   Calculate center coordinates:
            *   `xCenter = (x + width / 2.0f) * blockSize`
            *   `yCenter = (y + height / 2.0f) * blockSize`
            *   `zPos = z * blockSize`
        *   Calculate half-dimensions:
            *   `halfWidth = (width * blockSize) / 2.0f`
            *   `halfHeight = (height * blockSize) / 2.0f`
        *   Define vertices (back face, Z = `zPos`):
            *   `v0 = (xCenter + halfWidth, yCenter - halfHeight, zPos)`
            *   `v1 = (xCenter + halfWidth, yCenter + halfHeight, zPos)`
            *   `v2 = (xCenter - halfWidth, yCenter + halfHeight, zPos)`
            *   `v3 = (xCenter - halfWidth, yCenter - halfHeight, zPos)`
        *   Add vertices to `vertices` list.
        *   Let `index` be the current vertex count before adding these four.
        *   Add triangles to `triangles` list:
            *   `(index + 0, index + 1, index + 2)`
            *   `(index + 2, index + 3, index + 0)`

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_back_face(blockArray1D, blockArray1D_back, colors, chunkWidth, chunkLength, chunkHeight, blockSize, chunkToBack):
    vertices = []
    triangles = []
    
    # Temporary buffer for the current Z-slice
    blocks = [0] * (chunkWidth * chunkHeight)
    
    # Iterate z from 0 to chunkLength-1 (inclusive bounds for array indexing)
    for z in range(chunkLength):
        # Skip if no back neighbor and we are at the front
        if z == 0 and not chunkToBack:
            continue
            
        # Initialize the slice visibility
        for y in range(chunkHeight):
            for x in range(chunkWidth):
                idx = x + (chunkWidth * y)
                
                if z > 0:
                    # Check neighbor chunk at z-1
                    neighbor_idx = x + (chunkWidth * (z - 1)) + (chunkWidth * chunkLength * y)
                    if blockArray1D_back[neighbor_idx] > 0:
                        blocks[idx] = 0
                    else:
                        current_idx = x + (chunkWidth * z) + (chunkWidth * chunkLength * y)
                        blocks[idx] = blockArray1D[current_idx]
                else:
                    # z == 0, check the back of the current chunk against the neighbor
                    neighbor_idx = x + (chunkWidth * (chunkLength - 1)) + (chunkWidth * chunkLength * y)
                    if blockArray1D_back[neighbor_idx] > 0:
                        blocks[idx] = 0
                    else:
                        current_idx = x + (chunkWidth * z) + (chunkWidth * chunkLength * y)
                        blocks[idx] = blockArray1D[current_idx]
                        
        # 2D Greedy Meshing on the slice
        for y in range(chunkHeight):
            for x in range(chunkWidth):
                blockID = blocks[x + (chunkWidth * y)]
                
                if blockID == 0:
                    continue
                
                # Calculate Height
                height = 0
                while y + height < chunkHeight and blocks[x + (chunkWidth * (y + height))] == blockID:
                    height += 1
                
                # Calculate Width
                width = 1
                canExtend = True
                while canExtend and x + width < chunkWidth:
                    # Check if the entire column at x+width matches blockID for height
                    match = True
                    for h in range(height):
                        if blocks[(x + width) + (chunkWidth * (y + h))] != blockID:
                            match = False
                            break
                    if match:
                        width += 1
                    else:
                        canExtend = False
                
                # Mark as consumed
                for i in range(width):
                    for j in range(height):
                        blocks[(x + i) + (chunkWidth * (y + j))] = 0
                
                # Generate Quad
                xCenter = (x + width / 2.0) * blockSize
                yCenter = (y + height / 2.0) * blockSize
                zPos = z * blockSize
                halfWidth = (width * blockSize) / 2.0
                halfHeight = (height * blockSize) / 2.0
                
                v0 = (xCenter + halfWidth, yCenter - halfHeight, zPos)
                v1 = (xCenter + halfWidth, yCenter + halfHeight, zPos)
                v2 = (xCenter - halfWidth, yCenter + halfHeight, zPos)
                v3 = (xCenter - halfWidth, yCenter - halfHeight, zPos)
                
                vertices.append(v0)
                vertices.append(v1)
                vertices.append(v2)
                vertices.append(v3)
                
                index = len(vertices) - 4
                triangles.append((index + 0, index + 1, index + 2))
                triangles.append((index + 2, index + 3, index + 0))
                
    return vertices, triangles
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions**: The algorithm relies on `blockArray1D_back` to determine visibility at the chunk's rear. If `chunkToBack` is false, the front-most slice (`z=0`) is skipped, but subsequent slices are processed normally. If `blockArray1D_back` is not provided or empty, visibility checks may fail or return incorrect occlusion states [S1, unofficial].
*   **Precision**: Floating-point calculations for vertex centers and half-dimensions use standard float precision. For very large chunks, potential floating-point drift may occur, though `blockSize` is typically small [S1, unofficial].
*   **Degenerate Input**: If `blockArray1D` contains only 0s, no vertices or triangles are generated. If `chunkWidth` or `chunkHeight` is 0, the loops do not execute.

## Subtopics

### Greedy Meshing (Back Face)
Generates quads for the back-facing surfaces. Chosen when only the rear boundary of a chunk needs to be meshed, often as part of a multi-pass system. Link: [Greedy Meshing (Back Face)](8784-greedy-meshing-back-face.md).

### Greedy Meshing (Front Face)
Generates quads for the front-facing surfaces. Complements the back face method. Link: [Greedy Meshing (Front Face)](8785-greedy-meshing-front-face.md).

### Greedy Meshing (Side Faces)
Generates quads for the left, right, top, and bottom faces. Link: [Greedy Meshing (Side Faces)](8786-greedy-meshing-side-faces.md).

## Operating characteristics
*   **Time Complexity**: $O(W \cdot H \cdot L)$, where $W, H, L$ are chunk dimensions. Each voxel is visited a constant number of times during the slice initialization and greedy merge steps.
*   **Memory Complexity**: $O(W \cdot H)$ for the temporary `blocks` array per slice. The output size depends on the number of merged quads.
*   **Performance**: Measured performance depends on the density of visible blocks. High occlusion (many 0s) reduces the number of quads generated. The algorithm is efficient for sparse voxel grids.
*   **Strengths**: Reduces triangle count by merging adjacent quads. Simple to implement.
*   **Limitations**: Only handles back-facing surfaces. Requires neighbor chunk data for accurate boundary culling. Not suitable for generating the entire mesh in a single pass.

## Background
Greedy Meshing (Back Face) is part of a broader family of greedy meshing algorithms used in voxel engines. It was implemented in the UnityVoxelEngine project by BloodyFish [S1, unofficial]. The method extends the 2D greedy meshing concept to 3D by processing slices along the Z-axis, specifically targeting the back-facing boundary. It is often used in conjunction with other face-specific greedy meshing methods to generate the complete mesh for a voxel chunk.

## Sources
[S1] BloodyFish/UnityVoxelEngine, https://github.com/BloodyFish/UnityVoxelEngine, fetched 2026-10-27 (unofficial)
