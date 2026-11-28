# Block Mesh Alpha Rendering

Also called: ChunkMesh Alpha Rendering, Transparent Face Sorting & Rendering

## Overview
Block Mesh Alpha Rendering is a rendering technique for voxel-based environments that uses a custom index ordering algorithm to sort visible faces back-to-front relative to the camera's primary viewing axis, enabling correct alpha blending without full scene depth sorting [S1, unofficial]. It sorts transparent chunk meshes by distance to the camera and renders them with blending enabled to handle transparency correctly [S1, unofficial]. Unlike general-purpose transparent rendering which requires global depth sorting, this method exploits the grid structure of voxel chunks to determine a valid draw order efficiently [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of voxel chunks (typically $32 \times 32 \times 32$), camera position $(C_x, C_y, C_z)$, camera view vector $(v_x, v_y, v_z)$, and a collection of transparent chunk meshes [S1, unofficial].
*   **Output:** A rasterized image with correctly blended transparent surfaces [S1, unofficial].

**Data Structures:**
*   **Vertex Packing:** Each vertex is stored as two 32-bit unsigned integers (`vertex0`, `vertex1`) [S1, unofficial].
    *   `vertex0`:
        *   Bits 0-9: $x$ position (10 bits) [S1, unofficial]
        *   Bits 10-19: $y$ position (10 bits) [S1, unofficial]
        *   Bits 20-31: $z$ position (12 bits) [S1, unofficial]
    *   `vertex1`:
        *   Bits 0-9: Texture layer (10 bits) [S1, unofficial]
        *   Bits 10-14: Texture coordinate $x$ (5 bits) [S1, unofficial]
        *   Bits 15-19: Texture coordinate $y$ (5 bits) [S1, unofficial]
        *   Bits 20-31: Shade data (12 bits) [S1, unofficial]
*   **Block Position Index (`blocksPos`):** A nested array structure used to track visible transparent blocks for index generation [S1, unofficial]. The structure follows `blocksPos[greatestDir][secondary][tertiary]`, where each leaf node contains `[coordinate, [indices]]` and `blocksPos[greatestDir][secondary][tertiary][1]` holds the list of vertex indices for that block [S1, unofficial].

**Parameters:**
*   **Chunk Size:** 32 units [S1, unofficial].
*   **Fog Far Distance:** 256 units ($32 \times 8$) [S1, unofficial].
*   **Camera Matrix Scale:** $0.0625$ ($1/16$) to convert world units to chunk units [S1, unofficial].
*   **Alpha Discard Threshold:** 0.4. Fragments with alpha $< 0.4$ are discarded to prevent artifacts from effectively invisible surfaces [S1, unofficial].

**Algorithm Steps:**

1.  **Determine Primary Axis:** Calculate the absolute values of the camera's view vector components $(v_x, v_y, v_z)$ [S1, unofficial]. Identify `greatestDir` as the axis (X, Y, or Z) with the largest absolute component [S1, unofficial].
2.  **Determine Traversal Direction:** Calculate `greatestMag` as the signed magnitude of the view vector component along `greatestDir` [S1, unofficial]. If `greatestMag` is negative, reverse the list of blocks along the `greatestDir` axis to ensure traversal starts from the back relative to the camera view [S1, unofficial].
3.  **Calculate Chunk Distances:** For all chunks, compute the Manhattan distance from the camera to the chunk center [S1, unofficial]. The distance is calculated as `chunkDist = |chunk.x * 32 - C_x + 16| + |chunk.y * 32 - C_y + 16| + |chunk.z * 32 - C_z + 16|` [S1, unofficial]. Sort the chunks in descending order of `chunkDist` (far to near) [S1, unofficial].
4.  **Generate Indices for Transparent Meshes:** For each shader's transparent chunk meshes, perform the following nested iteration to build the index buffer [S1, unofficial]:
    *   Let `blocksPos` be the index structure for the current shader [S1, unofficial].
    *   Iterate through the `blocksPos` array along the `greatestDir` axis [S1, unofficial].
    *   For each block at index $i$ along `greatestDir`:
        *   Iterate through the secondary axis blocks (index $j$) using a rectilinear distance check against the camera's position on the other two axes (`otherPos`) [S1, unofficial].
        *   **Forward Tertiary Iteration:** Iterate through tertiary axis blocks (index $k$) from start to end [S1, unofficial]. If the block's coordinate is $\ge$ the camera's coordinate on the tertiary axis, break the loop [S1, unofficial]. Add all indices from `blocksPos[i][j][k][1]` to the output index list [S1, unofficial].
        *   **Reverse Tertiary Iteration:** Iterate through tertiary axis blocks (index $k$) from end to start [S1, unofficial]. If the block's coordinate is $<$ the camera's coordinate on the tertiary axis, break the loop [S1, unofficial]. Add all indices from `blocksPos[i][j][k][1]` to the output index list [S1, unofficial].
        *   **Secondary Axis Reverse:** After processing all tertiary blocks for the forward secondary iteration, iterate through secondary axis blocks (index $j$) in reverse order [S1, unofficial].
        *   **Forward Tertiary (Reverse Secondary):** For each secondary block in reverse, iterate through tertiary axis blocks (index $k$) from start to end [S1, unofficial]. Apply the same coordinate comparison logic against `otherPos` [S1, unofficial]. Add indices if valid [S1, unofficial].
        *   **Reverse Tertiary (Reverse Secondary):** Iterate through tertiary axis blocks (index $k$) from end to start [S1, unofficial]. Apply the same coordinate comparison logic [S1, unofficial]. Add indices if valid [S1, unofficial].
    *   If the traversal direction was reversed in Step 2, reverse the `blocksPos` list back to its original order [S1, unofficial].
5.  **Render:**
    *   Bind the generated index buffer [S1, unofficial].
    *   Set blend function: `gl.blendFuncSeparate(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA, gl.ONE, gl.ONE_MINUS_SRC_ALPHA)` [S1, unofficial].
    *   Draw the triangles [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_block_mesh_alpha_indices(blocks_pos, camera_pos, view_vector):
    """
    Generates draw indices for transparent block meshes.
    blocks_pos: dict or nested list structure indexed by axis direction (0,1,2).
    camera_pos: tuple (cx, cy, cz)
    view_vector: tuple (vx, vy, vz)
    """
    # 1. Determine greatestDir and greatestMag
    abs_vx = abs(view_vector[0])
    abs_vy = abs(view_vector[1])
    abs_vz = abs(view_vector[2])
    
    if abs_vx >= abs_vy and abs_vx >= abs_vz:
        greatest_dir = 0
        greatest_mag = view_vector[0]
    elif abs_vy >= abs_vx and abs_vy >= abs_vz:
        greatest_dir = 1
        greatest_mag = view_vector[1]
    else:
        greatest_dir = 2
        greatest_mag = view_vector[2]
        
    # Determine if we need to reverse the initial traversal order
    should_reverse_list = greatest_mag < 0
    
    indices = []
    
    # Get the specific blocks_pos structure for the determined direction
    axis_map = ['X', 'Y', 'Z']
    current_blocks = blocks_pos[axis_map[greatest_dir]]
    
    # Reverse list if traversal direction is negative
    if should_reverse_list:
        current_blocks.reverse()
        
    # Map axes
    secondary_axis = (greatest_dir + 1) % 3
    tertiary_axis = (greatest_dir + 2) % 3
    
    # Camera positions on secondary and tertiary axes
    other_pos = [camera_pos[secondary_axis], camera_pos[tertiary_axis]]
    
    # Process blocks along greatestDir
    for i in range(len(current_blocks)):
        block_i = current_blocks[i]
        
        # Secondary axis iteration (forward)
        for j in range(len(block_i)):
            block_ij = block_i[j]
            
            # Rectilinear check against camera position on secondary axis
            if block_ij[0] >= other_pos[0]:
                break
                
            # Forward Tertiary Iteration
            for k in range(len(block_ij)):
                block_ijk = block_ij[k]
                if block_ijk[0] >= other_pos[1]:
                    break
                indices.extend(block_ijk[1])
                
            # Reverse Tertiary Iteration
            for k in range(len(block_ij) - 1, -1, -1):
                block_ijk = block_ij[k]
                if block_ijk[0] < other_pos[1]:
                    break
                indices.extend(block_ijk[1])
                
        # Reverse Secondary Axis Iteration
        for j in range(len(block_i) - 1, -1, -1):
            block_ij = block_i[j]
            
            # Rectilinear check against camera position on secondary axis
            if block_ij[0] >= other_pos[0]:
                break
                
            # Forward Tertiary (for reverse secondary)
            for k in range(len(block_ij)):
                block_ijk = block_ij[k]
                if block_ijk[0] >= other_pos[1]:
                    break
                indices.extend(block_ijk[1])
                
            # Reverse Tertiary (for reverse secondary)
            for k in range(len(block_ij) - 1, -1, -1):
                block_ijk = block_ij[k]
                if block_ijk[0] < other_pos[1]:
                    break
                indices.extend(block_ijk[1])

    # Reverse list back to original order if it was reversed
    if should_reverse_list:
        current_blocks.reverse()
        
    return indices

def render_block_mesh_alpha(gl, chunk_meshes, camera_pos, view_vector, fog_far):
    """
    Renders the scene using Block Mesh Alpha Rendering.
    """
    # 1. Calculate Chunk Distances
    chunk_distances = {}
    for chunk_id, chunk in chunk_meshes.items():
        # Manhattan distance: |chunk.x * 32 - camera.x + 16| + ...
        dist = abs(chunk.pos[0] * 32 - camera_pos[0] + 16) + \
               abs(chunk.pos[1] * 32 - camera_pos[1] + 16) + \
               abs(chunk.pos[2] * 32 - camera_pos[2] + 16)
        chunk_distances[chunk_id] = dist
        
    # 2. Sort Chunks Far to Near
    sorted_chunks = sorted(chunk_meshes.items(), key=lambda item: chunk_distances[item[0]], reverse=True)
    
    # 3. Set Blend Function
    # gl.blendFuncSeparate(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA, gl.ONE, gl.ONE_MINUS_SRC_ALPHA)
    
    # 4. Render Each Chunk
    for chunk_id, chunk in sorted_chunks:
        # Call processCameraData to update indices for transparency
        chunk.process_camera_data(camera_pos)
        gl.bindVertexArray(chunk.vao)
        gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, chunk.index_buffer)
        gl.drawElements(gl.TRIANGLES, chunk.index_count, gl.UNSIGNED_INT, 0)
    
    return chunk_distances

def fragment_shader_shading(vertex1):
    """
    Calculates shading value from packed vertex data.
    vertex1: 32-bit unsigned integer
    """
    # Extract shade data from bits 20-31
    shade_data = (vertex1 >> 20) * 0.2 + 0.4
    return shade_data

def fragment_shader_fog(distance, fog_far):
    """
    Calculates fog amount.
    distance: float, length of camera-space position vector
    fog_far: float, 256.0
    """
    # smoothstep(fogFar - 64.0, fogFar, distance)
    edge0 = fog_far - 64.0
    edge1 = fog_far
    
    # Clamp distance to [edge0, edge1]
    t = max(0.0, min(1.0, (distance - edge0) / (edge1 - edge0)))
    
    # Smoothstep polynomial
    fog_amount = t * t * (3.0 - 2.0 * t)
    return fog_amount
```

**Edge Cases and Pitfalls:**
*   **Alpha Threshold:** Transparent blocks with alpha $< 0.4$ are discarded in the fragment shader. This prevents rendering artifacts from semi-transparent surfaces that are effectively invisible [S1, unofficial].
*   **Index Structure Assumption:** The index generation algorithm assumes a specific nested array structure for `blocksPos`: `blocksPos[greatestDir][secondary][tertiary][1]` contains the list of indices for each block [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** This method is optimized for voxel-based environments with frequent updates (block placing/destroying) [S1, unofficial]. It avoids the high cost of global depth sorting by leveraging the grid structure [S1, unofficial].
*   **Strengths:** Efficient handling of transparency in voxel worlds; supports dynamic updates; reduces overdraw by sorting chunks [S1, unofficial].
*   **Limitations:** Limited to voxel/chunk-based geometry; sorting is approximate (based on chunk distance and axis-aligned traversal), which may cause minor blending artifacts at chunk boundaries if not handled carefully [S1, unofficial].

## Background
Block Mesh Alpha Rendering was developed to provide efficient rendering of transparent blocks (like glass or water) in a WebGL-based voxel environment. The technique is described in the project's source code and documentation [S1, unofficial].

## Sources
[S1] kvxy/exolands, https://github.com/kvxy/exolands, fetched 2026-10-27 (unofficial)
