# Compact pointer tree

Also called: Hierarchical Voxel Compression with Wavelet Transform, Terabyte-scale volumetric compression, Terabyte-scale volumetric rendering, Voxel-scale Volumetric Compression and Traversal, Wavelet-compressed sparse volume tree

## Overview
The Compact Pointer Tree is a tree-based data structure and traversal algorithm designed for the interactive visualization of terabyte-scale RGBA volumetric data with voxel-level details [S1][S2, unofficial]. It solves the problem of rendering massive, sparse volumes on GPUs by combining wavelet-based color compression with compact 8-bit relative pointers [S1][S2, unofficial]. Unlike standard octrees that use fixed 8 children, this structure uses a variable branching factor $N$ ($2 \le N \le 6$) to optimize memory usage, enabling real-time frame rates for 8K resolution volumes [S1][S2, unofficial]. It builds on [Tree-based structures (Octree, Pointer Tree, Sparse Voxel DAG)](4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md) by introducing recursive color attribute compression and hierarchical DDA traversal that avoids full root traversal for subsequent accesses [S1][S2, unofficial].

## How it works
### Inputs and Outputs
- **Input:** Terabyte-scale RGBA volumetric data (e.g., $19,008 \times 12,672 \times 4,000$ voxels) [S1][S2, unofficial].
- **Output:** A compressed tree structure stored on GPU memory (except the bottom level) and a traversal state for interactive rendering [S1][S2, unofficial].

### Data Structures
The structure is a tree where each internal node contains $N^3$ children, minimum color, average color, maximum color, and child offsets [S1][S2, unofficial].

**Internal Node Layout:**
- **Base Address:** A 32-bit integer storing the base address for child offsets [S1][S2, unofficial].
- **Child Offsets:** $N^3$ entries, each an 8-bit offset relative to the Base Address [S1][S2, unofficial].
  - Constraint: $N \le 6$ because $6^3 = 216$ fits in 8 bits ($2^8=256$) [S1][S2, unofficial].
  - Null children are represented by specific offset values or flags (implementation detail not fully specified, but sparsity is encoded by absence/pointer logic) [S1][S2, unofficial].
- **Color Attributes:** Minimum, Average, and Maximum color values (RGBA) [S1][S2, unofficial]. These are used for wavelet compression [S1][S2, unofficial].
- **Sign Bit:** A single bit stored to select the reconstruction formula [S1][S2, unofficial].

**Leaf Node Layout:**
- Contains arrays of voxel color values [S1][S2, unofficial].
- Supports storage starting from 4-bit channels [S1][S2, unofficial].

### Construction Algorithm
Construction is a two-pass bottom-up process [S1][S2, unofficial].

**First Pass: Node Creation**
1. Initialize a hashmap with keys computed from `(level, index.x, index.y, index.z)` to track created nodes [S1][S2, unofficial].
2. Launch parallel jobs to build Level 0 nodes. A single thread processes $N$ images (or blocks) to build a Level 0 node [S1][S2, unofficial].
3. For each node, compute the child masks, minimum color, average color, and maximum color from its children [S1][S2, unofficial].
4. Store node data using 32-bit or 64-bit types initially [S1][S2, unofficial].
5. Repeat recursively for subsequent levels until the root node is reached [S1][S2, unofficial].
   - *Pitfall:* Children exporting to a linear buffer at each level must be sequential. The hashmap manages this parallelization challenge [S1][S2, unofficial].

**Second Pass: Compression**
1. Parallelize over nodes at any level to compress the max, min, and average color values [S1][S2, unofficial].
2. Compression is local; only requires knowledge of the level above [S1][S2, unofficial].
3. For each node, recursively calculate the wavelet compressed value using the color attributes [S1][S2, unofficial].
4. Store the compressed values and the sign bit [S1][S2, unofficial].

### Traversal Algorithm
1. **Initial Traversal:** Traverse the tree from the root to the bottom level once to initialize the rendering state [S1][S2, unofficial].
2. **Hierarchical DDA:** Implement a hierarchical DDA to skip empty space [S1][S2, unofficial].
3. **Stack Maintenance:** Keep a stack of computed values (node attributes) from the initial traversal to the bottom [S1][S2, unofficial].
4. **Subsequent Access:** When accessing other voxel data, backtrack the structure using the stack to find the node to start descending from, avoiding full root traversal [S1][S2, unofficial].
5. **Memory Management:** Store all data except the bottom level directly on GPU memory. Register host memory for the bottom level [S1][S2, unofficial].

### Reconstruction Formula
To reconstruct a voxel value $x_i^r$ from a compressed value $x_i^c$, use one of two formulas based on a stored sign bit [S1][S2, unofficial]:

$$ x_i^r = (max_{i+1} - ave_{i+1})x_i^c + ave_{i-1} $$

or

$$ x_i^r = (ave_{i+1} - min_{i+1})x_i^c + min_{i-1} $$

Where:
- $x_i^r$: Reconstructed value at level $i$ [S1][S2, unofficial].
- $x_i^c$: Compressed value at level $i$ [S1][S2, unofficial].
- $max_{i+1}, ave_{i+1}, min_{i+1}$: Maximum, average, and minimum color attributes of the parent node (level $i+1$) [S1][S2, unofficial].
- $ave_{i-1}, min_{i-1}$: Average and minimum color attributes of the sibling node (level $i-1$) [S1][S2, unofficial].
- The selection depends on the stored sign bit [S1][S2, unofficial].

### Parameters
- **Branching Factor ($N$):** Integer between 2 and 6 [S1][S2, unofficial].
  - Typical value: 3 (as shown in Fig. 2 of [S1][S2, unofficial]).
  - Constraint: $N \le 6$ due to 8-bit offset compression [S1][S2, unofficial].
- **Color Channel Bit Depth:** Supports 4-bit channels at the leaf level [S1][S2, unofficial].

### Edge Cases and Pitfalls
- **Parallelization:** Children exporting to a linear buffer at each level need to be sequential. This is handled by using a hashmap during the first pass [S1][S2, unofficial].
- **Precision:** Wavelet compression introduces approximation error. The method achieves 45 dB PSNR [S1][S2, unofficial].
- **Memory:** The bottom level is exposed via host memory registration, which may require careful management for large volumes [S1][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_compact_pointer_tree(input_data, N):
    """
    Builds the Compact Pointer Tree in two passes.
    :param input_data: Terabyte-scale RGBA volumetric data
    :param N: Branching factor (2 <= N <= 6)
    :return: Root node of the compressed tree
    """
    # First Pass: Node Creation
    node_map = {}  # Hashmap: key=(level, x, y, z), value=node_data
    
    # Build Level 0
    level = 0
    # Parallel job: process blocks of input data
    for block in parallel_blocks(input_data, N):
        node = create_level_0_node(block, N)
        key = (level, block.x, block.y, block.z)
        node_map[key] = node
        
    # Build higher levels recursively
    while not is_root(level):
        level += 1
        next_level_map = {}
        # Parallel job: process nodes from previous level
        for key, parent_node in node_map.items():
            child_nodes = get_children(parent_node, N)
            new_node = create_internal_node(child_nodes, N)
            new_key = (level, new_node.x, new_node.y, new_node.z)
            next_level_map[new_key] = new_node
        node_map = next_level_map
        
    root = node_map[root_key]
    
    # Second Pass: Compression
    compress_nodes(root, N)
    
    return root

def create_level_0_node(block, N):
    """
    Creates a Level 0 node from input data block.
    """
    node = Node()
    node.min_color = compute_min(block)
    node.ave_color = compute_average(block)
    node.max_color = compute_max(block)
    node.child_masks = compute_child_masks(block, N)
    # Store initial 32/64-bit types
    return node

def create_internal_node(child_nodes, N):
    """
    Creates an internal node from child nodes.
    """
    node = Node()
    node.min_color = compute_min(child_nodes)
    node.ave_color = compute_average(child_nodes)
    node.max_color = compute_max(child_nodes)
    node.child_masks = compute_child_masks(child_nodes, N)
    return node

def compress_nodes(node, N):
    """
    Recursively compresses color attributes using wavelet transform.
    """
    if is_leaf(node):
        return
    
    # Compress max, min, ave using local compression
    # Requires knowledge of one level above (handled by preprocessor seeking file handles)
    node.compressed_max = wavelet_compress(node.max_color)
    node.compressed_ave = wavelet_compress(node.ave_color)
    node.compressed_min = wavelet_compress(node.min_color)
    
    # Store sign bit for reconstruction
    node.sign_bit = determine_sign_bit(node)
    
    # Recurse
    for child in node.children:
        compress_nodes(child, N)

def reconstruct_voxel(node, compressed_value, sign_bit):
    """
    Reconstructs a voxel value from compressed data.
    """
    if sign_bit == 0:
        reconstructed = (node.max_color - node.ave_color) * compressed_value + node.ave_color
    else:
        reconstructed = (node.ave_color - node.min_color) * compressed_value + node.min_color
    return reconstructed

def traverse_hierarchical_dda(root, ray_origin, ray_direction):
    """
    Traverses the tree using hierarchical DDA.
    """
    # Initial traversal to bottom
    stack = []
    current_node = root
    while not is_leaf(current_node):
        stack.append(current_node)
        current_node = get_next_node_in_dda(current_node, ray_origin, ray_direction)
    
    # Process leaf nodes
    process_leaf(current_node)
    
    # Backtrack for subsequent accesses
    while stack:
        current_node = stack.pop()
        # Find next node to descend from
        next_node = find_next_node(current_node, ray_origin, ray_direction)
        if next_node:
            stack.append(next_node)
            # Descend to leaf
            while not is_leaf(next_node):
                stack.append(next_node)
                next_node = get_next_node_in_dda(next_node, ray_origin, ray_direction)
            process_leaf(next_node)
```

## Operating characteristics
- **Time Complexity:** Construction is $O(V)$ where $V$ is the number of voxels, with parallelization over nodes [S1][S2, unofficial]. Traversal is $O(\log V)$ for hierarchical DDA, but optimized to avoid full root traversal for subsequent accesses [S1][S2, unofficial].
- **Memory Complexity:** Compression ratio of approximately 3.7x (2.5 TB input to 0.7 TB compressed) [S1][S2, unofficial]. All data except the bottom level is stored on GPU memory [S1][S2, unofficial].
- **Measured Performance:** On an AMD Radeon PRO W6800 GPU, AMD Ryzen Threadripper PRO 3955WX, 1536 GB RAM, Windows 10, Vulkan 1.2, HIP through Orochi:
  - Compressed 2.5 TB of data (3.7 TB decoded) to 0.7 TB at 45 dB PSNR [S1][S2, unofficial].
  - Interactively visualizes 8K resolution at 33 ms per frame [S1][S2, unofficial].
- **Strengths:** Enables interactive visualization of terabyte-scale volumes with voxel-level details [S1][S2, unofficial]. Efficient memory usage through compact pointers and wavelet compression [S1][S2, unofficial].
- **Limitations:** Compression introduces approximation error (45 dB PSNR) [S1][S2, unofficial]. Parallelization during construction requires careful handling of sequential buffer exports [S1][S2, unofficial].

## Background
The Compact Pointer Tree was introduced by Takahiro Harada, Yusuke Takeda, and Yasuhiro Iba in 2022 at SIGGRAPH '22 Posters [S1][S2, unofficial]. It addresses the need for rendering high-resolution volumetric data (e.g., Earth and extraterrestrial material samples) that exceeds gigabyte scales, where previous methods relied on heavy approximations [S1][S2, unofficial]. The method builds on hierarchical DDA techniques [Museth 2021] and wavelet compression principles [S1][S2, unofficial].

## Sources
[S1] Compression and Interactive Visualization of Terabyte Scale Volumetric RGBA Data with Voxel-scale Details, https://doi.org/10.1145/3532719.3543256, fetched 2026-10-29
[S2] PDF Compression and Interactive Visualization of Terabyte Scale ... - GPUOpen, https://gpuopen.com/download/SIG2022_VoxelpoolCompressed.pdf, fetched 2026-10-29 (unofficial)
