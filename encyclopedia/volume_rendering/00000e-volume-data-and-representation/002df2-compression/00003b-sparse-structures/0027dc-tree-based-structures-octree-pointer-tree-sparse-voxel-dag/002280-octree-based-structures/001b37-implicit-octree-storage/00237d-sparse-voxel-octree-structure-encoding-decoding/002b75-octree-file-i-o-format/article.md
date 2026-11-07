# Octree File I/O Format

Also called: Octree File Format & Header, Octree Header Format, OctreeInfo

## Overview
This format stores octree metadata in a separate text header file containing compression type, grid size, node count, and data position [S1, S2, unofficial]. It solves the problem of persistent storage for CPU-based ray casting by separating structural metadata from voxel data, allowing the renderer to map memory efficiently without parsing complex binary headers for every access [S1, unofficial]. It differs from file-system-based indexing methods (like [Octree-based Space Subdivision with File System Indexing](5866-octree-based-space-subdivision-with-file-system-indexing.md)) by storing the entire octree structure in a single contiguous file rather than distributing nodes across the directory hierarchy [S1, S2, unofficial].

## How it works
### Inputs and Outputs
- **Input**: An in-memory octree structure containing node data and leaf voxel Morton codes [S1, unofficial].
- **Output**: An ASCII header file followed by binary data blocks [S1, S2, unofficial].

### Data Structures
The format relies on the `OctreeInfo` metadata structure [S1, unofficial].

**OctreeInfo Layout**:
- `version`: `int` (Default: 1) [S1, unofficial]
- `base_filename`: `string` (Used to locate the binary data file) [S1, unofficial]
- `gridlength`: `size_t` (Default: 1024) [S1, unofficial]
- `n_nodes`: `size_t` (Count of internal octree nodes) [S1, unofficial]
- `n_data`: `size_t` (Count of leaf voxel data entries) [S1, unofficial]

### Steps

#### Writing
1. Open the header file for writing [S2, unofficial].
2. Write the magic string `#octreeheader 1` followed by a newline [S1, unofficial].
3. Write `gridlength <value>` followed by a newline [S1, unofficial].
4. Write `n_nodes <value>` followed by a newline [S1, unofficial].
5. Write `n_data <value>` followed by a newline [S1, unofficial].
6. Write `END` followed by a newline [S1, unofficial].
7. Write a binary block of size `VOXELDATA_SIZE` containing the Morton codes for all leaf voxels [S1, unofficial].
8. Write a binary block containing `n_nodes` entries, each consisting of three `size_t` values representing node data [S1, unofficial].

#### Reading
1. Open the header file for reading [S2, unofficial].
2. Read the first line. Verify it starts with `#octreeheader`. Extract the version integer [S1, unofficial].
3. Loop through subsequent lines reading key-value pairs [S1, unofficial].
4. If the key is `gridlength`, parse the integer value into `info.gridlength` [S1, unofficial].
5. If the key is `n_nodes`, parse the integer value into `info.n_nodes` [S1, unofficial].
6. If the key is `n_data`, parse the integer value into `info.n_data` [S1, unofficial].
7. If the key is `END`, stop parsing [S1, unofficial].
8. If the key is unknown, skip the line [S1, unofficial].
9. Read a binary block of size `VOXELDATA_SIZE` to retrieve the Morton codes [S1, unofficial].
10. Read `n_nodes` entries, each consisting of three `size_t` values, to reconstruct the octree node structure [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def write_octree_header(filepath, gridlength, n_nodes, n_data, voxel_mortons, node_data):
    """Writes the Octree File I/O Format to disk."""
    with open(filepath, 'wb') as f:
        # 1. Write ASCII Header
        f.write(b'#octreeheader 1\n')
        f.write(f'gridlength {gridlength}\n'.encode('ascii'))
        f.write(f'n_nodes {n_nodes}\n'.encode('ascii'))
        f.write(f'n_data {n_data}\n'.encode('ascii'))
        f.write(b'END\n')
        
        # 2. Write Voxel Data
        # Source specifies a binary block of size VOXELDATA_SIZE containing morton codes [S1, unofficial].
        # We pack them as 64-bit unsigned integers.
        morton_size = 8
        voxel_data_size = len(voxel_mortons) * morton_size
        for morton in voxel_mortons:
            f.write(struct.pack('Q', morton))
            
        # 3. Write Node Data
        # Source specifies three size_t values per node [S1, unofficial].
        for node in node_data:
            for val in node:
                f.write(struct.pack('Q', val))

def read_octree_header(filepath):
    """Reads the Octree File I/O Format from disk."""
    info = {'version': 1, 'gridlength': 1024, 'n_nodes': 0, 'n_data': 0, 'voxel_mortons': [], 'node_data': []}
    
    with open(filepath, 'rb') as f:
        # 1. Parse Header
        line = f.readline().decode('ascii').strip()
        if not line.startswith('#octreeheader'):
            raise ValueError("Invalid octree header")
        info['version'] = int(line.split()[1])
        
        while True:
            line = f.readline().decode('ascii').strip()
            if line == 'END':
                break
            if not line:
                continue
            parts = line.split()
            key = parts[0]
            if key == 'gridlength':
                info['gridlength'] = int(parts[1])
            elif key == 'n_nodes':
                info['n_nodes'] = int(parts[1])
            elif key == 'n_data':
                info['n_data'] = int(parts[1])
            else:
                # Unknown keywords are skipped [S1, unofficial]
                pass
                
        # 2. Read Voxel Data
        morton_size = 8
        voxel_data_size = info['n_data'] * morton_size
        voxel_bytes = f.read(voxel_data_size)
        info['voxel_mortons'] = list(struct.unpack(f'{info["n_data"]}Q', voxel_bytes))
        
        # 3. Read Node Data
        size_t_size = 8
        node_entries = info['n_nodes']
        node_bytes = f.read(node_entries * 3 * size_t_size)
        values = list(struct.unpack(f'{node_entries * 3}Q', node_bytes))
        info['node_data'] = [tuple(values[i:i+3]) for i in range(0, len(values), 3)]
        
    return info
```

### Parameters
- `gridlength`: Default 1024 [S1, unofficial].
- `version`: Default 1 [S1, unofficial].

### Edge Cases and Pitfalls
- **Unknown Keywords**: The parser must skip any keyword not recognized as `gridlength`, `n_nodes`, `n_data`, or `END` to maintain forward compatibility [S1, unofficial].
- **Binary Endianness**: The source does not specify the endianness of the binary blocks (`VOXELDATA_SIZE` and node data) [G]. Implementers must assume a consistent endianness or document the choice, as mismatched endianness will corrupt Morton codes and node pointers.
- **Size Type**: `size_t` size varies by architecture (32-bit vs 64-bit) [G]. The file format must be consistent with the architecture that reads it, or the file must specify the pointer size.

## Subtopics

### Octree-based Space Subdivision with File System Indexing
A data structure and access method for large volumetric datasets that uses an octree for space subdivision and the Unix file system for storage, replacing traditional databases or slice formats to allow instant access to data blocks [S3]. At each level of the octree, the position of a voxel is determined using three binary decisions: left/right, front/back, and top/bottom [S3]. The octree traversal path is encoded in the file name, which consists of a sequence of numbers (0-7), where each number represents the index of the child node taken at that level of the tree [S3]. Empty areas are not stored in the octree structure, saving disk space and RAM, determined in a preprocessing step [S3]. Unlike the Octreemizer [38] which only uses octree subdivision without compression, this method incorporates wavelet compression to handle datasets exceeding RAM [S3]. Traditional slice formats are too slow because they require opening too many files to display a 3D volume from cross-sections [S3].

### Octree Node I/O
Serializes and deserializes individual octree nodes to/from a binary file stream [S2, unofficial]. Writing calls `fwrite` to output three `size_t` values starting from `m_Data` to the file; if compression is not `NONE`, it calls `WriteNodeCompress`, and if `m_Data.size() >= 1000`, it calls `CompressFileIncremental(false)` [S2, unofficial]. Reading calls `fread` to input three `size_t` values into `m_Data` [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Parsing the header is $O(1)$ relative to the data size, as it depends only on the number of metadata fields [G]. Reading the binary data is $O(N)$ where $N$ is the number of nodes and voxels [G].
- **Memory Complexity**: The format requires loading the entire binary payload into memory to reconstruct the octree, unless the reader implements streaming or memory mapping [G]. The `OctreeInfo` structure itself is small [S1, unofficial].
- **Strengths**: Simple ASCII header allows for easy debugging and manual inspection [S1, unofficial]. Binary payload is compact [S1, unofficial].
- **Limitations**: The entire file must be accessible [S1, unofficial]. The lack of specified endianness or exact binary layout for `VOXELDATA_SIZE` makes it fragile across platforms [G].

## Background
The format is described in the context of a CPU-based voxel raycaster implementation [S1, unofficial]. The file-system-based variant is described in the context of gigascale multiresolution volume rendering on distributed display clusters, aiming to overcome the slowness of traditional slice formats [S3].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
[S3] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
