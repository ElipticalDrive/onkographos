# OpenVDB File Format Parsing

Also called: OVDB Volume File Loading, OpenVDB deserialization, VDB file reader

## Overview
OpenVDB File Format Parsing is the method for reading and reconstructing sparse voxel grids from the binary OpenVDB file format [S1, unofficial]. It solves the problem of loading hierarchical, compressed sparse data structures into memory for rendering or simulation [G]. It is distinguished from generic volume loading by its specific handling of variable-length node types (Node3, Node4, Node5), active mask compression, and the strict little-endian binary layout [S1, unofficial][S3, unofficial].

## How it works
**Inputs:** A binary file stream conforming to the OpenVDB format [S1, unofficial].
**Outputs:** An in-memory representation of the grid, typically consisting of a `GridDescriptor`, a `Metadata` map, and a root node tree structure (`RootNode`) containing tiles and internal nodes [S1, unofficial].

### Data Structures and Layout
The file is read sequentially. All multi-byte integers are Little-Endian [S3, unofficial].

1. **GridDescriptor**:
   - `name`: Length-prefixed string.
   - `grid_type`: Length-prefixed string.
   - `instance_parent`: Length-prefixed string.
   - `grid_pos`: `u64`.
   - `block_pos`: `u64`.
   - `end_pos`: `u64`.
   - `compression`: Enum/Flags indicating compression types used (e.g., `ACTIVE_MASK`) [S1, unofficial].
   - `meta_data`: A map of key-value pairs [S1, unofficial].

2. **Metadata**:
   - Stored as a count (`u32`) followed by entries [S1, unofficial].
   - Each entry: `name` (length-prefixed string), `type` (length-prefixed string), `length` (`u32`) [S1, unofficial].
   - Value parsing depends on `type`:
     - `"string"`: Read `length` bytes as UTF-8.
     - `"bool"`: Read 1 byte.
     - `"int32"`: Read 4 bytes (`i32`).
     - `"int64"`: Read 8 bytes (`i64`).
     - `"float"`: Read 4 bytes (`f32`).
     - `"vec3i"`: Read 3 `i32` values.
     - Unknown types: Read `length` raw bytes [S1, unofficial].

3. **Tree Topology**:
   - `buffer_count`: `u32`. Must be 1. Multi-buffer trees are not supported [S1, unofficial].
   - `root_node_background`: `u32` interpreted as the background value of type `T` [S1, unofficial].
   - `num_tiles`: `u32`.
   - `num_node5s`: `u32` [S1, unofficial].

4. **Node Structures**:
   - **Tile**: Origin (`vec3i`), Value (`T`), Active flag (`u8`, 1=active) [S1, unofficial].
   - **Node5**:
     - Origin (`vec3i`).
     - `child_mask`: `u64` bitvector (LSB0), indicating which of the 32 children exist [S1, unofficial].
     - `value_mask`: `u64` bitvector (LSB0), indicating which children are active [S1, unofficial].
     - `data`: Compressed payload [S1, unofficial].
   - **Node4**:
     - Contains `value_mask` (`u64` bitvector) [S1, unofficial].
     - Contains 16 children, which are either Tiles or Node3s [S1, unofficial].
   - **Node3 (Leaf)**:
     - Contains 64 values [S1, unofficial].

### Parsing Steps
1. **Read Header and Descriptor**:
   - Read `GridDescriptor` fields as defined above [S1, unofficial].
   - If file version $\ge$ `OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION`, read `compression` flag as `u32` [S1, unofficial]. Otherwise, assume default (no compression) [G].

2. **Read Metadata**:
   - Read `meta_data_count` (`u32`) [S1, unofficial].
   - Loop `meta_data_count` times:
     - Read `name` and `type` strings [S1, unofficial].
     - Read `length` (`u32`) [S1, unofficial].
     - Parse value based on `type` [S1, unofficial].
     - Store in `HashMap<String, MetadataValue>` [S1, unofficial].

3. **Read Tree Topology**:
   - Read `buffer_count` (`u32`). Assert `buffer_count == 1` [S1, unofficial].
   - Read `root_node_background` (`u32`), cast to type `T` [S1, unofficial].
   - Read `num_tiles` (`u32`) and `num_node5s` (`u32`) [S1, unofficial].

4. **Iterate Tiles**:
   - Loop `num_tiles` times:
     - Read `origin` (`vec3i`) [S1, unofficial].
     - Compute `root_key` from `origin` [G].
     - Read `value` (`u32`), cast to `T` [S1, unofficial].
     - Read `active` (`u8`) [S1, unofficial].
     - Store as `RootData::Tile(value, active)` [S1, unofficial].

5. **Iterate Node5s**:
   - Loop `num_node5s` times:
     - Read `origin` (`vec3i`) [S1, unofficial].
     - Compute `root_key` [G].
     - Read `child_mask` (`u64`) and `value_mask` (`u64`) [S1, unofficial].
     - Read compressed `data` for the Node5 [S1, unofficial].
     - Construct `Node5` with `child_mask`, `value_mask`, and `data` [S1, unofficial].
     - For each child index `i` where `child_mask[i]` is set:
       - Read `value_mask` (`u64`) for the child Node4 [S1, unofficial].
       - Construct `Node3` leaf [S1, unofficial].
       - Assign to `Node5`'s data array at index `i` [S1, unofficial].

6. **Read Compressed Node Data**:
   - For each node (Node5 or Node4) with compressed data:
     - If version $\ge$ `OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION`, read `meta_data` byte (`u8`) [S1, unofficial].
     - Determine inactive values based on `meta_data`:
       - `NoMaskAndAllVals`: No inactive values stored.
       - `NoMaskAndOneInactiveVal`: Read one inactive value (`T`) [S1, unofficial].
       - `MaskAndOneInactiveVal`: Read one inactive value (`T`) [S1, unofficial].
       - `MaskAndTwoInactiveVals`: Read two inactive values (`T`, `T`) [S1, unofficial].
     - If `meta_data` indicates a selection mask (`MaskAnd...`), read `selection_mask` (`u64`) [S1, unofficial].
     - Calculate `active_count`:
       - If `ACTIVE_MASK` compression is enabled and `meta_data` is not `NoMaskAndAllVals`, `active_count = value_mask.count_ones()` [S1, unofficial].
       - Otherwise, `active_count = node_size` (e.g., 64 for Node3, 16 for Node4, 32 for Node5) [S1, unofficial].
     - Read `active_count` values of type `T` (or `f32`/`f16` depending on storage) [S1, unofficial].
     - Handle type conversion (e.g., `f32` to `f16` or vice versa) if metadata indicates half-float storage [S1, unofficial].
     - Expand the data array to full node size if `ACTIVE_MASK` is used:
       - Initialize `expanded` array of size `node_size` with background or inactive values [S1, unofficial].
       - Iterate `dest_idx` from 0 to `node_size - 1`:
         - If `value_mask[dest_idx]` is true:
           - Assign next value from compressed data array [S1, unofficial].
         - Else if `selection_mask[dest_idx]` is true:
           - Assign `inactive_val1` [S1, unofficial].
         - Else:
           - Assign `inactive_val0` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_openvdb_file(file_stream):
    """
    Parses an OpenVDB file into a Grid structure.
    Assumes little-endian byte order.
    """
    # Helper to read little-endian integers
    def read_u32(): return int.from_bytes(file_stream.read(4), 'little')
    def read_u64(): return int.from_bytes(file_stream.read(8), 'little')
    def read_u8(): return int.from_bytes(file_stream.read(1), 'little')
    def read_i32(): return int.from_bytes(file_stream.read(4), 'little', signed=True)
    def read_f32(): import struct; return struct.unpack('<f', file_stream.read(4))[0]
    def read_length_prefixed_string():
        length = read_u32()
        return file_stream.read(length).decode('utf-8')

    # 1. Read Grid Descriptor
    grid_descriptor = {
        'name': read_length_prefixed_string(),
        'grid_type': read_length_prefixed_string(),
        'instance_parent': read_length_prefixed_string(),
        'grid_pos': read_u64(),
        'block_pos': read_u64(),
        'end_pos': read_u64()
    }
    
    file_version = read_u32()
    OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION = 10 # Threshold constant from spec
    if file_version >= OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION:
        grid_descriptor['compression'] = read_u32()
    else:
        grid_descriptor['compression'] = 0

    # 2. Read Metadata
    meta_data_count = read_u32()
    grid_descriptor['meta_data'] = {}
    for _ in range(meta_data_count):
        name = read_length_prefixed_string()
        type_str = read_length_prefixed_string()
        length = read_u32()
        
        value = None
        if type_str == "string":
            value = file_stream.read(length).decode('utf-8')
        elif type_str == "bool":
            value = read_u8() != 0
        elif type_str == "int32":
            value = read_i32()
        elif type_str == "int64":
            value = int.from_bytes(file_stream.read(8), 'little', signed=True)
        elif type_str == "float":
            value = read_f32()
        elif type_str == "vec3i":
            value = (read_i32(), read_i32(), read_i32())
        else:
            value = file_stream.read(length)
            
        grid_descriptor['meta_data'][name] = value

    # 3. Read Tree Topology
    buffer_count = read_u32()
    if buffer_count != 1:
        raise ValueError("Multi-buffer trees are not supported")
        
    root_background_raw = read_u32()
    # Assume type T is known from metadata or defaults to float
    root_background = float(root_background_raw) 
    
    num_tiles = read_u32()
    num_node5s = read_u32()
    
    root_node = {'background': root_background, 'tiles': {}, 'node5s': {}}

    # 4. Iterate Tiles
    for _ in range(num_tiles):
        origin = (read_i32(), read_i32(), read_i32())
        root_key = compute_root_key(origin)
        value_raw = read_u32()
        value = float(value_raw)
        active = read_u8() == 1
        
        root_node['tiles'][root_key] = {'value': value, 'active': active}

    # 5. Iterate Node5s
    for _ in range(num_node5s):
        origin = (read_i32(), read_i32(), read_i32())
        root_key = compute_root_key(origin)
        child_mask = read_u64()
        value_mask = read_u64()
        
        compressed_data = read_compressed_node_data(grid_descriptor, node_size=32, value_mask=value_mask)
        
        node5 = {'child_mask': child_mask, 'value_mask': value_mask, 'data': compressed_data, 'children': {}}
        
        for i in range(32):
            if (child_mask >> i) & 1:
                child_value_mask = read_u64()
                leaf_data = read_compressed_node_data(grid_descriptor, node_size=64, value_mask=child_value_mask)
                node3 = {'data': leaf_data}
                node5['children'][i] = node3 
                
        root_node['node5s'][root_key] = node5

    return {'grid_descriptor': grid_descriptor, 'root_node': root_node}

def read_compressed_node_data(grid_descriptor, node_size, value_mask):
    meta_data_byte = 0
    if grid_descriptor.get('file_version', 0) >= OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION:
        meta_data_byte = read_u8()
        
    inactive_val0 = 0.0
    inactive_val1 = 0.0
    
    # NodeMetaData enum mapping: 0=NoMaskAndAllVals, 1=MaskAndOneInactiveVal, 2=NoMaskAndOneInactiveVal, 3=MaskAndTwoInactiveVals, 4=MaskAndNoInactiveVals
    if meta_data_byte in [1, 2]: 
        inactive_val0 = read_f32()
    elif meta_data_byte == 3: 
        inactive_val0 = read_f32()
        inactive_val1 = read_f32()
        
    selection_mask = 0
    if meta_data_byte in [0, 1, 3, 4]: 
        selection_mask = read_u64()
        
    active_count = node_size
    if grid_descriptor['compression'] & 0x1 and meta_data_byte != 0: # ACTIVE_MASK flag
        active_count = bin(value_mask).count('1')
        
    raw_data = []
    for _ in range(active_count):
        raw_data.append(read_f32())
        
    if grid_descriptor['compression'] & 0x1 and len(raw_data) != node_size:
        expanded = [0.0] * node_size
        read_idx = 0
        for dest_idx in range(node_size):
            if (value_mask >> dest_idx) & 1:
                expanded[dest_idx] = raw_data[read_idx]
                read_idx += 1
            elif (selection_mask >> dest_idx) & 1:
                expanded[dest_idx] = inactive_val1
            else:
                expanded[dest_idx] = inactive_val0
        return expanded
    else:
        return raw_data
```

### Parameters and Edge Cases
*   **File Version**: Determines whether compression flags and metadata bytes are present. Version `OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION` is the critical threshold [S1, unofficial].
*   **Compression Flags**: `ACTIVE_MASK` indicates that only active values are stored in the compressed data array, requiring expansion using `value_mask` and `selection_mask` [S1, unofficial].
*   **Half-Float Storage**: The sources note uncertainty about whether half-floats take 4 bytes in the file format, suggesting implementation-specific handling or a TODO [S1, unofficial].
*   **Blosc Decompression**: If the uncompressed size (`nbytes`) is 0, Blosc decompression fails with `UnsupportedBloscFormat` [S1, unofficial].
*   **Multi-Buffer**: The reader explicitly panics or errors if `buffer_count != 1` [S1, unofficial].

## Subtopics

### [OpenVDB Import via Python](7124-openvdb-import-via-python.md)
This method converts seismic SEG-Y data into the OpenVDB format using Python, and imports the resulting VDB file into a rendering engine (specifically Unreal Engine 5) for sparse voxel texture (SVT) rendering. It addresses handling massive datasets (e.g., 4.4 GB) that exceed standard 32-bit integer limits for memory addressing and buffer sizes, which causes import crashes and rendering artifacts. It is distinguished from generic VDB loading by its specific focus on the Python-side conversion pipeline and the subsequent engine-side integer overflow fixes required for large-scale sparse volumes.

### [OpenVDB Volume Data Source](14158-openvdb-volume-data-source.md)
The OpenVDB Volume Data Source is a plugin for the Mitsuba renderer that enables ray marching through volumetric data stored in the OpenVDB sparse voxel format. It implements the `VolumeDataSource` interface, providing scalar field lookups via trilinear interpolation and automatic bounding box calculation. It is distinguished from generic import methods by its integration into the rendering pipeline's caching system and its specific handling of OpenVDB's coordinate transforms and metadata.

### [OpenVDB Field Writer](16391-openvdb-field-writer.md)
The OpenVDB Field Writer is a Python-based serialization technique that converts 3D regular volumetric grids into the OpenVDB sparse tree format for export to VFX software like Blender. It differs from dense grid export by leveraging OpenVDB's sparse tree structure to efficiently store volumetric data and prune background values. It builds on [OpenVDB File Format Parsing](5173-openvdb-file-format-parsing.md) by providing the high-level API to construct and persist the grid structure.

## Operating characteristics
*   **Time Complexity**: Parsing is linear with respect to the number of nodes and tiles in the file, plus the cost of decompression (Blosc). Decompression is typically fast but depends on the compression ratio.
*   **Memory Complexity**: The in-memory representation is proportional to the number of non-background nodes and tiles. Sparse structures allow efficient storage of large grids with low occupancy.
*   **Performance**: Reading is generally fast due to the compact binary format and efficient compression. However, decompression overhead can be significant for highly compressed data.
*   **Strengths**: Efficient storage of sparse data, fast random access to nodes after loading, support for various data types and compression methods.
*   **Limitations**: Multi-buffer trees are not supported. Half-float handling may be implementation-specific.

## Background
OpenVDB is an open-source C++ library developed by DreamWorks Animation for the storage and manipulation of large volumetric datasets, particularly for visual effects. The file format is designed to be compact and efficient for sparse data. The parsing method described here is based on the implementation details found in the `woxel` library [S1, unofficial].

## Sources
[S1] NemoInfo/woxel, https://github.com/NemoInfo/woxel, fetched 2026-10-24 (unofficial)
[S2] RickyMexx/yocto-gl-vpt, https://github.com/RickyMexx/yocto-gl-vpt, fetched 2026-10-26 (unofficial)
[S3] JangaFX - Insight: VDB, a deep dive, https://jangafx.com/insights/vdb-a-deep-dive, fetched 2026-10-27 (unofficial)
