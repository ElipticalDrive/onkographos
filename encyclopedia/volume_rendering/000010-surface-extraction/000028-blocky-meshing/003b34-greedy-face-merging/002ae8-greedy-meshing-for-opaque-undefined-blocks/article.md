# Greedy Meshing for Opaque/Undefined Blocks

Also called: RightGreedyMesh

## Overview
This method generates a mesh for opaque voxel blocks by greedily merging adjacent faces of identical material into larger quads [S2, unofficial]. It specifically targets the right-facing surface normal ($+X$), iterating along the X-axis to construct the mesh in the Z-Y plane [S2, unofficial].

## How it works
**Inputs:**
- `blockArray1D`: 1D array of block IDs representing the chunk volume.
- `blockArray1D_right`: 1D array of block IDs representing the immediate neighbor chunk to the right ($+X$ direction) [S2, unofficial].
- `chunkWidth`, `chunkHeight`, `chunkLength`: Dimensions of the chunk in blocks [S2, unofficial].
- `chunkToRight`: Boolean flag indicating if a right neighbor exists [S2, unofficial].
- `blockSize`: Physical size of a single block unit [G].
- `colors`: Array of color values indexed by block ID [G].

**Outputs:**
- `verticies`: List of vertex positions.
- `uv`: List of UV coordinates.
- `colors`: List of vertex colors.

**Data Structures:**
- `blocks`: A 2D mask array of size `chunkLength` $\times$ `chunkHeight`, indexed as `blocks[z + (chunkLength * y)]`. It stores the block ID if the face is visible and unprocessed, or `0` if processed or hidden [S2, unofficial].

**Steps:**
1. **Initialization:** Initialize empty lists for `verticies`, `uv`, and `colors` [S2, unofficial].
2. **Sweep Loop:** Iterate X from `chunkWidth - 1` down to `0` [S2, unofficial].
   - If `x == chunkWidth - 1` and `!chunkToRight`, continue to the next X iteration [S2, unofficial].
3. **Slice Construction:** Construct the `blocks` mask for the current X slice [S2, unofficial].
   - If `x < chunkWidth - 1`:
     - Check `blockArray1D` at index `x + 1`.
     - If `blockArray1D[x + 1] > 0`, set all entries in `blocks` to `0` (face is internal, hidden by neighbor) [S2, unofficial].
     - Else, copy `blockArray1D` values for the current X slice into `blocks` [S2, unofficial].
   - If `x == chunkWidth - 1`:
     - Check `blockArray1D_right` at index `0` [S2, unofficial].
     - If `blockArray1D_right[0] > 0`, set all entries in `blocks` to `0` [S2, unofficial].
     - Else, copy `blockArray1D` values for the current X slice into `blocks` [S2, unofficial].
4. **Greedy Expansion:** Iterate Z from `0` to `chunkLength` and Y from `0` to `chunkHeight` [S2, unofficial].
   - Let `blockID = blocks[z + (chunkLength * y)]` [S2, unofficial].
   - If `blockID == 0`, continue [S2, unofficial].
   - **Expand Height:** Initialize `height = 1`. While `y + height < chunkHeight` and `blocks[z + (chunkLength * (y + height))] == blockID`, increment `height` [S2, unofficial].
   - **Expand Length:** Initialize `length = 1`, `canExtend = true`. While `canExtend` and `z + length < chunkLength`:
     - Check if all cells in the column `z + length` for Y range `[y, y + height)` match `blockID` [S2, unofficial].
     - If yes, `length++`.
     - Else, `canExtend = false` [S2, unofficial].
5. **Mark Processed:** Set `blocks[(z + i) + (chunkLength * (y + j))] = 0` for `i` in `[0, length)` and `j` in `[0, height)` [S2, unofficial].
6. **Vertex Generation:**
   - Calculate `size = blockSize` [G].
   - `xPos = x * size` [S2, unofficial].
   - `yCenter = (y + height / 2.0f) * size` [S2, unofficial].
   - `zCenter = (z + length / 2.0f) * size` [S2, unofficial].
   - `halfHeight = (height * size) / 2.0f` [S2, unofficial].
   - `halfLength = (length * size) / 2.0f` [S2, unofficial].
   - `color = colors[blockID - 1]` [S2, unofficial].
   - Add 4 vertices to `verticies`:
     1. `(xPos + size, yCenter - halfHeight, zCenter - halfLength)` [S2, unofficial].
     2. `(xPos + size, yCenter + halfHeight, zCenter - halfLength)` [S2, unofficial].
     3. `(xPos + size, yCenter + halfHeight, zCenter + halfLength)` [S2, unofficial].
     4. `(xPos + size, yCenter - halfHeight, zCenter + halfLength)` [S2, unofficial].
   - Extend `uv` with 4 sets of `[0.0, 2.0]` UV coordinates [S1, unofficial].
   - Extend `colors` with 4 sets of `[0.0, 4.0]` color coordinates (RGBA) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def right_greedy_mesh(block_array_1d, block_array_1d_right, chunk_width, chunk_height, chunk_length, block_size, colors, chunk_to_right):
    """
    Generates a mesh for the right-facing (+X) surfaces of a voxel chunk.
    
    Args:
        block_array_1d: List[int], block IDs for the current chunk.
        block_array_1d_right: List[int], block IDs for the chunk to the right.
        chunk_width: int, width of the chunk in blocks.
        chunk_height: int, height of the chunk in blocks.
        chunk_length: int, length of the chunk in blocks.
        block_size: float, physical size of one block.
        colors: List[List[float]], colors indexed by block ID (1-based).
        chunk_to_right: bool, True if a right neighbor exists.
        
    Returns:
        tuple: (vertices, uvs, vertex_colors)
    """
    vertices = []
    uvs = []
    vertex_colors = []
    
    # Iterate X from right to left
    for x in range(chunk_width - 1, -1, -1):
        # If it's the rightmost block and no neighbor, skip (assuming air boundary)
        if x == chunk_width - 1 and not chunk_to_right:
            continue
            
        # Construct the 2D mask for this X slice
        # Mask is indexed by z + (chunk_length * y)
        mask = [0] * (chunk_length * chunk_height)
        
        # Determine visibility based on neighbor
        if x < chunk_width - 1:
            # Check immediate right neighbor in the same chunk
            # Follows source's direct indexing assumption [S2, unofficial]
            neighbor_val = block_array_1d[x + 1]
            if neighbor_val > 0:
                # Face is internal, mask remains 0
                pass
            else:
                # Face is exposed, copy current X slice to mask
                for y in range(chunk_height):
                    for z in range(chunk_length):
                        idx = x + y * chunk_width + z * chunk_width * chunk_height
                        mask[z + (chunk_length * y)] = block_array_1d[idx]
        else: # x == chunk_width - 1
            # Check right neighbor chunk
            if chunk_to_right:
                neighbor_val = block_array_1d_right[0]
                if neighbor_val > 0:
                    pass # Internal
                else:
                    # Exposed, copy from x
                    for y in range(chunk_height):
                        for z in range(chunk_length):
                            idx = x + y * chunk_width + z * chunk_width * chunk_height
                            mask[z + (chunk_length * y)] = block_array_1d[idx]
            else:
                # No neighbor, assume air boundary -> exposed
                for y in range(chunk_height):
                    for z in range(chunk_length):
                        idx = x + y * chunk_width + z * chunk_width * chunk_height
                        mask[z + (chunk_length * y)] = block_array_1d[idx]

        # Greedy Expansion on the mask
        for y in range(chunk_height):
            for z in range(chunk_length):
                block_id = mask[z + (chunk_length * y)]
                if block_id == 0:
                    continue
                
                # Expand Height (Y)
                height = 1
                while y + height < chunk_height:
                    if mask[z + (chunk_length * (y + height))] == block_id:
                        height += 1
                    else:
                        break
                
                # Expand Length (Z)
                length = 1
                can_extend = True
                while can_extend and z + length < chunk_length:
                    # Check all cells in the column z+length for Y range [y, y+height)
                    match = True
                    for dy in range(height):
                        if mask[(z + length) + (chunk_length * (y + dy))] != block_id:
                            match = False
                            break
                    if match:
                        length += 1
                    else:
                        can_extend = False
                
                # Mark as processed
                for dy in range(height):
                    for dz in range(length):
                        mask[(z + dz) + (chunk_length * (y + dy))] = 0
                
                # Generate Vertices
                size = block_size
                x_pos = x * size
                y_center = (y + height / 2.0) * size
                z_center = (z + length / 2.0) * size
                half_height = (height * size) / 2.0
                half_length = (length * size) / 2.0
                
                # Color lookup (1-based index in source)
                color = colors[block_id - 1]
                
                # Add 4 vertices
                vertices.append((x_pos + size, y_center - half_height, z_center - half_length))
                vertices.append((x_pos + size, y_center + half_height, z_center - half_length))
                vertices.append((x_pos + size, y_center + half_height, z_center + half_length))
                vertices.append((x_pos + size, y_center - half_height, z_center + half_length))
                
                # Add UVs
                uvs.extend([0.0, 2.0] * 4)
                
                # Add Colors
                vertex_colors.extend([0.0, 4.0] * 4)
                
    return vertices, uvs, vertex_colors
```

## Subtopics

### Chunk Neighbor Optimization
An optimization where chunks are made aware of their neighboring chunks to remove inner polygons that are never visible [S3, unofficial]. It solves the issue where neighboring chunks previously did not know about each other [S3, unofficial]. The world is constructed cube by cube, passing neighbor coordinates to the current cube to check visibility against each side as it is created [S3, unofficial]. A pitfall is Unity's restriction of ~65,000 vertices per mesh; a 65,536 cube chunk exceeds this, requiring recombination of multiple chunks [S3, unofficial].

### Greedy Meshing (Right Face)
This is the specific orientation implemented by the main method, focusing on the $+X$ normal. It iterates X backwards and merges in the Z-Y plane [S2, unofficial].

### Greedy Meshing (Right Face)
Not given in the sources.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Reduces vertex count by merging adjacent faces. Simple to implement. [G]
- **Limitations:** Only handles one face orientation at a time. Requires separate passes for other normals. [G]
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Originating from voxel engine implementations, this method is part of the broader greedy meshing family. The "RightGreedyMesh" name appears in Unity Voxel Engine implementations [S2, unofficial].

## Sources
[S1] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S2] BloodyFish/UnityVoxelEngine, https://github.com/BloodyFish/UnityVoxelEngine, fetched 2026-10-27 (unofficial)
[S3] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
