# PLY File Parsing

Also called: PLY Header Parsing

## Overview
This method parses the ASCII header of a PLY file to determine vertex counts, property types, and the byte offset of the subsequent data section [S1, unofficial]. It serves as the initial step for extracting structured data from ASCII PLY files [S1, unofficial].

## How it works
**Inputs:** A binary buffer (`plyBuffer`) containing the raw bytes of an ASCII PLY file.
**Outputs:** An object containing:
- `vertexCount`: Integer number of vertices.
- `propertyTypes`: Object mapping property names to their data types.
- `vertexData`: A byte-access structure pointing to the start of the vertex data section.
- `headerOffset`: The byte index where the header text begins.

**Data Structures:**
- `headerText`: String accumulating the header content.
- `headerChunk`: Fixed-size byte array (50 bytes) for reading the buffer in chunks.
- `propertyTypes`: Dictionary storing `property_name -> type_string` mappings.

**Steps:**
1. Initialize `headerOffset` to 0 and `headerText` to an empty string [S1, unofficial].
2. Loop to read the header:
   a. Read 50 bytes from `plyBuffer` at `headerOffset` into `headerChunk` [S1, unofficial].
   b. Decode `headerChunk` and append to `headerText` [S1, unofficial].
   c. Increment `headerOffset` by 50 [S1, unofficial].
   d. If `headerText` includes `end_header`, break the loop [S1, unofficial].
3. Split `headerText` by newline into `headerLines` [S1, unofficial].
4. Iterate through `headerLines`:
   a. If a line starts with `element vertex`, extract the integer count using regex `\d+` [S1, unofficial].
   b. If a line starts with `property`, extract type and name using regex `(\w+)\s+(\w+)\s+(\w+)` [S1, unofficial].
   c. Store the mapping `name -> type` in `propertyTypes` [S1, unofficial].
5. Calculate `vertexByteOffset` as the index of `end_header` plus the length of `end_header` plus 1 [S1, unofficial].
6. Create a `vertexData` view starting at `vertexByteOffset` over the rest of `plyBuffer` [S1, unofficial].
7. Return the object `{ vertexCount, propertyTypes, vertexData, headerOffset }` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_ply_header(ply_buffer: bytes) -> dict:
    header_offset = 0
    header_text = ""
    
    while True:
        chunk = ply_buffer[header_offset : header_offset + 50]
        header_text += chunk.decode('utf-8')
        header_offset += 50
        if 'end_header' in header_text:
            break
            
    header_lines = header_text.split('\n')
    
    vertex_count = 0
    property_types = {}
    
    for line in header_lines:
        if line.startswith('element vertex'):
            import re
            match = re.search(r'\d+', line)
            if match:
                vertex_count = int(match.group())
        elif line.startswith('property'):
            match = re.match(r'(\w+)\s+(\w+)\s+(\w+)', line)
            if match:
                prop_type = match.group(1)
                prop_name = match.group(3)
                property_types[prop_name] = prop_type
                
    end_header_index = header_text.find('end_header')
    vertex_byte_offset = end_header_index + len('end_header') + 1
    
    vertex_data = SimpleDataView(ply_buffer, vertex_byte_offset)
    
    return {
        "vertexCount": vertex_count,
        "propertyTypes": property_types,
        "vertexData": vertex_data,
        "headerOffset": header_offset
    }

class SimpleDataView:
    def __init__(self, buffer: bytes, offset: int):
        self.buffer = buffer
        self.offset = offset
    def read_float(self, index: int) -> float:
        return float.frombuffer(self.buffer, self.offset + index * 4)
```

**Edge Cases and Pitfalls:**
- The method assumes ASCII encoding and does not handle binary PLY headers [G].
- Property type strings must be mapped to actual byte sizes by the caller before reading `vertexData` [G].

## Subtopics

### PLY Format Detection
Detects PLY format by checking header elements [S2, unofficial]. If the `vertex` element exists and the first element is `chunk`, it uses `SuperSplatCompressed` [S2, unofficial]. Else if `vertex` exists, it uses `Ply` [S2, unofficial]. Else it raises an error [S2, unofficial].

### [PLY Mesh Loader & Normal Calculator](2859-ply-mesh-loader-normal-calculator.md)
Parses ASCII PLY files to extract vertex coordinates and face indices, converting them into flat float arrays and triangle lists while computing face normals [G]. It solves the problem of loading raw mesh geometry for subsequent voxelization or rendering [G]. It differs from its parent [PLY File Parsing](2781-ply-file-parsing.md) by handling the actual data extraction and geometric computation rather than just header structure, and from its child [OBJ Mesh Loader & Triangulator](12664-obj-mesh-loader-triangulator.md) by relying on the PLY format's explicit face definition rather than parsing token-based text and performing polygon-to-triangle conversion [G].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] akbartus/Gaussian-Splatting-WebViewers, https://github.com/akbartus/Gaussian-Splatting-WebViewers, fetched 2026-10-27 (unofficial)
[S2] Brush – A new compatible Gaussian splatting engine, https://github.com/ArthurBrussee/brush, fetched 2026-10-26 (unofficial)
