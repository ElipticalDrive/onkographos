# PLY File Parsing

Also called: PLY Header Parsing

## Overview
This method parses the ASCII header of a Polygon File Format (PLY) file to determine vertex counts, property types, and the byte offset of the subsequent data section [S1, unofficial]. It serves as the initial step in extracting structured mesh geometry (vertices and faces) for voxelization pipelines, specifically targeting the ASCII format used in outputs from 3D Gaussian Splatting tutorials [S1, unofficial]. Unlike binary PLY parsers or OBJ loaders, this implementation assumes triangular topology and ignores quad faces, providing the raw data required for subsequent normal calculation and voxelization [S1, unofficial].

## How it works
**Inputs:** A binary buffer (`plyBuffer`) containing the raw bytes of an ASCII PLY file.
**Outputs:** An object containing:
- `vertexCount`: Integer number of vertices.
- `propertyTypes`: Object mapping property names to their data types (e.g., `{"x": "float", "y": "float"}`).
- `vertexData`: A `DataView` (or equivalent byte-access structure) pointing to the start of the vertex data section.
- `headerOffset`: The byte index where the header text begins (typically 0).

**Data Structures:**
- `headerText`: String accumulating the header content.
- `headerChunk`: Fixed-size byte array (50 bytes) for reading the buffer in chunks.
- `propertyTypes`: Dictionary/Object storing `property_name -> type_string` mappings.

**Steps:**
1. Initialize `headerOffset` to 0 and `headerText` to an empty string.
2. Loop to read the header:
   a. Read 50 bytes from `plyBuffer` starting at `headerOffset` into `headerChunk`.
   b. Decode `headerChunk` using UTF-8 (or ASCII) into a string and append to `headerText`.
   c. Increment `headerOffset` by 50.
   d. If `headerText` contains the string `end_header`, break the loop.
3. Split `headerText` by newline characters into a list of `headerLines`.
4. Iterate through `headerLines`:
   a. If a line starts with `element vertex`, extract the integer count using a regex pattern `\d+`. Store this as `vertexCount`.
   b. If a line starts with `property`, extract the type and name using the regex `(\w+)\s+(\w+)\s+(\w+)`. The first group is the property type (e.g., "float"), the second is "property" (literal), and the third is the property name (e.g., "x"). Store the mapping `name -> type` in `propertyTypes`.
5. Calculate `vertexByteOffset`:
   a. Find the index of the substring `end_header` in `headerText`.
   b. Add the length of the string `end_header` plus 1 (for the newline character) to this index.
   c. This value represents the byte offset in `plyBuffer` where the vertex data begins.
6. Create a `vertexData` view (e.g., `DataView`) starting at `vertexByteOffset` over the remainder of `plyBuffer`.
7. Return the object `{ vertexCount, propertyTypes, vertexData, headerOffset }`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_ply_header(ply_buffer: bytes) -> dict:
    header_offset = 0
    header_text = ""
    
    # Step 2: Read header in chunks until 'end_header' is found
    while True:
        # Read 50 bytes from ply_buffer at header_offset
        chunk = ply_buffer[header_offset : header_offset + 50]
        # Decode chunk and append
        header_text += chunk.decode('utf-8')
        header_offset += 50
        
        if 'end_header' in header_text:
            break
            
    # Step 3: Split into lines
    header_lines = header_text.split('\n')
    
    vertex_count = 0
    property_types = {}
    
    # Step 4: Parse lines
    for line in header_lines:
        if line.startswith('element vertex'):
            # Extract integer count
            import re
            match = re.search(r'\d+', line)
            if match:
                vertex_count = int(match.group())
        elif line.startswith('property'):
            # Extract type and name: "property <type> <name>"
            match = re.match(r'(\w+)\s+(\w+)\s+(\w+)', line)
            if match:
                prop_type = match.group(1)
                prop_name = match.group(3)
                property_types[prop_name] = prop_type
                
    # Step 5: Calculate byte offset
    end_header_index = header_text.find('end_header')
    # Add length of 'end_header' + 1 for newline
    vertex_byte_offset = end_header_index + len('end_header') + 1
    
    # Step 6: Create data view
    # Assuming a generic DataView-like structure accessible via offset
    vertex_data = DataView(ply_buffer, vertex_byte_offset)
    
    # Step 7: Return result
    return {
        "vertexCount": vertex_count,
        "propertyTypes": property_types,
        "vertexData": vertex_data,
        "headerOffset": header_offset
    }
```

**Edge Cases and Pitfalls:**
- **Binary PLY:** This method fails on binary PLY files because the header structure differs (it does not use `element vertex` text lines in the same way, and data is not ASCII). It is strictly for ASCII PLY [S1, unofficial].
- **Non-Triangular Meshes:** The method extracts vertex data but does not parse face indices. Downstream consumers must assume triangular topology or handle missing face data [S1, unofficial].
- **Precision:** Property types are stored as strings. The actual parsing of float/int values from `vertexData` depends on the endianness of the file, which is not handled in this header-parsing step [G].

## Subtopics

### [PLY Mesh Loader & Normal Calculator](2859-ply-mesh-loader-normal-calculator.md)
Parses ASCII PLY files to extract vertex coordinates and triangle face indices, computing per-face normals via cross products. It bridges raw mesh geometry and voxelization pipelines. Unlike this method, it handles the full data extraction and geometric computation, assuming triangular topology.

## Operating characteristics
- **Time Complexity:** Linear with respect to the header size, which is typically small and constant relative to the vertex count. The loop reads fixed 50-byte chunks, so it is $O(H)$ where $H$ is header length.
- **Memory Complexity:** $O(H)$ for storing the header text. The returned `vertexData` view is $O(1)$ as it references the original buffer.
- **Strengths:** Simple, robust parsing of ASCII headers; handles variable-length headers by chunking; provides immediate access to data offsets.
- **Limitations:** Does not parse binary PLY; does not extract face indices or normals; assumes ASCII encoding.
- **When to use:** When converting ASCII PLY files (e.g., from Gaussian Splatting tutorials) to other formats like `.splat` or for voxelization [S1, unofficial].
- **When not to use:** For binary PLY files or when face indices are required immediately without a separate parsing step.

## Background
This method originates from tools designed to convert PLY files generated by 3D Gaussian Splatting tutorials into formats suitable for WebGL viewers (e.g., `.splat`) [S1, unofficial]. It is a simplified version of Kevin Kwok's WebGL implementation of Gaussian Splatting (MIT License, Copyright (c) 2023 Kevin Kwok) [S1, unofficial]. The project is part of the Gaussian-Splatting-WebViewers repository, which uses A-Frame and Three.js for rendering [S1, unofficial].

## Sources
[S1] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
