# Splatv Binary File Format Parser

Also called: .splatv format, SplatvLoader

## Overview
The Splatv Binary File Format Parser is a loader for the `.splatv` binary format, which encapsulates Gaussian splat data alongside embedded camera metadata in a single file [S1, unofficial]. It solves the problem of initializing a rendering scene with both geometry and view parameters without requiring separate asset files. Unlike the [Antimatter15 .splat File Parsing](6902-antimatter15-splat-file-parsing.md) method, which handles flat, headerless arrays of splat records, Splatv includes a structured header and JSON-encoded camera information, allowing the parser to automatically configure the scene's initial viewpoint [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: An `ArrayBuffer` or binary stream representing a `.splatv` file.
- Output: A `Splat` object (containing `SplatvData`) and a configured camera (position and rotation) within the scene graph.

**Data Structures**
The file consists of a sequence of chunks. Each chunk follows this layout:
1.  **Header**: 8 bytes total.
    -   Bytes 0-3: Magic number (`Int32`).
    -   Bytes 4-7: Chunk data size (`Int32`).
2.  **Chunk Type**: A null-terminated UTF-8 string indicating the chunk type (e.g., `"chunks"`, `"splat"`).
3.  **Chunk Data**: `Uint8Array` of length equal to the size specified in the header.

**Numbered Steps**
1.  **Magic Number Validation**: Read the first 4 bytes as an `Int32`. Compare against the magic constant `0x674b`. If they do not match, abort with an error [S1, unofficial].
2.  **Chunk Iteration**: Read the next 4 bytes as the size of the first chunk's data. Read the chunk type string.
3.  **Process 'chunks' Chunk**:
    -   If the chunk type is `"chunks"`, decode the data as UTF-8 JSON [S1, unofficial].
    -   Parse the JSON into an array of splat chunk descriptors.
    -   **Edge Case**: If the array length is greater than 1, log a warning and proceed only with the first element (`chunks[0]`) [S1, unofficial].
    -   Extract camera metadata from the first chunk descriptor:
        -   Position: Array `[x, y, z]`.
        -   Rotation: 3x3 matrix.
    -   Convert the 3x3 rotation matrix to a Quaternion.
    -   Set the scene camera's position and rotation using these values [S1, unofficial].
4.  **Process 'splat' Chunk**:
    -   If the chunk type is `"splat"`, pass the raw buffer and the texture dimensions (`texwidth`, `texheight`) to the `SplatvData.Deserialize` method [S1, unofficial].
    -   Create a `Splat` object from the deserialized data.
    -   Add the `Splat` object to the scene [S1, unofficial].
5.  **Termination**: Stop after processing the relevant chunks. If no valid `splat` chunk is found, the scene remains empty of geometry.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_splatv(buffer: ArrayBuffer, texwidth: int, texheight: int) -> Scene:
    # Step 1: Magic Number Validation
    int_view = Int32Array(buffer)
    if int_view[0] != 0x674b:
        raise Error("Invalid splatv file: Magic number mismatch")

    # Step 2: Read first chunk header
    # Note: The source implies a specific structure where the first chunk 
    # is immediately accessible after the magic/size header or the magic 
    # is part of the first chunk header. Based on [S1, unofficial], we assume the 
    # buffer starts with the magic number which is the first Int32 of the 
    # first chunk header.
    
    # Read chunk size (bytes 4-7)
    chunk_size = int_view[1]
    
    # Read chunk type string starting at byte 8
    # We need to find the null terminator or use the size to determine length.
    # The source uses TextDecoder on the buffer, implying the chunk type 
    # is at the start of the data section or the header structure is:
    # [Magic][Size][TypeString][Data]
    
    # Re-interpreting [S1, unofficial] code:
    # const intView = new Int32Array(buffer.buffer);
    # This suggests the buffer contains the raw bytes.
    
    # Let's assume standard chunk layout:
    # Offset 0: Magic (Int32)
    # Offset 4: Size (Int32)
    # Offset 8: Type String (UTF-8, null-terminated or length-prefixed?)
    # The source says: "expect next chunk type 'chunks' with size intView[1]"
    # This implies the structure is: [Magic][Size][TypeString][Data]
    
    # Extract Type String
    # We need to decode bytes starting at offset 8.
    # Since we don't have the exact length of the string in the header 
    # (only the data size), we assume the string is null-terminated 
    # or we read until the end of the header block. 
    # However, [S1, unofficial] code snippet doesn't show explicit string parsing logic 
    # other than JSON parsing. We infer the type check from context.
    
    # For implementation completeness, we assume a helper to read the string:
    chunk_type = read_string(buffer, offset=8)
    
    if chunk_type == "chunks":
        # Step 3: Process chunks chunk
        data_start = 8 + len(chunk_type) + 1 # +1 for null terminator
        data_end = data_start + chunk_size
        chunk_data = buffer.slice(data_start, data_end)
        
        json_str = TextDecoder("utf-8").decode(chunk_data)
        splat_chunks = JSON.parse(json_str)
        
        if len(splat_chunks) > 1:
            console.warn("Splatv file contains more than one chunk, only the first one will be loaded")
        
        first_chunk = splat_chunks[0]
        
        # Extract Camera Data
        if "camera" in first_chunk and first_chunk["camera"]:
            cam = first_chunk["camera"]
            if "position" in cam and "rotation" in cam:
                pos = cam["position"] # [x, y, z]
                rot_matrix = cam["rotation"] # 3x3 matrix
                
                # Convert 3x3 matrix to Quaternion [G]
                q = matrix_to_quaternion(rot_matrix)
                
                # Set camera [S1, unofficial]
                scene.camera.position = Vector3(pos[0], pos[1], pos[2])
                scene.camera.rotation = q
                
    elif chunk_type == "splat":
        # Step 4: Process splat chunk
        data_start = 8 + len(chunk_type) + 1
        data_end = data_start + chunk_size
        splat_data = buffer.slice(data_start, data_end)
        
        # Deserialize [S1, unofficial]
        splat_data_obj = SplatvData.Deserialize(splat_data, texwidth, texheight)
        splat = Splat(splat_data_obj)
        scene.add(splat)
        
    else:
        # Unknown chunk type, skip or error
        pass

    return scene
```

**Parameters**
-   `texwidth`, `texheight`: Required by `SplatvData.Deserialize`. Not given in the sources. [G] Typically derived from the scene configuration or the splat data itself if embedded.

**Edge Cases and Pitfalls**
-   **Multiple Chunks**: If the `chunks` JSON array contains more than one element, only the first is loaded. This may result in incomplete scene data if the file is malformed or contains multiple distinct splat sets [S1, unofficial].
-   **Missing Camera Data**: If the `camera` object is missing or invalid in the JSON, the camera position and rotation are not updated. The scene will retain default camera settings [S1, unofficial].
-   **Invalid Magic**: The parser strictly checks for `0x674b`. Any other value causes immediate failure [S1, unofficial].

## Subtopics

### [4DGS v1 Progressive Loader](20182-4dgs-v1-progressive-loader.md)
The 4DGS v1 Progressive Loader is a streaming parser for volumetric splat files (`.4dsplat` v1) that loads independent Gaussian blocks per timestep.

## Operating characteristics
-   **Time Complexity**: Linear with respect to the size of the JSON metadata and the splat data, as it involves parsing JSON and deserializing binary data.
-   **Memory Complexity**: Proportional to the file size, as the entire buffer is loaded into memory before parsing.
-   **Strengths**: Combines geometry and camera data in a single file, simplifying asset management.
-   **Limitations**: Requires the entire file to be loaded into memory, which may be problematic for very large scenes compared to streaming parsers. The format is specific to the `gsplat.js` ecosystem and not a standard open format.
-   **When to use**: When using `gsplat.js` and needing to load a pre-converted `.splatv` file with embedded camera poses.
-   **When not to use**: For streaming large datasets or when interoperability with other renderers is required.

## Background
The Splatv format and its parser are part of the `gsplat.js` library, a JavaScript implementation of 3D Gaussian Splatting for the web [S1, unofficial]. The format was designed to facilitate the loading of Gaussian splat scenes in browser-based environments, bundling the necessary camera initialization data with the splat primitives. It is used within the `gsplat.js` scene graph to initialize scenes from exported `.splatv` files.

## Sources
[S1] huggingface/gsplat.js, https://github.com/huggingface/gsplat.js, fetched 2026-10-27 (unofficial)
