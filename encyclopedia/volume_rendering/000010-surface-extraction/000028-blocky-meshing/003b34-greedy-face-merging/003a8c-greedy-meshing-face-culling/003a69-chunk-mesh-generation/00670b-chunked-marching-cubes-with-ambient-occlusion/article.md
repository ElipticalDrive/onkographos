# Chunked Marching Cubes with Ambient Occlusion

Also called: Chunk Mesh Builder

## Overview
This method generates a static, manifold mesh from a voxel chunk by exposing only the faces of non-empty voxels that border void space [S1, unofficial]. It differs from standard Marching Cubes by operating on a per-face basis rather than interpolating vertices within cells, resulting in a blocky, axis-aligned mesh suitable for voxel terrain [S1, unofficial]. It integrates baked ambient occlusion (AO) directly into the vertex data to provide shading without per-pixel lighting calculations [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `chunk`: The current voxel chunk object containing a 1D array of voxel IDs (`chunk.voxels`) and its world position (`chunk_pos`).
*   `world_voxels`: The global voxel data structure allowing access to neighboring chunks.
*   `CHUNK_SIZE`: The linear dimension of the chunk (e.g., 16 or 32) [S1, unofficial].
*   `WORLD_W`, `WORLD_H`, `WORLD_D`: Dimensions of the world in chunks [S1, unofficial].

**Outputs:**
*   `vertex_data`: A 1D array of `uint32` values representing packed vertex attributes.
*   `index_data`: (Implicit) Indices referencing the `vertex_data` array to form triangles.

### Data Structures
**Packed Vertex Format (`uint32`)**
Each vertex is packed into a single 32-bit unsigned integer. The bit layout from Most Significant Bit (MSB) to Least Significant Bit (LSB) is:
*   `x`: 6 bits (local X coordinate relative to chunk)
*   `y`: 6 bits (local Y coordinate relative to chunk)
*   `z`: 6 bits (local Z coordinate relative to chunk)
*   `voxel_id`: 8 bits (ID of the voxel generating this face)
*   `face_id`: 3 bits (Identifier for the face orientation: Top, Bottom, Right, Left, Back, Front)
*   `ao_id`: 2 bits (Index into the precomputed AO values for this face)
*   `flip_id`: 1 bit (Determines diagonal winding order)

Total: 32 bits [S1, unofficial].

**Memory Allocation**
The `vertex_data` array is pre-allocated with size `CHUNK_VOL * 18 * format_size`, where `CHUNK_VOL = CHUNK_SIZE^3` and `format_size = 4` bytes (for `uint32`) [S1, unofficial]. This upper bound accounts for the maximum possible vertices if every voxel face is exposed.

### Algorithm Steps
1.  **Initialization**: Calculate `CHUNK_AREA = CHUNK_SIZE * CHUNK_SIZE` and `CHUNK_VOL = CHUNK_SIZE * CHUNK_AREA` [S1, unofficial]. Initialize `vertex_data` array.
2.  **Voxel Iteration**: Iterate through every voxel index `i` from `0` to `CHUNK_VOL - 1`.
    *   Calculate local voxel position `(lx, ly, lz)` from `i`:
        $$lx = i \% CHUNK\_SIZE$$
        $$ly = (i // CHUNK\_SIZE) \% CHUNK\_SIZE$$
        $$lz = i // CHUNK\_AREA$$
    *   Calculate world position `(wx, wy, wz)` using chunk offset:
        $$wx = chunk\_pos.x + lx$$
        $$wy = chunk\_pos.y + ly$$
        $$wz = chunk\_pos.z + lz$$
    *   Retrieve `voxel_id = chunk.voxels[i]` [S1, unofficial].
3.  **Empty Check**: If `voxel_id == 0` (empty), skip to the next voxel [S1, unofficial].
4.  **Face Exposure Check**: For each of the 6 faces (Top, Bottom, Right, Left, Back, Front):
    *   Determine the neighbor's local/world position in that direction.
    *   Call `is_void(neighbor_local_pos, neighbor_world_pos, world_voxels)` [S1, unofficial].
    *   If the neighbor is void (or out of bounds), the face is exposed.
5.  **Ambient Occlusion Calculation**: For each exposed face:
    *   Call `get_ao(local_pos, world_pos, world_voxels, plane)` to compute 4 AO values [S1, unofficial].
    *   The `plane` parameter is 'X', 'Y', or 'Z' depending on the face normal.
6.  **Vertex Packing**: Pack the 4 corner vertices of the face quad. Each vertex includes:
    *   Local coordinates `(lx, ly, lz)` shifted to the specific corner.
    *   `voxel_id`.
    *   `face_id`.
    *   `ao_id` (index 0-3).
    *   `flip_id` (computed below).
7.  **Triangle Generation**: Add two triangles to the mesh for the quad.
    *   Use `flip_id` to determine the diagonal winding order.
    *   If `flip_id` is true, use one winding order; otherwise, use the opposite [S1, unofficial].
    *   Each quad adds 6 vertices (2 triangles * 3 vertices) to the output buffer.

### Formulas
**Flip ID Calculation**
The diagonal winding order is determined by the sum of opposite AO values to ensure consistent normal direction [S1, unofficial]:
$$flip\_id = (ao[1] + ao[3]) > (ao[0] + ao[2])$$
Where `ao[0]`, `ao[1]`, `ao[2]`, `ao[3]` are the 4 computed AO values for the face corners [S1, unofficial].

**AO Calculation (Y-Plane Example)**
For faces perpendicular to the Y-axis (Top/Bottom), the 8 surrounding voxels are checked. Let `a` through `h` be boolean results of `is_void()` for the 8 neighbors. The AO values are sums of these booleans:
$$ao[0] = a + b + c$$
$$ao[1] = g + h + a$$
$$ao[2] = e + f + g$$
$$ao[3] = c + d + e$$
The specific mapping of `a-h` to spatial positions depends on the face orientation (Top vs Bottom) and the coordinate system [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_chunk_mesh(chunk, world_voxels, CHUNK_SIZE, WORLD_W, WORLD_H, WORLD_D):
    CHUNK_AREA = CHUNK_SIZE * CHUNK_SIZE
    CHUNK_VOL = CHUNK_SIZE * CHUNK_AREA
    vertex_data = [0] * (CHUNK_VOL * 18 * 4) # Pre-allocated uint32 array
    vertex_count = 0
    
    chunk_pos = chunk.pos # (cx, cy, cz)
    
    for i in range(CHUNK_VOL):
        # Calculate local and world positions
        lx = i % CHUNK_SIZE
        ly = (i // CHUNK_SIZE) % CHUNK_SIZE
        lz = i // CHUNK_AREA
        
        wx = chunk_pos.x + lx
        wy = chunk_pos.y + ly
        wz = chunk_pos.z + lz
        
        voxel_id = chunk.voxels[i]
        
        if voxel_id == 0:
            continue
            
        # Check 6 faces
        faces = [
            ('Y', 1, 0, 0, 'Top'),    # Normal +Y
            ('Y', -1, 0, 0, 'Bottom'), # Normal -Y
            ('X', 0, 0, 1, 'Right'),   # Normal +X
            ('X', 0, 0, -1, 'Left'),   # Normal -X
            ('Z', 0, 1, 0, 'Front'),   # Normal +Z
            ('Z', 0, -1, 0, 'Back')    # Normal -Z
        ]
        
        for plane, dx, dy, dz, face_name in faces:
            # Neighbor position
            nx, ny, nz = wx + dx, wy + dy, wz + dz
            
            # Check if void
            if is_void((nx, ny, nz), world_voxels, WORLD_W, WORLD_H, WORLD_D):
                # Compute AO
                ao_values = get_ao((lx, ly, lz), (wx, wy, wz), world_voxels, plane)
                
                # Determine flip_id
                flip_id = (ao_values[1] + ao_values[3]) > (ao_values[0] + ao_values[2])
                
                # Generate 4 vertices for the quad
                # Note: Corner coordinates depend on face normal and flip_id
                # This is a simplified representation; actual corner offsets must be derived
                # from the face geometry and flip_id to ensure correct winding.
                
                vertices = generate_quad_vertices(lx, ly, lz, plane, flip_id, voxel_id, face_name, ao_values)
                
                for v in vertices:
                    vertex_data[vertex_count] = pack_vertex(v)
                    vertex_count += 1
                    
                # Add indices for 2 triangles (6 vertices)
                # Indices are relative to the start of this quad's vertices in vertex_data
                base_idx = vertex_count - 6
                # Triangle 1: 0, 1, 2
                # Triangle 2: 2, 1, 3 (or similar depending on flip_id)
                # Implementation detail: Indices are stored separately or interleaved.
                # The source implies vertex_data holds the vertices, indices are implied by order.
                
    return vertex_data

def is_void(pos, world_voxels, WORLD_W, WORLD_H, WORLD_D):
    x, y, z = pos
    # Boundary check: if out of world bounds, treat as solid (not void)
    if x < 0 or x >= WORLD_W * CHUNK_SIZE or \
       y < 0 or y >= WORLD_H * CHUNK_SIZE or \
       z < 0 or z >= WORLD_D * CHUNK_SIZE:
        return False
    
    # Determine chunk and local index
    cx, cy, cz = x // CHUNK_SIZE, y // CHUNK_SIZE, z // CHUNK_SIZE
    lx, ly, lz = x % CHUNK_SIZE, y % CHUNK_SIZE, z % CHUNK_SIZE
    
    # Check if chunk exists
    chunk_idx = cx + cy * WORLD_W + cz * WORLD_W * WORLD_H
    if chunk_idx < 0 or chunk_idx >= WORLD_W * WORLD_H * WORLD_D:
        return False # Out of bounds
        
    chunk = world_voxels[chunk_idx]
    if chunk is None:
        return False # Empty chunk is void? Or solid? Source says "returns False if out of bounds"
                     # implying non-void. Assuming empty chunk is void if not handled.
                     # However, standard voxel logic: if chunk doesn't exist, it's void.
                     # But source says "returns False if ... out of world bounds".
                     # Let's assume if chunk is None, it is void (True).
                     # Correction: Source says "is_void() returns False if the neighbor chunk index is out of world bounds".
                     # It does not specify behavior for existing but empty chunks.
                     # Assuming empty chunk voxels are 0, which is void.
    
    voxel_index = lx + CHUNK_SIZE * lz + CHUNK_AREA * ly
    return chunk.voxels[voxel_index] == 0

def get_ao(local_pos, world_pos, world_voxels, plane):
    x, y, z = local_pos
    wx, wy, wz = world_pos
    
    if plane == 'Y':
        # Y-plane neighbors (Top/Bottom)
        # a-h correspond to 8 surrounding voxels in the XZ plane at y-1 and y+1?
        # Source formula: ao[0] = a+b+c, etc.
        # We need to map a-h to specific neighbors.
        # Assuming standard ordering for Y-face:
        a = is_void((x, y, z - 1), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        b = is_void((x - 1, y, z - 1), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        c = is_void((x - 1, y, z), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        d = is_void((x - 1, y, z + 1), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        e = is_void((x, y, z + 1), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        f = is_void((x + 1, y, z + 1), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        g = is_void((x + 1, y, z), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        h = is_void((x + 1, y, z - 1), world_voxels, WORLD_W, WORLD_H, WORLD_D)
        
        ao = (a + b + c), (g + h + a), (e + f + g), (c + d + e)
        return ao
    else:
        # Similar logic for X and Z planes
        # Not given in the sources for X/Z specific mappings.
        # [G] Placeholder for X/Z plane AO calculation logic.
        return (0, 0, 0, 0)

def pack_vertex(vertex):
    # vertex is a dict or tuple with x, y, z, voxel_id, face_id, ao_id, flip_id
    x = vertex['x']
    y = vertex['y']
    z = vertex['z']
    voxel_id = vertex['voxel_id']
    face_id = vertex['face_id']
    ao_id = vertex['ao_id']
    flip_id = vertex['flip_id']
    
    # Pack into uint32
    # MSB to LSB: x(6), y(6), z(6), voxel_id(8), face_id(3), ao_id(2), flip_id(1)
    packed = 0
    packed |= (x & 0x3F) << 26
    packed |= (y & 0x3F) << 20
    packed |= (z & 0x3F) << 14
    packed |= (voxel_id & 0xFF) << 6
    packed |= (face_id & 0x07) << 3
    packed |= (ao_id & 0x03) << 1
    packed |= (flip_id & 0x01)
    
    return packed
```

### Edge Cases and Pitfalls
*   **Boundary Check**: `is_void()` must return `False` if the neighbor chunk index is out of world bounds. This prevents out-of-bounds memory access and treats world edges as solid/non-void [S1, unofficial].
*   **Winding Order Consistency**: The `flip_id` logic ensures that the normal vector points outward consistently regardless of the specific AO configuration, preventing back-face culling issues [S1, unofficial].
*   **Precision**: The 6-bit coordinate field limits local coordinates to 0-63. `CHUNK_SIZE` must be $\le 64$ to fit in 6 bits [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the chunk. Each voxel is processed once, and for each exposed face, 8 neighbor lookups are performed [S1, unofficial].
*   **Memory Complexity**: $O(N)$ for the vertex buffer, pre-allocated to the maximum possible size (`CHUNK_VOL * 18 * 4` bytes) [S1, unofficial].
*   **Strengths**: Simple implementation, fast generation, baked AO reduces runtime shading cost.
*   **Limitations**: Blocky geometry (no smooth surfaces), fixed resolution, high memory usage for large chunks due to pre-allocation.
*   **When to use**: Voxel-based games or applications where performance is critical and blocky aesthetic is acceptable.
*   **When not to use**: Applications requiring smooth surfaces or dynamic mesh deformation.

## Background
This method is derived from procedural voxel world generation techniques, specifically as implemented in the "PYVOXEL-Procedural-World" project [S1, unofficial]. It is a variant of greedy meshing or face culling techniques adapted for chunk-based rendering.

## Sources
[S1] GitHub - Shubhayu15/PYVOXEL-Procedural-World: A Python-based procedural ..., https://github.com/Shubhayu15/PYVOXEL-Procedural-World, fetched 2026-10-29 (unofficial)
