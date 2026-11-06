# Octree File I/O Format

Also called: Octree Header Format, OctreeInfo

## Overview
This format defines the ASCII header and binary payload structure for serializing a Sparse Voxel Octree (SVO) to disk, enabling efficient loading and unloading of volumetric data [S1, unofficial]. It solves the problem of persistent storage for CPU-based ray casting by separating structural metadata from voxel data, allowing the renderer to map memory efficiently without parsing complex binary headers for every access [S1, unofficial]. It differs from file-system-based indexing methods (like [Octree-based Space Subdivision with File System Indexing](5866-octree-based-space-subdivision-with-file-system-indexing.md)) by storing the entire octree structure in a single contiguous file rather than distributing nodes across the directory hierarchy [S1, unofficial][S2].

## How it works
### Inputs and Outputs
- **Input**: An in-memory Sparse Voxel Octree structure containing node pointers and voxel data.
- **Output**: A single file containing an ASCII header followed by binary blocks.

### Data Structures
The format relies on the `OctreeInfo` metadata structure [S1, unofficial].

**OctreeInfo Layout**:
- `version`: `int` (Default: 1)
- `base_filename`: `string` (Not stored in the header itself, but used to locate the binary data file)
- `gridlength`: `size_t` (Default: 1024)
- `n_nodes`: `size_t` (Count of internal octree nodes)
- `n_data`: `size_t` (Count of leaf voxel data entries)

### Steps

#### Writing
1.  **Write Header**:
    -   Write the magic string `#octreeheader 1` followed by a newline.
    -   Write `gridlength <value>` followed by a newline.
    -   Write `n_nodes <value>` followed by a newline.
    -   Write `n_data <value>` followed by a newline.
    -   Write `END` followed by a newline.
2.  **Write Voxel Data**:
    -   Write a binary block of size `VOXELDATA_SIZE` containing the Morton codes for all leaf voxels. The exact definition of `VOXELDATA_SIZE` and the byte order are not specified in the source, but it is implied to be a contiguous binary dump of the Morton codes [S1, unofficial].
3.  **Write Nodes**:
    -   Write a binary block containing `n_nodes` entries. Each entry consists of three `size_t` values representing the node data (typically child pointers or indices) [S1, unofficial].

#### Reading
1.  **Parse Header**:
    -   Read the first line. Verify it starts with `#octreeheader`. Extract the version integer.
    -   Loop through subsequent lines reading key-value pairs.
    -   If the key is `gridlength`, parse the integer value into `info.gridlength`.
    -   If the key is `n_nodes`, parse the integer value into `info.n_nodes`.
    -   If the key is `n_data`, parse the integer value into `info.n_data`.
    -   If the key is `END`, stop parsing.
    -   If the key is unknown, skip the line [S1, unofficial].
2.  **Load Voxel Data**:
    -   Read a binary block of size `VOXELDATA_SIZE` to retrieve the Morton codes [S1, unofficial].
3.  **Load Nodes**:
    -   Read `n_nodes` sets of three `size_t` values to reconstruct the octree node structure [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import os

def write_octree_file(filepath, gridlength, n_nodes, n_data, voxel_mortons, node_data):
    """
    Writes the Octree File I/O Format.
    
    Args:
        filepath (str): Path to the output file.
        gridlength (int): The side length of the voxel grid.
        n_nodes (int): Number of internal octree nodes.
        n_data (int): Number of leaf voxel data entries.
        voxel_mortons (list[int]): List of Morton codes for leaf voxels.
        node_data (list[tuple]): List of tuples, each containing 3 size_t values for a node.
    """
    # Determine size of voxel data block. 
    # Source [S1, unofficial] mentions VOXELDATA_SIZE but does not define it. 
    # Assuming Morton codes are stored as 64-bit integers for simplicity, 
    # though exact type is not specified [S1, unofficial].
    morton_size = 8 
    voxel_data_size = len(voxel_mortons) * morton_size
    
    # Calculate node data size. size_t is typically 8 bytes on 64-bit systems.
    size_t_size = 8
    node_data_size = n_nodes * 3 * size_t_size
    
    with open(filepath, 'wb') as f:
        # 1. Write ASCII Header
        f.write(b'#octreeheader 1\n')
        f.write(f'gridlength {gridlength}\n'.encode('ascii'))
        f.write(f'n_nodes {n_nodes}\n'.encode('ascii'))
        f.write(f'n_data {n_data}\n'.encode('ascii'))
        f.write(b'END\n')
        
        # 2. Write Voxel Data (Binary)
        # Source [S1, unofficial] states: "Voxel data is written/read as a binary block of size VOXELDATA_SIZE containing the morton code."
        # We pack the Morton codes. Endianness is not specified [S1, unofficial].
        for morton in voxel_mortons:
            f.write(struct.pack('Q', morton)) # 'Q' is unsigned long long (64-bit)
            
        # 3. Write Node Data (Binary)
        # Source [S1, unofficial] states: "Octree nodes are written/read as three size_t values representing n.data."
        for node in node_data:
            # node is a tuple of 3 integers
            for val in node:
                f.write(struct.pack('Q', val))

def read_octree_file(filepath):
    """
    Reads the Octree File I/O Format.
    
    Returns:
        dict: Contains 'version', 'gridlength', 'n_nodes', 'n_data', 'voxel_mortons', 'node_data'.
    """
    info = {
        'version': 1,
        'gridlength': 1024,
        'n_nodes': 0,
        'n_data': 0,
        'voxel_mortons': [],
        'node_data': []
    }
    
    with open(filepath, 'rb') as f:
        # 1. Parse Header
        line = f.readline().decode('ascii').strip()
        if not line.startswith('#octreeheader'):
            raise ValueError("Invalid octree header")
        
        version = int(line.split()[1])
        info['version'] = version
        
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
        # Source [S1, unofficial] implies a binary block follows the header.
        # We assume Morton codes are 64-bit unsigned integers.
        morton_size = 8
        voxel_data_size = info['n_data'] * morton_size
        
        # Read the binary block
        voxel_bytes = f.read(voxel_data_size)
        info['voxel_mortons'] = list(struct.unpack(f'{info["n_data"]}Q', voxel_bytes))
        
        # 3. Read Node Data
        # Source [S1, unofficial] states nodes are three size_t values.
        size_t_size = 8
        node_entries = info['n_nodes']
        node_bytes = f.read(node_entries * 3 * size_t_size)
        
        # Unpack into tuples of 3
        values = list(struct.unpack(f'{node_entries * 3}Q', node_bytes))
        info['node_data'] = [tuple(values[i:i+3]) for i in range(0, len(values), 3)]
        
    return info
```

### Parameters
- `gridlength`: Default 1024 [S1, unofficial].
- `version`: Default 1 [S1, unofficial].

### Edge Cases and Pitfalls
- **Unknown Keywords**: The parser must skip any keyword not recognized as `gridlength`, `n_nodes`, `n_data`, or `END` to maintain forward compatibility [S1, unofficial].
- **Binary Endianness**: The source does not specify the endianness of the binary blocks (`VOXELDATA_SIZE` and node data). Implementers should assume native endianness or a standard (e.g., little-endian) and document it, as mismatched endianness will corrupt Morton codes and node pointers [S1, unofficial].
- **Size Type**: `size_t` size varies by architecture (32-bit vs 64-bit). The file format must be consistent with the architecture that reads it, or the file must specify the pointer size [S1, unofficial].

## Subtopics

### Octree-based Space Subdivision with File System Indexing
A data structure and access method for large volumetric datasets that uses an octree for space subdivision and the Unix file system for storage, replacing traditional databases or slice formats to allow instant access to data blocks [S2]. At each level of the octree, the position of a voxel is determined using three binary decisions: left/right, front/back, and top/bottom [S2]. The octree traversal path is encoded in the file name, which consists of a sequence of numbers (0-7), where each number represents the index of the child node taken at that level of the tree [S2]. Empty areas are not stored in the octree structure, saving disk space and RAM, determined in a preprocessing step [S2]. Unlike the Octreemizer [38] which only uses octree subdivision without compression, this method incorporates wavelet compression to handle datasets exceeding RAM [S2]. Traditional slice formats are too slow because they require opening too many files to display a 3D volume from cross-sections [S2].

## Operating characteristics
- **Time Complexity**: Parsing the header is $O(1)$ relative to the data size, as it depends only on the number of metadata fields. Reading the binary data is $O(N)$ where $N$ is the number of nodes and voxels.
- **Memory Complexity**: The format requires loading the entire binary payload into memory to reconstruct the octree, unless the reader implements streaming or memory mapping. The `OctreeInfo` structure itself is small [S1, unofficial].
- **Strengths**: Simple ASCII header allows for easy debugging and manual inspection. Binary payload is compact.
- **Limitations**: The entire file must be accessible. The lack of specified endianness or exact binary layout for `VOXELDATA_SIZE` makes it fragile across platforms [S1, unofficial]. It does not support partial loading of the octree structure (e.g., loading only the root) as efficiently as file-system-based indexing [S2].

## Background
The format is described in the context of a CPU-based voxel raycaster implementation [S1, unofficial]. The file-system-based variant is described in the context of gigascale multiresolution volume rendering on distributed display clusters, aiming to overcome the slowness of traditional slice formats [S2].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
