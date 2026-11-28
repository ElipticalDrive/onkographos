# VDB 3-4-5 Tree Traversal

Also called: OpenVDB 3-4-5 Tree, VDB345

## Overview
The VDB 3-4-5 Tree Traversal is a method for retrieving leaf node data from an OpenVDB file stored in a compressed texture atlas format using a hierarchical 3-4-5 tree topology [S1, unofficial]. It solves the problem of random access to sparse volumetric data by mapping 3D coordinates to leaf values through a fixed-depth traversal of nodes with sizes $3^3$, $4^4$, and $5^5$ [S1, unofficial]. It differs from standard pointer-based OpenVDB by storing the tree structure in compressed texture atlases rather than explicit pointers, and from NanoVDB by using a specific 3-4-5 hierarchy instead of a linearized or standard octree representation [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- `pos`: The 3D voxel coordinates to query, represented as `vec3<f32>` or `vec3<i32>` [S1, unofficial].
- `leaff`: An output structure of type `VdbLeaf` [S1, unofficial].
- Internal data structures:
  - `origins`: Array of node origins [S1, unofficial].
  - `node5s`, `node4s`, `node3s`: Texture atlases containing node data [S1, unofficial].
  - `kids5`, `vals5`, `kids4`, `vals4`, `vals3`: Mask arrays for child and value existence [S1, unofficial].
  - `NODE5_TOTAL_LOG_D`, `NODE4_TOTAL_LOG_D`, `NODE3_TOTAL_LOG_D`: Precomputed logarithmic dimensions [S1, unofficial].

**Outputs:**
- `leaff`: A `VdbLeaf` struct containing the retrieved color, distance, number of parents, and parent pointers [S1, unofficial].

### Data Structures

**VdbLeaf**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
struct VdbLeaf {
    color: vec3<f32>  // Retrieved voxel value or default
    dist: u32         // Distance or level indicator
    num_parents: u32  // Number of valid parent pointers (0-3)
    parents: array<Parent, 3> // Parent nodes in traversal order
}
```

**Parent**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
struct Parent {
    origin: vec3<i32> // Global coordinate of the parent node
    idx: u32          // Index within the parent's texture atlas
}
```

**ArchiveHeader** (File Header)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
struct ArchiveHeader {
    file_version: u32
    library_major: u32
    library_minor: u32
    uuid: String
    has_grid_offsets: bool
    compression: Compression // ZIP or Blosc
    grid_number: u32
    meta_data: Metadata
}
```

**GridDescriptor**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```text
struct GridDescriptor {
    name: String
    grid_type: String
    instance_parent: String
    grid_pos: u64
    block_pos: u64
    end_pos: u64
    compression: Compression
    meta_data: Metadata
}
```

### Parameters
- `NODE5_TOTAL_LOG_D`: $u32 = 12$ (Approximation for $5^5=3125$, used for coordinate shifting) [S1, unofficial].
- `NODE4_TOTAL_LOG_D`: $u32 = 7$ ($\log_2(4^4) = \log_2(256) = 8$, source states 7) [S1, unofficial].
- `NODE3_TOTAL_LOG_D`: $u32 = 3$ ($\log_2(3^3) = \log_2(27) \approx 4.75$, source states 3) [S1, unofficial].
- `OPENVDB_MIN_SUPPORTED_VERSION`: $u32 = 218$ [S1, unofficial].
- `MAGIC_BYTES`: $u32 = 0x56444220$ [S1, unofficial].

### Steps

1.  **File Initialization**:
    - Read the first 4 bytes. If not equal to `MAGIC_BYTES` ($0x56444220$), return `MagicMismatch` error [S1, unofficial].
    - Read `file_version`. If `file_version` < `OPENVDB_MIN_SUPPORTED_VERSION` (218), return `UnsupportedVersion` error [S1, unofficial].
    - Read `library_major`, `library_minor`, `uuid`, `has_grid_offsets`, `compression`, `grid_number`, and `meta_data` into `ArchiveHeader` [S1, unofficial].
    - Read `GridDescriptor` for the target grid, including `grid_pos`, `block_pos`, `end_pos`, and `compression` [S1, unofficial].
    - Decompress the grid data based on the `compression` type (ZIP or Blosc) [S1, unofficial].

2.  **Traversal Entry**:
    - Call `get_vdb_leaf_from_leaf(pos, leaff)` [S1, unofficial].

3.  **Leaf Check (`get_vdb_leaf_from_leaf`)**:
    - Check `leaff.num_parents` [S1, unofficial].
    - If `num_parents` == 3:
        - Check if `parents[2]` matches `node3_global` (derived from `pos`) [S1, unofficial].
        - If match, proceed to Node 3 logic. If no match, call `get_vdb_leaf_from_nothing` [S1, unofficial].
    - If `num_parents` == 2:
        - Check if `parents[1]` matches `node4_global` [S1, unofficial].
        - If match, proceed to Node 4 logic. If no match, call `get_vdb_leaf_from_nothing` [S1, unofficial].
    - If `num_parents` == 1:
        - Check if `parents[0]` matches `node5_global` [S1, unofficial].
        - If match, proceed to Node 5 logic. If no match, call `get_vdb_leaf_from_nothing` [S1, unofficial].
    - If `num_parents` == 0:
        - Call `get_vdb_leaf_from_nothing` [S1, unofficial].

4.  **Root Lookup (`get_vdb_leaf_from_nothing`)**:
    - Iterate through the `origins` array [S1, unofficial].
    - If `node5_global` matches `origins[node5_idx]`:
        - Set `leaf.parents[0] = Parent(node5_global, node5_idx)` [S1, unofficial].
        - Set `leaf.num_parents = 1` [S1, unofficial].
        - Call `get_vdb_leaf_from_node5` [S1, unofficial].
    - If no match is found after iterating all origins:
        - Return `VdbLeaf(vec3<f32>(0.0), 1u, 0u, leaf.parents)` [S1, unofficial].

5.  **Node 5 Traversal (`get_vdb_leaf_from_node5`)**:
    - Calculate `node5_local`, `node5_child`, and `node5_offset` from `pos` and `node5_global` [S1, unofficial].
    - Check masks in `kids5` and `vals5` arrays using bit operations based on `node5_child` [S1, unofficial].
    - Load `node4_idx` from the `node5s` texture atlas [S1, unofficial].
    - If `in_val5` is true:
        - Return `VdbLeaf(vec3<f32>(0.2), 0u, 1u, leaf.parents)` [S1, unofficial].
    - If `!in_kid5` is true:
        - Return `VdbLeaf(vec3<f32>(0.0), node4_idx, 1u, leaf.parents)` [S1, unofficial].
    - Else:
        - Set `leaf.parents[1] = Parent(node4_global, node4_idx)` [S1, unofficial].
        - Set `leaf.num_parents = 2` [S1, unofficial].
        - Call `get_vdb_leaf_from_node4` [S1, unofficial].

6.  **Node 4 Traversal (`get_vdb_leaf_from_node4`)**:
    - Calculate `node4_local`, `node4_child`, and `node4_offset` from `pos` and `node4_global` [S1, unofficial].
    - Check masks in `kids4` and `vals4` arrays using bit operations based on `node4_child` [S1, unofficial].
    - Load `node3_idx` from the `node4s` texture atlas [S1, unofficial].
    - If `in_val4` is true:
        - Return `VdbLeaf(vec3<f32>(0.2), 0u, 2u, leaf.parents)` [S1, unofficial].
    - If `!in_kid4` is true:
        - Return `VdbLeaf(vec3<f32>(0.0), node3_idx, 2u, leaf.parents)` [S1, unofficial].
    - Else:
        - Set `leaf.parents[2] = Parent(node3_global, node3_idx)` [S1, unofficial].
        - Set `leaf.num_parents = 3` [S1, unofficial].
        - Call `get_vdb_leaf_from_node3` [S1, unofficial].

7.  **Node 3 Traversal (`get_vdb_leaf_from_node3`)**:
    - Calculate `node3_local` and `node3_offset` from `pos` and `node3_global` [S1, unofficial].
    - Check mask in `vals3` array based on `node3_local` [S1, unofficial].
    - Load voxel value from the `node3s` texture atlas [S1, unofficial].
    - If `in_val3` is true:
        - Return `VdbLeaf(vec3<f32>(0.1), 0u, 3u, leaf.parents)` [S1, unofficial].
    - Else:
        - Return `VdbLeaf(vec3<f32>(0.0), voxel, 3u, leaf.parents)` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAGIC_BYTES = 0x56444220
OPENVDB_MIN_SUPPORTED_VERSION = 218
NODE5_TOTAL_LOG_D = 12
NODE4_TOTAL_LOG_D = 7
NODE3_TOTAL_LOG_D = 3

# Data Structures
class Parent:
    def __init__(self, origin, idx):
        self.origin = origin
        self.idx = idx

class VdbLeaf:
    def __init__(self, color, dist, num_parents, parents):
        self.color = color
        self.dist = dist
        self.num_parents = num_parents
        self.parents = parents

# Global Data (Loaded from file)
origins = [] # Array of vec3<i32>
node5s = []  # Texture atlas for Node 5
node4s = []  # Texture atlas for Node 4
node3s = []  # Texture atlas for Node 3
kids5 = []   # Bitmask array
vals5 = []   # Bitmask array
kids4 = []   # Bitmask array
vals4 = []   # Bitmask array
vals3 = []   # Bitmask array

def read_vdb_file(filepath):
    """Initialize VDB reader and validate header."""
    # 1. Read magic bytes
    magic = read_u32(filepath, 0)
    if magic != MAGIC_BYTES:
        raise Error("MagicMismatch")
    
    # 2. Read version
    file_version = read_u32(filepath, 4)
    if file_version < OPENVDB_MIN_SUPPORTED_VERSION:
        raise Error("UnsupportedVersion")
    
    # 3. Read ArchiveHeader
    library_major = read_u32(filepath, 8)
    library_minor = read_u32(filepath, 12)
    uuid = read_string(filepath, 16)
    has_grid_offsets = read_bool(filepath, 16 + len(uuid))
    compression = read_compression(filepath, ...)
    grid_number = read_u32(filepath, ...)
    meta_data = read_metadata(filepath, ...)
    
    # 4. Read GridDescriptor
    grid_pos = read_u64(filepath, ...)
    block_pos = read_u64(filepath, ...)
    end_pos = read_u64(filepath, ...)
    
    # 5. Decompress and load data structures
    # This step involves decompressing the grid data based on 'compression'
    # and populating 'origins', 'node5s', 'node4s', 'node3s', 'kids5', etc.
    # Implementation details depend on the specific compression library (ZIP/Blosc)
    decompress_and_load_grid_data(grid_pos, block_pos, end_pos, compression)
    
    return True

def get_vdb_leaf_from_leaf(pos, leaff):
    """Entry point for traversal."""
    # Determine global node coordinates for each level
    node5_global = get_global_node5(pos)
    node4_global = get_global_node4(pos)
    node3_global = get_global_node3(pos)
    
    if leaff.num_parents == 3:
        if leaff.parents[2].origin == node3_global:
            return get_vdb_leaf_from_node3(pos, leaff)
        else:
            return get_vdb_leaf_from_nothing(pos, leaff)
    elif leaff.num_parents == 2:
        if leaff.parents[1].origin == node4_global:
            return get_vdb_leaf_from_node4(pos, leaff)
        else:
            return get_vdb_leaf_from_nothing(pos, leaff)
    elif leaff.num_parents == 1:
        if leaff.parents[0].origin == node5_global:
            return get_vdb_leaf_from_node5(pos, leaff)
        else:
            return get_vdb_leaf_from_nothing(pos, leaff)
    else:
        return get_vdb_leaf_from_nothing(pos, leaff)

def get_vdb_leaf_from_nothing(pos, leaff):
    """Start traversal from root if no parents match."""
    for node5_idx in range(len(origins)):
        if origins[node5_idx] == get_global_node5(pos):
            leaff.parents[0] = Parent(get_global_node5(pos), node5_idx)
            leaff.num_parents = 1
            return get_vdb_leaf_from_node5(pos, leaff)
    
    # No match found
    return VdbLeaf(vec3(0.0, 0.0, 0.0), 1, 0, leaff.parents)

def get_vdb_leaf_from_node5(pos, leaff):
    """Traverse Node 5 level."""
    node5_global = leaff.parents[0].origin
    node5_idx = leaff.parents[0].idx
    
    # Calculate local coordinates within Node 5
    node5_local = pos - node5_global * 5 # Assuming 5^5 block size, logic simplified
    node5_child = get_child_index(node5_local, 5) # Specific bit packing logic
    
    # Check masks
    in_val5 = (vals5[node5_idx] >> node5_child) & 1
    in_kid5 = (kids5[node5_idx] >> node5_child) & 1
    
    # Load Node 4 index from texture atlas
    node4_idx = load_from_texture_atlas(node5s, node5_idx, node5_child)
    
    if in_val5:
        return VdbLeaf(vec3(0.2, 0.2, 0.2), 0, 1, leaff.parents)
    
    if not in_kid5:
        return VdbLeaf(vec3(0.0, 0.0, 0.0), node4_idx, 1, leaff.parents)
    
    # Proceed to Node 4
    node4_global = get_global_node4(pos) # Derived from node5_child
    leaff.parents[1] = Parent(node4_global, node4_idx)
    leaff.num_parents = 2
    return get_vdb_leaf_from_node4(pos, leaff)

def get_vdb_leaf_from_node4(pos, leaff):
    """Traverse Node 4 level."""
    node4_global = leaff.parents[1].origin
    node4_idx = leaff.parents[1].idx
    
    # Calculate local coordinates within Node 4
    node4_local = pos - node4_global * 4 # Assuming 4^4 block size
    node4_child = get_child_index(node4_local, 4)
    
    # Check masks
    in_val4 = (vals4[node4_idx] >> node4_child) & 1
    in_kid4 = (kids4[node4_idx] >> node4_child) & 1
    
    # Load Node 3 index from texture atlas
    node3_idx = load_from_texture_atlas(node4s, node4_idx, node4_child)
    
    if in_val4:
        return VdbLeaf(vec3(0.2, 0.2, 0.2), 0, 2, leaff.parents)
    
    if not in_kid4:
        return VdbLeaf(vec3(0.0, 0.0, 0.0), node3_idx, 2, leaff.parents)
    
    # Proceed to Node 3
    node3_global = get_global_node3(pos) # Derived from node4_child
    leaff.parents[2] = Parent(node3_global, node3_idx)
    leaff.num_parents = 3
    return get_vdb_leaf_from_node3(pos, leaff)

def get_vdb_leaf_from_node3(pos, leaff):
    """Traverse Node 3 level (Leaf)."""
    node3_global = leaff.parents[2].origin
    node3_idx = leaff.parents[2].idx
    
    # Calculate local coordinates within Node 3
    node3_local = pos - node3_global * 3 # Assuming 3^3 block size
    node3_offset = get_offset(node3_local, 3)
    
    # Check mask
    in_val3 = (vals3[node3_idx] >> node3_offset) & 1
    
    # Load voxel value from texture atlas
    voxel = load_from_texture_atlas(node3s, node3_idx, node3_offset)
    
    if in_val3:
        return VdbLeaf(vec3(0.1, 0.1, 0.1), 0, 3, leaff.parents)
    else:
        return VdbLeaf(vec3(0.0, 0.0, 0.0), voxel, 3, leaff.parents)

def get_global_node5(pos):
    """Helper to compute global Node 5 coordinate."""
    # Logic depends on NODE5_TOTAL_LOG_D and coordinate transformation
    # Not fully specified in sources, assumed to be integer division by 5
    return vec3(i32(pos.x / 5), i32(pos.y / 5), i32(pos.z / 5))

def get_global_node4(pos):
    """Helper to compute global Node 4 coordinate."""
    # Logic depends on NODE4_TOTAL_LOG_D
    return vec3(i32(pos.x / 4), i32(pos.y / 4), i32(pos.z / 4))

def get_global_node3(pos):
    """Helper to compute global Node 3 coordinate."""
    # Logic depends on NODE3_TOTAL_LOG_D
    return vec3(i32(pos.x / 3), i32(pos.y / 3), i32(pos.z / 3))

def get_child_index(local_pos, block_size):
    """Calculate child index from local position."""
    # Implementation depends on specific bit packing of 3^3, 4^4, 5^5
    # Not fully specified in sources
    return 0 # Placeholder for actual bit manipulation logic

def load_from_texture_atlas(atlas, idx, offset):
    """Load value from compressed texture atlas."""
    # Implementation depends on texture atlas format
    # Not fully specified in sources
    return 0 # Placeholder
```

### Edge Cases and Pitfalls
- **Unsupported Version**: If `file_version` < 218, the file is rejected [S1, unofficial].
- **Magic Mismatch**: If the first 4 bytes are not `0x56444220`, the file is invalid [S1, unofficial].
- **Precision**: The use of `NODE5_TOTAL_LOG_D = 12` for $5^5=3125$ suggests an approximation or specific bit-packing scheme that may not align with exact logarithmic values, potentially causing coordinate mapping errors if not implemented exactly as in the source [S1, unofficial].
- **Boundary Conditions**: The traversal assumes valid `pos` coordinates within the grid bounds. Out-of-bounds access is not explicitly handled in the provided notes [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity**: The traversal is fixed-depth, involving a constant number of steps (Node 5 -> Node 4 -> Node 3) [S1, unofficial]. Each step involves texture atlas lookups and bit operations, which are $O(1)$ relative to the grid size [S1, unofficial].
- **Memory Complexity**: Memory usage is dominated by the compressed texture atlases (`node5s`, `node4s`, `node3s`) and mask arrays (`kids`, `vals`) [S1, unofficial]. The sparse nature of the VDB structure means memory usage scales with the number of non-empty nodes, not the total volume [S1, unofficial].

### Measured Performance
- Not given in the sources.

### Scaling
- Not given in the sources.

### Strengths and Limitations
- **Strengths**: Fast random access due to fixed-depth traversal and compressed storage [S1, unofficial]. Efficient memory usage for sparse data [S1, unofficial].
- **Limitations**: Complexity of implementation due to custom texture atlas format and bit-packing [S1, unofficial]. Limited to OpenVDB files with specific compression and structure [S1, unofficial].

### When to Use
- Use when accessing sparse volumetric data stored in OpenVDB format with a 3-4-5 tree structure [S1, unofficial].
- Use when memory efficiency is critical and the data is sparse [S1, unofficial].

### When Not to Use
- Do not use for dense volumetric data where a simple 3D array is more efficient [S1, unofficial].
- Do not use if the OpenVDB file version is < 218 [S1, unofficial].

## Background

### Origin
The VDB 3-4-5 Tree Traversal method is described in the unofficial source "NemoInfo/woxel" [S1, unofficial]. It is part of the OpenVDB ecosystem, which was originally developed by DreamWorks Animation for visual effects [S1, unofficial].

### History and Development
OpenVDB has evolved through various versions, with version 218 being the minimum supported version for this traversal method [S1, unofficial]. The 3-4-5 tree topology is a specific variation of the sparse voxel grid structure, designed to optimize storage and access for certain types of volumetric data [S1, unofficial].

### Where It Is Used
This method is used in applications that require efficient random access to sparse volumetric data, such as visual effects, scientific visualization, and medical imaging [S1, unofficial].

## Sources

[S1] NemoInfo/woxel, https://github.com/NemoInfo/woxel, fetched 2026-10-24 (unofficial)
