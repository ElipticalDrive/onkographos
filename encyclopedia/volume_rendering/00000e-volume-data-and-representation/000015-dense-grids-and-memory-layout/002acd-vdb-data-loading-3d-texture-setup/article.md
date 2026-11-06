# VDB Data Loading & 3D Texture Setup

Also called: VDB Loader, VDB to 3D Texture, getVDBData

## Overview
This method loads a VDB-formatted binary file, parses its sparse voxel coordinates and tile data, and uploads the resulting dense volume as a 3D texture for GPU rendering [S1, unofficial]. It bridges the gap between sparse VDB storage and the dense grid requirements of [3D Texture Storage and Sampling](3064-3d-texture-storage-and-sampling.md). Unlike [Bricking](914-bricking.md) which manages memory hierarchy for large datasets, this method focuses on the initial conversion of a sparse binary representation into a contiguous GPU texture, suitable for moderate-resolution volumes where the entire dataset fits in VRAM.

## How it works
**Inputs:** A VDB binary file (or `ArrayBuffer`), GPU context (WebGL/WebGPU).
**Outputs:** A bound 3D texture object containing the volume data.

**Data Structures:**
1.  **VDB Header:** Fixed-size binary structure.
    *   `width` (uint16, offset 0)
    *   `height` (uint16, offset 2)
    *   `depth` (uint16, offset 4)
    *   `nodeCount` (uint16, offset 6)
2.  **Coordinates:** Array of `uint16` triplets $(x, y, z)$ representing tile positions. Size: $3 \times \text{nodeCount}$. Starts at offset 8.
3.  **Tile Data:** Array of `uint8` values representing an $8 \times 8 \times 8$ tile. Size: $512 \times \text{nodeCount}$. Starts immediately after coordinates.
4.  **Cloud Data:** A `Uint8Array` of size $W \times H \times D$, initialized to zero.

**Steps:**
1.  **Cache Check:** Check a `dataRepository` for an existing texture associated with the file. If found, bind and return [S1, unofficial].
2.  **File Load:** Download the file as an `ArrayBuffer` [S1, unofficial].
3.  **Header Parsing:**
    *   Read `resolution = [getUint16(0), getUint16(2), getUint16(4)]` [S1, unofficial].
    *   Read `nodeCount = getUint16(6)` [S1, unofficial].
4.  **Dimension Calculation:**
    *   Compute tile dimensions: $D_{tile} = [\lfloor \text{resolution}[0]/8 \rfloor, \lfloor \text{resolution}[1]/8 \rfloor, \lfloor \text{resolution}[2]/8 \rfloor]$ [S1, unofficial].
    *   *Note:* The source implementation assumes resolution is a multiple of 8 or truncates. For robustness, ensure resolution is padded to multiples of 8 before division if the VDB spec requires it [G].
5.  **Memory Allocation:**
    *   Allocate `cloudData` as a `Uint8Array` of size $\text{resolution}[0] \times \text{resolution}[1] \times \text{resolution}[2]$ [S1, unofficial].
    *   Initialize all bytes to 0.
6.  **Data Parsing:**
    *   Extract `coordinates` as a `Uint16Array` starting at offset 8.
    *   Extract `nodeData` as a `Uint8Array` starting at offset $8 + \text{coordinates.byteLength}$ [S1, unofficial].
7.  **Tile Placement:**
    *   Iterate $i$ from $0$ to $\text{nodeCount} - 1$:
        *   Extract tile position: $pos = \text{coordinates.subarray}(i \times 3, i \times 3 + 3)$ [S1, unofficial].
        *   Extract tile values: $vals = \text{nodeData.subarray}(i \times 512, i \times 512 + 512)$ [S1, unofficial].
        *   Call `setTile(cloudData, dimensions, pos, vals)` to place the $8 \times 8 \times 8$ block into `cloudData` at the correct global coordinates.
8.  **Texture Upload:**
    *   Create a 3D texture object.
    *   Bind to `gl.TEXTURE0`.
    *   Upload `cloudData` using `gl.texImage3D` with dimensions `resolution`, format `gl.RED`, type `gl.UNSIGNED_BYTE` [S1, unofficial].
    *   Set texture parameters (min/mag filter, wrap mode) as required by the renderer [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_vdb_to_texture(file_buffer, gl_context):
    # 1. Cache check
    if gl_context.data_repository.has_texture(file_buffer.id):
        return gl_context.data_repository.get_texture(file_buffer.id)

    # 2. Parse Header
    # Assuming little-endian uint16
    width = file_buffer.get_uint16(0)
    height = file_buffer.get_uint16(2)
    depth = file_buffer.get_uint16(4)
    node_count = file_buffer.get_uint16(6)

    # 3. Calculate Dimensions
    # Source uses integer division; ensure resolution is multiple of 8 for full coverage
    dim_x = width // 8
    dim_y = height // 8
    dim_z = depth // 8
    
    total_voxels = width * height * depth
    cloud_data = bytearray(total_voxels) # Initialized to 0

    # 4. Parse Coordinates and Data
    # Coordinates start at offset 8, each is 3 uint16s
    coords_offset = 8
    coords_byte_len = node_count * 3 * 2 # 3 coords * 2 bytes
    
    # Coordinates array
    coords = file_buffer.get_uint16_array(coords_offset, node_count * 3)
    
    # Node data starts immediately after coordinates
    data_offset = coords_offset + coords_byte_len
    # Each node has 512 uint8s (8x8x8)
    node_data = file_buffer.get_uint8_array(data_offset, node_count * 512)

    # 5. Place Tiles
    for i in range(node_count):
        # Extract position (x, y, z) in tile coordinates
        tx = coords[i * 3]
        ty = coords[i * 3 + 1]
        tz = coords[i * 3 + 2]
        
        # Extract tile data (512 bytes)
        tile_src = node_data[i * 512 : (i + 1) * 512]
        
        # Place tile into cloud_data
        # Global voxel coordinates:
        # x = tx * 8 + local_x
        # y = ty * 8 + local_y
        # z = tz * 8 + local_z
        base_idx = (tz * dim_y * dim_x + ty * dim_x + tx) * 512 # This is incorrect for flat array mapping
        
        # Correct mapping for flat 3D array (width, height, depth)
        # We need to map the 8x8x8 tile to the global grid
        for local_z in range(8):
            for local_y in range(8):
                for local_x in range(8):
                    global_x = tx * 8 + local_x
                    global_y = ty * 8 + local_y
                    global_z = tz * 8 + local_z
                    
                    # Check bounds (source doesn't explicitly check, but implied by valid VDB)
                    if global_x < width and global_y < height and global_z < depth:
                        # Index into flat array: z * (width * height) + y * width + x
                        idx = global_z * (width * height) + global_y * width + global_x
                        cloud_data[idx] = tile_src[local_z * 64 + local_y * 8 + local_x]

    # 6. Upload to GPU
    texture_id = gl_context.create_texture_3d()
    gl_context.bind_texture(gl.TEXTURE_3D, texture_id)
    gl_context.tex_image_3d(
        target=gl.TEXTURE_3D,
        level=0,
        internal_format=gl.R8, # or gl.RED
        width=width,
        height=height,
        depth=depth,
        border=0,
        format=gl.RED,
        type=gl.UNSIGNED_BYTE,
        data=cloud_data
    )
    
    # Set filtering (G: Standard practice for volume rendering)
    gl_context.tex_parameteri(gl.TEXTURE_3D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
    gl_context.tex_parameteri(gl.TEXTURE_3D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
    gl_context.tex_parameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
    gl_context.tex_parameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
    gl_context.tex_parameteri(gl.TEXTURE_3D, gl.TEXTURE_WRAP_R, gl.CLAMP_TO_EDGE)

    # Cache and return
    gl_context.data_repository.add_texture(file_buffer.id, texture_id)
    return texture_id
```

**Parameters:**
*   `gl.R8` / `gl.RED` / `gl.UNSIGNED_BYTE`: The GPU texture format used for single-channel 8-bit data [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Offset Calculation:** Coordinates and data offsets must be calculated based on `coordinates.byteLength`. Mismatching the header layout results in reading garbage data [S1, unofficial].
*   **Resolution Alignment:** The source code divides resolution by 8. If the VDB file contains a resolution not divisible by 8, the last partial tiles are ignored or cause indexing errors unless padding is applied [G].
*   **Sparse vs Dense:** This method allocates a dense array of size $W \times H \times D$. For very large sparse volumes, this will exhaust memory. It is only suitable when the dense footprint is manageable [G].

## Subtopics

### VDB 16-bit Tile Loading
Loads a VDB file format consisting of 8^3 leaf tiles, parsing the binary header and tile data, and uploading it as a 3D texture [S1, unofficial].
*   **Definition:** Extends the 8-bit loader to handle 16-bit tile data, requiring compression to 8-bit for `R8` texture upload [S1, unofficial].
*   **Header Layout:**
    *   Offset 0-1: `width` (uint16, little endian)
    *   Offset 2-3: `height` (uint16)
    *   Offset 4-5: `depth` (uint16)
    *   Offset 6-7: `format` (uint16)
    *   Offset 8-9: `sparse` (uint16)
    *   Offset 10-11: `block size` (uint16)
    *   Offset 12-15: `nodeCount` (uint32)
    *   Offset 16-19: `normalization` (float32)
    *   Offset 20-31: `offsets` (int32[3])
    *   Offset 32+: `coordinates` (int32[3] * nodeCount)
    *   Followed by: `nodeData` (uint16[512] * nodeCount) [S1, unofficial].
*   **Steps:**
    1.  Parse header to get resolution (rounded up to multiples of 8), dimensions (resolution/8), and nodeCount [S1, unofficial].
    2.  Extract coordinates and nodeData arrays from the binary buffer [S1, unofficial].
    3.  For each node $i$, call `setTile(cloudData, dimensions, coordinates[i], nodeData[i*512])` [S1, unofficial].
    4.  Compress 16-bit data to 8-bit: if `cloudData[i] > 0`, `compressedData[i] = 255 * (cloudData[i] / 65535)` [S1, unofficial].
    5.  Upload `compressedData` to a 3D texture with format `R8` [S1, unofficial].
*   **When to choose:** Use when the source VDB file contains 16-bit precision data but the target GPU texture format is limited to 8-bit per channel, or when memory bandwidth is a constraint and 8-bit precision is sufficient for the visualization.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of nodes (tiles) plus the cost of allocating the dense grid $O(W \times H \times D)$. The parsing and placement are linear in the number of tiles.
*   **Memory Complexity:** $O(W \times H \times D)$ for the dense `cloudData` array plus $O(\text{file size})$ for the buffer. This is the primary limitation; it requires the entire volume to fit in RAM during loading.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Simple implementation; leverages GPU hardware texture sampling; efficient for moderate-sized volumes.
*   **Limitations:** High memory usage for large sparse volumes; 8-bit precision loss in the 16-bit variant; assumes resolution is a multiple of 8.
*   **When to use:** Small to medium volumes that fit in VRAM; prototyping; applications where sparse VDB is used for editing but dense texture is needed for rendering.
*   **When not to use:** Very large sparse volumes (use [Bricking](914-bricking.md) or [Chunk-based Voxel Data Structure](1818-chunk-based-voxel-data-structure.md)); when 16-bit precision is critical and GPU supports it (use `R16` texture if available).

## Background
This method originates from the `webgl-3d-math` module and the `volumetrics` project by al-ro, designed for WebGL-based volume rendering [S1, unofficial]. It addresses the need to convert sparse, tile-based VDB data into a format consumable by standard WebGL 3D texture samplers.

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
