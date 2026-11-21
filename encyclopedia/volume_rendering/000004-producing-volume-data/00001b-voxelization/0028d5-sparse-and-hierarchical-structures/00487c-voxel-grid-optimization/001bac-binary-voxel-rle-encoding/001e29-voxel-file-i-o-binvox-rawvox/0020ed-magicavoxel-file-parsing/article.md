# MagicaVoxel File Parsing

Also called: .vox parser, .VOX File Parsing, MagicaVoxel .vox Loader, MagicaVoxel .vox Import and Coordinate Conversion, Magica Voxel format loader, MagicaVoxel import, VOX File Format Parsing, VOX File Format Importer

## Overview
A loader for MagicaVoxel `.vox` files (versions 150 and 200) that parses models, palette, MATL materials, and the nTRN/nGRP/nSHP scene graph, converting visible faces into quads within a BVH [S1, unofficial]. It structures data into chunks containing tags, sizes, and child chunks [S2, unofficial]. Unlike [Voxel File I/O (binvox/rawvox)](1698-voxel-file-i-o-binvox-rawvox.md) which uses dense or simple sparse grids, this method handles the hierarchical scene graph and chunk-based binary format specific to Magica Voxel [S2, unofficial].

## How it works
**Inputs and outputs:**
Input is a binary buffer containing a `.vox` file [S2, unofficial]. Output is a scene representation consisting of a palette, voxel data (coordinates and color indices), and a scene graph (transforms, groups, shapes) [S1, unofficial].

**Data structures:**
1. **Chunk Header:** `id` (4-byte ASCII tag), `contentSize` (4-byte integer, content length in bytes), `childrenSize` (4-byte integer, child chunk length in bytes), `end` (4-byte integer, absolute offset to the next chunk calculated as `current_pos + contentSize + childrenSize`) [S3, unofficial]. All integers are stored in little-endian order [S3, unofficial].
2. **Voxel Data (XYZI Chunk):** `numVoxels` (4-byte integer), followed by `numVoxels` voxels where each voxel is 4 bytes: `x` (uint8), `y` (uint8), `z` (uint8), `colorIndex` (uint8) [S2, unofficial][S3, unofficial].
3. **Palette (RGBA Chunk):** Array of 255 colors (indices 1–255), each 4 bytes (R, G, B, A uint8). Index 0 is reserved for the default palette [S2, unofficial][S4, unofficial].
4. **Scene Graph Nodes:** `nTRN` (transform node with translation/rotation, child is `nGRP` or `nSHP`), `nGRP` (group node, children are `nTRN`), `nSHP` (shape/model node, leaf node with no children) [S2, unofficial].

**Numbered steps:**
1. Read 4-byte magic ID. Verify it is `'VOX '` (0x564F5820) [S3, unofficial].
2. Read 4-byte version. Verify it is 150 (or 200 for extended features) [S1, unofficial][S3, unofficial].
3. Read the `MAIN` chunk [S3, unofficial].
4. Loop through sub-chunks until the end of the `MAIN` chunk is reached:
   a. Read chunk `id`, `contentSize`, `childrenSize` [S3, unofficial].
   b. Calculate `end` position: `current_pos + contentSize + childrenSize` [S3, unofficial].
   c. Process content based on `id`:
      i. If `SIZE`: Read `size.x`, `size.y`, `size.z` (3 integers) [S3, unofficial].
      ii. If `XYZI`: Read `numVoxels`. Loop `i` from 0 to `numVoxels`: Read `x`, `y`, `z`, `colorIndex` (uint8 each) [S3, unofficial].
      iii. If `RGBA`: Loop `i` from 0 to `contentSize/4`: Read `r`, `g`, `b`, `a` (uint8 each) [S4, unofficial].
      iv. If `nTRN`, `nGRP`, `nSHP`: Parse scene graph nodes recursively [S1, unofficial].
      v. If unknown: Skip `contentSize` bytes [S4, unofficial].
   d. Jump to next chunk at `end` [S3, unofficial].
5. If palette is empty, fill with MagicaVoxel default palette (256 colors) [S4, unofficial].
6. Convert voxel coordinates to internal representation. Index calculation for 3D voxel grid: `index = z * height * width + y * width + x`, where `width`, `height`, `depth` correspond to `x`, `y`, `z` dimensions respectively [S3, unofficial].
7. Convert from MagicaVoxel's z-up coordinate system (X=Left, Y=Back, Z=Up) to the renderer's y-up system if necessary [S1, unofficial][S4, unofficial].
8. Convert visible faces into quad primitives [S1, unofficial].
9. Place quads in the scene using instance transforms [S1, unofficial].
10. Merge glass voxels into a single volume representation [S1, unofficial].
11. Mark emissive voxels as light sources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_vox_file(buffer):
    # 1. Read header
    magic = buffer.read(4)
    if magic != b'VOX ':
        raise ValueError("Invalid magic number")
    
    version = buffer.read_uint32()
    if version not in (150, 200):
        raise ValueError(f"Unsupported version: {version}")
    
    # 2. Read MAIN chunk
    main_chunk = read_chunk(buffer)
    if main_chunk.id != b'MAIN':
        raise ValueError("Expected MAIN chunk")
    
    # Initialize palette (256 slots, index 0 reserved)
    palette = [None] * 256
    voxels = []
    size = None
    scene_graph = []
    
    # 3. Process sub-chunks
    current_pos = buffer.tell()
    while current_pos < main_chunk.end:
        chunk = read_chunk(buffer)
        
        if chunk.id == b'SIZE':
            size = (buffer.read_uint32(), buffer.read_uint32(), buffer.read_uint32())
        elif chunk.id == b'XYZI':
            num_voxels = buffer.read_uint32()
            for _ in range(num_voxels):
                x = buffer.read_uint8()
                y = buffer.read_uint8()
                z = buffer.read_uint8()
                color_index = buffer.read_uint8()
                voxels.append({'x': x, 'y': y, 'z': z, 'color_index': color_index})
        elif chunk.id == b'RGBA':
            num_colors = chunk.content_size // 4
            for i in range(num_colors):
                r = buffer.read_uint8()
                g = buffer.read_uint8()
                b = buffer.read_uint8()
                a = buffer.read_uint8()
                palette[i + 1] = (r, g, b, a)
        elif chunk.id in (b'nTRN', b'nGRP', b'nSHP'):
            scene_graph.append(parse_scene_chunk(chunk, buffer))
        else:
            buffer.seek(chunk.content_size, 1)
        
        buffer.seek(chunk.end)
        current_pos = buffer.tell()
    
    # 4. Handle default palette
    if not any(palette):
        palette = get_default_palette()
    
    # 5. Coordinate conversion (z-up to y-up)
    converted_voxels = []
    for v in voxels:
        converted_voxels.append({
            'x': v['x'],
            'y': v['z'],
            'z': v['y'],
            'color_index': v['color_index']
        })
    
    return {
        'size': size,
        'voxels': converted_voxels,
        'palette': palette,
        'scene_graph': scene_graph
    }

def read_chunk(buffer):
    id = buffer.read(4)
    content_size = buffer.read_uint32()
    children_size = buffer.read_uint32()
    end = buffer.tell() + content_size + children_size
    return {'id': id, 'content_size': content_size, 'children_size': children_size, 'end': end}

def parse_scene_chunk(chunk, buffer):
    # Recursively parse child chunks until chunk.end
    node = {'type': chunk.id, 'children': []}
    current_pos = buffer.tell()
    while current_pos < chunk.end:
        child = read_chunk(buffer)
        if child.id in (b'nTRN', b'nGRP', b'nSHP'):
            node['children'].append(parse_scene_chunk(child, buffer))
        else:
            buffer.seek(child.content_size, 1)
        buffer.seek(child.end)
        current_pos = buffer.tell()
    return node

def get_default_palette():
    # 256 colors, index 0 is default, 1-255 from file
    return [None] * 256
```

**Parameters:**
*   `version`: Expected file version number is 150 [S3, unofficial].
*   `bImportXForward`: Boolean flag to swap X and Y axes during import [S4, unofficial].
*   `bImportMaterial`: Boolean flag to enable material and texture generation from the voxel palette [S4, unofficial].
*   `bImportXYCenter`: Boolean flag to center the voxel grid on the XY plane [S4, unofficial].
*   `BuildScale3D`: Scale factors applied to each voxel box during collision/physics setup [S4, unofficial].

**Edge cases and pitfalls:**
*   **Coordinate Limit:** Standard `.vox` files limit coordinates to 0-255 per axis because each coordinate component uses 1 byte. This restricts the space to $\{(x_1, x_2, x_3) \in \mathbb{N}^3 \mid 0 \le x_i < 256\}$ [S2, unofficial].
*   **Invalid size:** If `size.x <= 0 || size.y <= 0 || size.z <= 0`, return error [S3, unofficial].
*   **Negative voxel count:** If `numVoxels < 0`, return error [S3, unofficial].
*   **Coordinate system mismatch:** MagicaVoxel uses X=Left, Y=Back, Z=Up. UE4 uses X=Forward, Y=Right, Z=Up. The importer handles this via `bImportXForward` flag which swaps axes and inverts X/Y during XYZI import [S4, unofficial].

## Subtopics

### MagicaVoxel File Output
*   **What it is:** Method to write `.vox` files from internal representations.
*   **What sets it apart:** It generates the chunk structure and binary layout required by MagicaVoxel.
*   **When to choose it:** When exporting voxel data from a custom renderer or tool to MagicaVoxel.
*   **Link:** [MagicaVoxel File Output](1752-magicavoxel-file-output.md)

### PLY File Parsing
*   **What it is:** Parser for PLY (Polygon File Format) files.
*   **What sets it apart:** PLY is a general-purpose mesh format, not voxel-specific. It handles triangles/quads, not voxel grids.
*   **When to choose it:** When importing mesh data that needs to be voxelized later.
*   **Link:** [PLY File Parsing](2781-ply-file-parsing.md)

### Voxel File I/O
*   **What it is:** General interface for reading/writing voxel data in various formats.
*   **What sets it apart:** Abstracts the specific format details (binvox, rawvox, vox) into a common API.
*   **When to choose it:** When building a generic voxel pipeline that supports multiple input formats.
*   **Link:** [Voxel File I/O](2786-voxel-file-i-o.md)

## Operating characteristics
*   **Time complexity:** Not given in the sources.
*   **Memory complexity:** Proportional to the number of voxels and the size of the palette (256 colors) [S2, unofficial].
*   **Measured performance:** Not given in the sources.
*   **Scaling:** Scales with the number of voxels, but limited by the 256x256x256 coordinate space per model [S2, unofficial].
*   **Strengths:** Handles complex scenes with transforms and groups [S1, unofficial]. Supports materials and emissive voxels [S1, unofficial].
*   **Limitations:** Coordinate limit of 256 per axis per model [S2, unofficial]. Version 200 features may not be fully supported in all parsers [S4, unofficial].
*   **When to use it:** When working with MagicaVoxel assets or needing to import voxel data with scene hierarchy.
*   **When not:** When working with dense voxel grids or formats like binvox that do not support scene graphs [S2, unofficial].

## Background
*   **Origin:** Developed for Magica Voxel by Ephtracy [S2, unofficial].
*   **History:** The format has evolved from version 150 to 200, adding support for materials and extended features [S1, unofficial][S4, unofficial].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S3] JamesRandall/SimpleVoxelEngine, https://github.com/JamesRandall/SimpleVoxelEngine, fetched 2026-10-26 (unofficial)
[S4] mik14a/VOX4U, https://github.com/mik14a/VOX4U, fetched 2026-10-26 (unofficial)
