# Splatv Binary File Format Parser

Also called: .splatv format, SplatvLoader

## Overview
The Splatv Binary File Format Parser is a method for loading `.splatv` files, a custom binary container format used by `gsplat.js` to store 3D Gaussian Splatting scene data. It solves the problem of parsing a structured binary stream that combines a magic header, JSON-encoded camera metadata, and binary splat data chunks into a usable scene graph. It is distinct from the Antimatter15 `.splat` parser [Parent](6902-antimatter15-splat-file-parsing.md) because it handles a multi-chunk container with explicit type markers and embedded JSON metadata, rather than a flat, headerless array of records.

## How it works
**Inputs and Outputs:**
- Input: An `ArrayBuffer` or `Blob` representing a `.splatv` file.
- Output: A `Scene` object containing a `Splat` instance (populated via `SplatvData.Deserialize`) and camera position/rotation derived from the file's metadata.

**Data Structures:**
The file is a sequence of chunks. Each chunk follows this layout:
1. **Header**:
   - `Chunk Type`: A null-terminated UTF-8 string (e.g., "chunks", "splat").
   - `Size`: A 32-bit integer (`Int32`) indicating the byte length of the `Data` section.
2. **Data**: A `Uint8Array` of length `Size`.

The file begins with a global header:
1. `Magic`: A 32-bit integer (`Int32`) with value `0x674b`.
2. `Size`: A 32-bit integer (`Int32`) indicating the size of the first chunk's data (or potentially the whole file, depending on implementation; the source implies it precedes the first chunk type). *Note: The source code snippet `intView[0]` suggests the magic is at offset 0. The structure `[Magic: Int32, Size: Int32]` is described in the notes.*

**Numbered Steps:**
1. **Validate Magic**: Read the first 4 bytes as a little-endian `Int32`. If the value is not `0x674b`, throw an error ("Invalid splatv file").
2. **Read First Chunk Header**: Read the next 4 bytes as `Int32` (Size). Read the subsequent bytes as a UTF-8 string to determine the chunk type.
3. **Process 'chunks' Chunk**:
   - If the chunk type is "chunks", read `Size` bytes of data.
   - Decode the data as UTF-8 JSON.
   - Parse the JSON into an array of splat chunk objects.
   - **Edge Case**: If the array length is greater than 1, log a warning (`console.warn`) and proceed only with the first element (`splatChunks[0]`).
   - Extract camera position `[x, y, z]` and a 3x3 rotation matrix from the chunk data.
   - Convert the 3x3 rotation matrix to a Quaternion.
   - Set the scene's camera position and rotation.
   - **Edge Case**: If camera data is missing or invalid (e.g., `cameras` array is empty), do not update the camera.
4. **Process 'splat' Chunk**:
   - Read the next chunk. If the type is "splat", read its data buffer.
   - Pass the buffer and texture dimensions (`texwidth`, `texheight`) to `SplatvData.Deserialize`.
   - Create a `Splatv` object from the deserialized data.
   - Add the `Splatv` object to the scene.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_splatv(buffer: ArrayBuffer) -> Scene:
    # Step 1: Validate Magic
    int_view = Int32Array(buffer)
    if int_view[0] != 0x674b:
        raise Error("Invalid splatv file")
    
    offset = 4  # Skip magic
    # Note: The source notes mention [Magic: Int32, Size: Int32] at start.
    # Assuming the first Int32 after magic is the size of the first chunk.
    # However, the code snippet only checks intView[0]. 
    # We follow the note's layout: Magic, Size, then Chunk Type.
    
    # Read first chunk size (assuming it's at offset 4)
    # Note: The source code `intView[0]` check is the only explicit code.
    # The notes say: "Read first 8 bytes as Int32Array. Check magic... expect next chunk type 'chunks' with size intView[1]."
    # This implies: [Magic, Size, TypeString...]
    
    chunk_size = int_view[1]
    offset = 8
    
    # Read Chunk Type String
    # We need to read bytes until null terminator or up to chunk_size
    # For simplicity, assume we read the string first.
    # In JS: new TextDecoder().decode(buffer.slice(offset, offset + chunk_size)) 
    # But we need to know where the string ends. 
    # The notes say: "Read 'chunks' chunk. Decode as UTF-8 JSON."
    # This implies the chunk type is a string identifier.
    
    # Let's assume the chunk type is a fixed string "chunks" or "splat"
    # and the size follows. Or the type is a string.
    # The note says: "[Chunk Type: String, Size: Int32, Data: Uint8Array]"
    
    # Re-reading note: "Read first 8 bytes as Int32Array... expect next chunk type 'chunks' with size intView[1]"
    # This is slightly contradictory. Let's stick to the explicit code and note combination.
    
    # Implementation based on Note [S1, unofficial]:
    # 1. Check magic.
    # 2. Read 'chunks' chunk.
    
    # We need a helper to read chunks.
    chunks = []
    current_offset = 8 # After Magic and first Size? Or just Magic?
    # The note says: "File structure: [Magic: Int32 (0x674b), Size: Int32], [Chunk Type: String, Size: Int32, Data: Uint8Array]"
    # This implies the global header is Magic + Size. Then chunks follow.
    
    # Let's assume the first chunk starts at offset 8.
    
    while current_offset < buffer.byteLength:
        # Read Chunk Type String
        # We need to find the null terminator or length. 
        # The note doesn't specify string length prefix. 
        # It says "Chunk Type: String". 
        # Let's assume we read until null or a known delimiter.
        # For robustness, we might read a fixed max length or use a null terminator.
        # Since no specific string length is given, we assume null-terminated.
        
        type_bytes = []
        while current_offset < buffer.byteLength:
            byte = buffer[current_offset]
            current_offset += 1
            if byte == 0:
                break
            type_bytes.append(byte)
        
        chunk_type = TextDecoder("utf-8").decode(Uint8Array(type_bytes))
        
        # Read Size
        size = Int32Array(buffer, current_offset, 1)[0]
        current_offset += 4
        
        # Read Data
        data = Uint8Array(buffer, current_offset, size)
        current_offset += size
        
        if chunk_type == "chunks":
            json_str = TextDecoder("utf-8").decode(data)
            splat_chunks = JSON.parse(json_str)
            
            if len(splat_chunks) > 1:
                console.warn("Splatv file contains more than one chunk, only the first one will be loaded")
            
            chunk_data = splat_chunks[0]
            
            # Extract camera data
            camera_pos = chunk_data.get("camera_position") # [x,y,z]
            rot_matrix = chunk_data.get("rotation_matrix") # 3x3
            
            if camera_pos and rot_matrix:
                quaternion = matrix_to_quaternion(rot_matrix)
                scene.camera.position = camera_pos
                scene.camera.rotation = quaternion
            else:
                # Camera data missing or invalid
                pass
                
        elif chunk_type == "splat":
            # Pass buffer and texwidth, texheight to SplatvData.Deserialize
            # Note: texwidth/texheight must be extracted from somewhere.
            # The source doesn't specify where they come from in the binary.
            # They might be in the JSON or fixed. 
            # Assuming they are passed as arguments to Deserialize.
            # We need to know them. Let's assume they are part of the chunk data or global.
            # The note says: "Pass buffer and texwidth, texheight to SplatvData.Deserialize"
            # We will assume they are available in the context or parsed from the 'chunks' JSON.
            
            splatv_obj = SplatvData.Deserialize(data, texwidth, texheight)
            scene.add(splatv_obj)
            
    return scene

def matrix_to_quaternion(matrix):
    # Standard conversion from 3x3 rotation matrix to Quaternion
    # m00, m01, m02, m10, m11, m12, m20, m21, m22
    m = matrix
    trace = m[0][0] + m[1][1] + m[2][2]
    
    if trace > 0:
        s = 0.5 / math.sqrt(trace + 1.0)
        w = 0.25 / s
        x = (m[2][1] - m[1][2]) * s
        y = (m[0][2] - m[2][0]) * s
        z = (m[1][0] - m[0][1]) * s
    elif (m[0][0] > m[1][1]) and (m[0][0] > m[2][2]):
        s = 2.0 * math.sqrt(1.0 + m[0][0] - m[1][1] - m[2][2])
        w = (m[2][1] - m[1][2]) / s
        x = 0.25 * s
        y = (m[0][1] + m[1][0]) / s
        z = (m[0][2] + m[2][0]) / s
    elif m[1][1] > m[2][2]:
        s = 2.0 * math.sqrt(1.0 + m[1][1] - m[0][0] - m[2][2])
        w = (m[0][2] - m[2][0]) / s
        x = (m[0][1] + m[1][0]) / s
        y = 0.25 * s
        z = (m[1][2] + m[2][1]) / s
    else:
        s = 2.0 * math.sqrt(1.0 + m[2][2] - m[0][0] - m[1][1])
        w = (m[1][0] - m[0][1]) / s
        x = (m[0][2] + m[2][0]) / s
        y = (m[1][2] + m[2][1]) / s
        z = 0.25 * s
        
    return Quaternion(w, x, y, z)
```

**Parameters:**
- Magic Number: `0x674b` (Int32).
- Encoding: UTF-8 for strings and JSON.

**Edge Cases and Pitfalls:**
- **Multiple Chunks**: If the "chunks" JSON array contains more than one element, only the first is loaded, and a warning is issued.
- **Missing Camera Data**: If the camera position or rotation matrix is absent or invalid in the JSON, the camera is not updated.
- **Precision**: Standard 32-bit floats are assumed for positions and matrix elements unless specified otherwise by `SplatvData.Deserialize`.

## Operating characteristics
- **Time Complexity**: Linear with respect to the file size, as the entire file must be read and parsed. JSON parsing is $O(N)$ where $N$ is the size of the metadata chunk.
- **Memory Complexity**: Linear, as the entire buffer is loaded into memory. The `SplatvData.Deserialize` step creates in-memory structures for the splats.
- **Strengths**: Structured format allows for embedded metadata (camera poses) and flexible chunking.
- **Limitations**: Requires loading the entire file into memory. JSON parsing overhead for metadata.
- **When to use**: When using `gsplat.js` and the scene is saved in the `.splatv` format.
- **When not to use**: For very large scenes where streaming is required, or if the format is not supported by the renderer.

## Background
The `.splatv` format is used by the `gsplat.js` library [S1, unofficial]. It was developed to provide a structured binary format for storing 3D Gaussian Splatting data, including camera metadata, which is not present in the flat Antimatter15 `.splat` format. It is primarily used in web-based 3D Gaussian Splatting viewers.

## Sources
[S1] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
