# OpenVDB File Format Parsing

Also called: OpenVDB deserialization, VDB file reader

## Overview

This method describes the binary parsing and deserialization of the OpenVDB (VDB) file format to reconstruct the hierarchical sparse voxel grid in memory. It solves the problem of converting the compact, compressed on-disk representation into a traversable in-memory tree structure suitable for rendering or simulation. It is distinguished from generic VDB loading by its explicit handling of the specific binary layout of Node5 and Node4 hierarchies, including the complex logic for decompressing active masks and inactive values, as opposed to using high-level library abstractions which may hide these details.

## How it works

**Inputs and Outputs**
*   **Input**: A binary file stream conforming to the OpenVDB specification, identified by a magic number and file version.
*   **Output**: An in-memory representation of the grid, typically structured as a `GridDescriptor` containing a `RootNode` (or `Tree`), which holds the sparse voxel data.

**Data Structures**

1.  **GridDescriptor**:
    *   `name`: Length-prefixed string.
    *   `grid_type`: Length-prefixed string.
    *   `instance_parent`: Length-prefixed string.
    *   `grid_pos`: `u64` (little-endian).
    *   `block_pos`: `u64` (little-endian).
    *   `end_pos`: `u64` (little-endian).
    *   `compression`: Enum/Flags indicating compression type (e.g., `ACTIVE_MASK`, `BLOSC`).
    *   `meta_data`: A map of metadata entries.

2.  **Metadata**:
    *   Structure: `HashMap<String, MetadataValue>`.
    *   `MetadataValue` variants:
        *   `String`: Length-prefixed bytes.
        *   `Bool`: Single byte (`u8`).
        *   `I32`: 4-byte little-endian integer.
        *   `I64`: 8-byte little-endian integer.
        *   `Float`: 4-byte IEEE 754 float.
        *   `Vec3i`: Three 4-byte integers.
        *   `Unknown`: Name (string) + raw bytes (`Vec<u8>`).

3.  **NodeHeader** (for Node5 and Node4):
    *   `child_mask`: `BitVec<u64>` (LSB0), size $N::SIZE$ (e.g., 64 bits for Node5). Indicates which children exist.
    *   `value_mask`: `BitVec<u64>` (LSB0), size $N::SIZE$. Indicates which voxels/children have non-background values.
    *   `data`: `Vec<T>`. The compressed or raw value data.
    *   `log_2_dim`: `usize`. The logarithm of the node dimension.

4.  **RootNode**:
    *   `map`: `HashMap<RootKey, RootData>`.
    *   `background`: Value $T$ representing the default background value.
    *   `RootData` variants:
        *   `Tile<T>`: Contains `value` ($T$) and `active` (`bool`). Represents a uniform block of voxels.
        *   `Node(Box<Node5>)`: Points to a child internal node.

5.  **Node5**:
    *   Contains an array of `InternalData` (either `Tile<T>` or `Node(Box<Node4>)`).
    *   Indexed by the `child_mask`.

6.  **Node4**:
    *   Contains an array of `InternalData` (either `Tile<T>` or `Node(Box<Node3>)`).
    *   Indexed by the `child_mask`.

7.  **Node3**:
    *   Leaf node containing the actual voxel values.

**Numbered Steps**

1.  **Read Grid Descriptor**:
    *   Read `name`, `grid_type`, `instance_parent` as length-prefixed strings.
    *   Read `grid_pos`, `block_pos`, `end_pos` as `u64` little-endian integers.
    *   If file version $\ge$ `OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION`, read `compression` flag as `u32` little-endian; otherwise, use default (no compression).

2.  **Read Metadata**:
    *   Read metadata count as `u32` little-endian.
    *   For each entry:
        *   Read `name` and `type` as length-prefixed strings.
        *   Read `length` as `u32` little-endian.
        *   Parse value based on `type`:
            *   `"string"`: Read `length` bytes.
            *   `"bool"`: Read 1 byte.
            *   `"int32"`: Read 4 bytes, interpret as `i32`.
            *   `"int64"`: Read 8 bytes, interpret as `i64`.
            *   `"float"`: Read 4 bytes, interpret as `f32`.
            *   `"vec3i"`: Read 12 bytes, interpret as three `i32`.
            *   Unknown: Read `length` bytes into raw buffer.

3.  **Read Tree Topology**:
    *   Read `buffer_count` as `u32` little-endian. **Pitfall**: If `buffer_count != 1`, the format is unsupported (multi-buffer trees are not supported by this parser) [S1, unofficial].
    *   Read `root_node_background` as `u32` little-endian, converted to type $T$.
    *   Read `num_tiles` and `num_node5s` as `u32` little-endian.

4.  **Iterate Tiles**:
    *   For each tile:
        *   Read `origin` as `vec3i` (three `i32`).
        *   Compute `root_key` from `origin`.
        *   Read `value` as `u32` little-endian, converted to $T$.
        *   Read `active` flag as `u8` (1 = active, 0 = inactive).
        *   Store as `RootData::Tile<T>(value, active)`.

5.  **Iterate Node5s**:
    *   For each Node5:
        *   Read `origin` as `vec3i`.
        *   Compute `root_key`.
        *   Read internal node header:
            *   `child_mask`: `u64` bitvector (LSB0).
            *   `value_mask`: `u64` bitvector (LSB0).
            *   `compressed_data`: The raw bytes following the header.
        *   Construct `Node5`.
        *   For each active child index $i$ in `child_mask`:
            *   If the child is a Node4:
                *   Read `value_mask` for the Node4 as `u64` bitvector.
                *   Construct `Node3` leaf.
                *   Assign to `Node5` data array at index $i$.
            *   If the child is a Tile:
                *   Read tile data (value, active).
                *   Assign to `Node5` data array at index $i$.

6.  **Read Compressed Node Data**:
    *   For each internal node (Node5/Node4) with compressed data:
        *   Read `meta_data` byte if version $\ge$ `OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION` [S1, unofficial].
        *   Determine inactive values:
            *   If `meta_data` indicates `MaskAndOneInactiveVal` or `NoMaskAndOneInactiveVal`: Read one inactive value ($val_0$).
            *   If `meta_data` indicates `MaskAndTwoInactiveVals`: Read two inactive values ($val_0, val_1$).
            *   Otherwise: No inactive values read.
        *   Read `selection_mask` (`u64` bitvector) if `meta_data` indicates masked inactive values (`MaskAndNoInactiveVals`, `MaskAndOneInactiveVal`, or `MaskAndTwoInactiveVals`).
        *   Compute `active_count`:
            *   If `grid_descriptor.compression` contains `ACTIVE_MASK` and `meta_data != NoMaskAndAllVals` and version $\ge$ mask compression: `active_count = value_mask.count_ones()`.
            *   Else: `active_count = size` (where `size` is $2^{log\_2\_dim}$).
        *   Read compressed data array of size `active_count`.
        *   Handle type conversion:
            *   If file stores `f16` but target is `f32`: Convert `f16` to `f32`.
            *   If file stores `f32` but target is `f16`: Convert `f32` to `f16`.
            *   If types match: Use directly.
            *   **Pitfall**: Half-float storage logic may be incomplete; uncertainty exists regarding whether half-floats take 4 bytes in the file format [S1, unofficial].
        *   Expand active mask data if `ACTIVE_MASK` compression is used and `data.len() != size`:
            *   Initialize `expanded` array of size `size` with background value.
            *   Iterate `dest_idx` from 0 to `size - 1`:
                *   If `value_mask[dest_idx]` is true:
                    *   Assign `data[read_idx]` to `expanded[dest_idx]`.
                    *   Increment `read_idx`.
                *   Else if `selection_mask[dest_idx]` is true:
                    *   Assign `inactive_val1` to `expanded[dest_idx]`.
                *   Else:
                    *   Assign `inactive_val0` to `expanded[dest_idx]`.
            *   Return `expanded`.
        *   **Pitfall**: Blosc decompression fails if uncompressed size (`nbytes`) is 0, returning `UnsupportedBloscFormat` error [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def read_compressed(grid_descriptor, size, value_mask, reader):
    """
    Reads and decompresses node data for a Node5 or Node4.
    
    Args:
        grid_descriptor: The grid descriptor containing compression flags.
        size: The total number of elements in the node (e.g., 128 for Node5, 64 for Node4).
        value_mask: BitSlice<u64> indicating which elements are active/non-background.
        reader: A binary reader object with read_u8, read_u64_into, read_exact methods.
        
    Returns:
        Vec<T>: The decompressed array of values.
    """
    # Determine metadata
    meta_data = NodeMetaData::NoMaskAndAllVals
    if reader.header.file_version >= OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION:
        meta_data_byte = reader.read_u8()
        meta_data = try_into_node_meta_data(meta_data_byte)
        
    # Read inactive values
    inactive_val0 = T::zeroed()
    inactive_val1 = T::zeroed()
    
    if meta_data in [NodeMetaData::MaskAndOneInactiveVal, NodeMetaData::NoMaskAndOneInactiveVal]:
        inactive_val0 = reader.read_exact(bytes_of(inactive_val0))
    elif meta_data == NodeMetaData::MaskAndTwoInactiveVals:
        inactive_val0 = reader.read_exact(bytes_of(inactive_val0))
        inactive_val1 = reader.read_exact(bytes_of(inactive_val1))
        
    # Read selection mask
    selection_mask = bitvec![u64, Lsb0; 0; size]
    if meta_data in [NodeMetaData::MaskAndNoInactiveVals, NodeMetaData::MaskAndOneInactiveVal, NodeMetaData::MaskAndTwoInactiveVals]:
        reader.read_u64_into::<LittleEndian>(selection_mask.as_raw_mut_slice())
        
    # Compute active count
    if grid_descriptor.compression.contains(Compression::ACTIVE_MASK) and \
       meta_data != NodeMetaData::NoMaskAndAllVals and \
       reader.header.file_version >= OPENVDB_FILE_VERSION_NODE_MASK_COMPRESSION:
        count = value_mask.count_ones()
    else:
        count = size
        
    # Determine real storage type (f16 vs f32)
    expected_type = std::any::TypeId::of::<T>()
    f16_type = std::any::TypeId::of::<f16>()
    f32_type = std::any::TypeId::of::<f32>()
    real_type = f16_type if grid_descriptor.meta_data.is_half_float() else f32_type
    
    # Read compressed data
    if expected_type == real_type:
        data = read_compressed_data(grid_descriptor, count)
    elif expected_type == f16_type and real_type == f32_type:
        f32_data = read_compressed_data::<f32>(grid_descriptor, count)
        data = [f16::from_f32(v) for v in f32_data]
    elif expected_type == f32_type and real_type == f16_type:
        f16_data = read_compressed_data::<f16>(grid_descriptor, count)
        data = [f16::to_f32(v) for v in f16_data]
    elif real_type == f16_type:
        f16_data = read_compressed_data::<f16>(grid_descriptor, count)
        data = [T::from_f16_bites(v) for v in f16_data]
    elif real_type == f32_type:
        f32_data = read_compressed_data::<f32>(grid_descriptor, count)
        data = [T::from_f32_bites(v) for v in f32_data]
    else:
        raise UnreachableError()
        
    # Expand active mask if necessary
    if grid_descriptor.compression.contains(Compression::ACTIVE_MASK) and len(data) != size:
        expanded = [T::zeroed()] * size
        read_idx = 0
        for dest_idx in range(size):
            if value_mask[dest_idx]:
                expanded[dest_idx] = data[read_idx]
                read_idx += 1
            elif selection_mask[dest_idx]:
                expanded[dest_idx] = inactive_val1
            else:
                expanded[dest_idx] = inactive_val0
        return expanded
    else:
        return data
```

## Subtopics

### [OpenVDB Import via Python](7124-openvdb-import-via-python.md)
This method describes the process of converting seismic SEG-Y data into the OpenVDB format using Python, and subsequently importing that VDB file into a rendering engine (specifically Unreal Engine 5) for sparse voxel texture (SVT) rendering. It addresses the problem of handling massive datasets (e.g., 4.4 GB) that exceed standard 32-bit integer limits for memory addressing and buffer sizes, which causes import crashes and rendering artifacts. It is distinguished from generic VDB loading by its specific focus on the Python-side conversion pipeline and the subsequent engine-side integer overflow fixes required for large-scale sparse volumes.

## Operating characteristics

*   **Time Complexity**: Parsing is linear with respect to the number of nodes and tiles in the file, $O(N)$, where $N$ is the number of non-background elements. Decompression adds overhead proportional to the compressed data size.
*   **Memory Complexity**: The in-memory representation requires storing the full tree structure. For sparse data, this is significantly less than a dense grid, but the overhead of node headers and metadata adds to the memory footprint.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Performance scales with the sparsity of the data. Highly sparse data with many tiles will have faster parsing but potentially higher memory overhead due to tile structures.
*   **Strengths**: Efficient storage of sparse data; fast random access once loaded; supports various compression schemes.
*   **Limitations**: Multi-buffer trees are not supported [S1, unofficial]; half-float handling may be incomplete [S1, unofficial]; Blosc decompression can fail on empty data [S1, unofficial].
*   **When to use**: When working with large, sparse volumetric data that needs to be loaded into memory for rendering or simulation.
*   **When not to use**: When the data is dense and small enough to fit in a simple array; when multi-buffer support is required.

## Background

OpenVDB was developed by DreamWorks Animation for use in visual effects, particularly for fluid simulations and volumetric effects. The file format was designed to efficiently store large, sparse voxel grids. The parsing method described here is based on the implementation found in the `woxel` library [S1, unofficial].

## Sources

[S1] NemoInfo/woxel, https://github.com/NemoInfo/woxel, fetched 2026-10-24 (unofficial)
