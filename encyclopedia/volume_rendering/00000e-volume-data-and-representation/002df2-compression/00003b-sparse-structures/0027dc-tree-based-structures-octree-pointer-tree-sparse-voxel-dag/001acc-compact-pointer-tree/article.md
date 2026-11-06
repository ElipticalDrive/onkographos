# Compact pointer tree

Also called: Hierarchical Voxel Compression with Wavelet Transform, Terabyte-scale volumetric compression

## Overview
The Compact Pointer Tree is a tree-based data structure designed for the interactive visualization of terabyte-scale volumetric RGBA data. It solves the problem of memory bandwidth and storage limitations by compressing color values using wavelet transforms and encoding spatial locality via compact 8-bit offsets. It sets itself apart from standard octrees by using a variable branching factor ($N^3$) and storing min/avg/max color attributes to enable lossy compression of the color space, allowing for on-demand voxel reconstruction from compressed nodes.

## How it works
**Inputs:** Terabyte-scale volumetric RGBA data.
**Outputs:** A compressed tree structure stored on disk/GPU memory, enabling interactive rendering.

**Data Structures:**
*   **Internal Node:**
    *   `base_address`: 32-bit integer. The base address for child offsets.
    *   `offsets`: Array of $N^3$ 8-bit integers. Each offset points to a child node relative to `base_address`. Value 0 or specific sentinel indicates null/empty child.
    *   `min_color`: 32-bit (or higher) color attribute.
    *   `avg_color`: 32-bit (or higher) color attribute.
    *   `max_color`: 32-bit (or higher) color attribute.
    *   `sign_bit`: 1-bit flag indicating which reconstruction formula to use.
*   **Leaf Node (Bottom Level):**
    *   Array of voxel color values. Channels can be stored at 4-bit precision.

**Parameters:**
*   $N$: Branching factor per axis. Range: $2 \le N \le 6$.
    *   Constraint: $N \le 6$ because child pointers are compressed to 8-bit offsets. The maximum addressable range within an 8-bit offset relative to a 32-bit base must cover the $N^3$ children.
*   Typical values: $N=2$ (Octree-like) to $N=6$.

**Algorithm Steps:**

1.  **Construction - First Pass (Bottom-Up):**
    1.  Initialize a hashmap with keys `(level, index.x, index.y, index.z)` to track created nodes and ensure sequential export to linear buffers.
    2.  Process input data to build Level 0 nodes.
    3.  For each Level 0 node, compute `min_color`, `max_color`, and `avg_color` from its constituent voxels.
    4.  Set child pointers to null (since Level 0 are leaves in the tree structure, or point to deeper levels if $N>1$ is applied recursively, but the text implies Level 0 are the base units). *Note: The text describes building Level 0, then Level 1, etc. Level 0 nodes contain the raw voxel data or initial aggregates.*
    5.  Launch parallel jobs equal to the dense grid node count for the next level.
    6.  Threads iterate over children of nodes at the current level to compute their `min`, `max`, and `avg` color attributes and child masks.
    7.  Store node data via hashmap lookup.
    8.  Repeat recursively until the root node is reached.

2.  **Construction - Second Pass (Compression):**
    1.  Parallelize over nodes at any level.
    2.  For each node, compress its `min`, `max`, and `avg` color attributes using a wavelet transform.
    3.  The compression is local; it requires knowledge of the node's own attributes and those of the level above (parent) to calculate wavelet coefficients.
    4.  The preprocessor seeks file handles to determined positions to perform this recursive calculation.
    5.  Store the compressed values and the `sign_bit` in the node.

3.  **Traversal (Rendering):**
    1.  Implement hierarchical DDA (Digital Differential Analyzer) to skip empty space.
    2.  Scale down and modulate voxel position at each level while descending the tree.
    3.  Maintain a stack of computed values from the first traversal to the bottom.
    4.  For subsequent accesses, backtrack the structure to find the node to start descending from, avoiding full root traversal.
    5.  Store all data except the bottom level on GPU memory.
    6.  Register host memory for the bottom level.

**Reconstruction Formula:**
To reconstruct a voxel value $x_i^r$ from a compressed value $x_i^c$ at level $i$:
$$ x_i^r = (max_{i+1} - ave_{i+1})x_i^c + ave_{i-1} $$
OR
$$ x_i^r = (ave_{i+1} - min_{i+1})x_i^c + min_{i-1} $$

Where:
*   $x_i^r$: Reconstructed value.
*   $x_i^c$: Compressed value (wavelet coefficient).
*   $max_{i+1}, ave_{i+1}, min_{i+1}$: Color attributes (max, average, min) of the parent node (level $i+1$).
*   $ave_{i-1}, min_{i-1}$: Color attributes (average, min) of the sibling or adjacent node at level $i-1$? *Clarification from text:* The text says "min/ave/max refer to color attributes of adjacent levels." Specifically, the formula uses attributes from level $i+1$ (parent) and level $i-1$ (sibling/neighbor?). The text states: "reconstruction is done by traversing nodes from root to leaf." The selection between the two equations depends on a stored `sign_bit`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_tree_first_pass(input_data, N):
    """
    Bottom-up construction of the tree structure.
    """
    # Hashmap to manage node creation and ensure sequential buffer export
    node_map = HashMap() 
    
    # Build Level 0 nodes
    level_0_nodes = []
    for voxel_block in input_data:
        node = Node()
        node.min_color = compute_min(voxel_block)
        node.max_color = compute_max(voxel_block)
        node.avg_color = compute_avg(voxel_block)
        node.offsets = [0] * (N**3) # Null pointers initially
        node.base_address = 0 # To be resolved
        level_0_nodes.append(node)
        node_map[(0, node.index)] = node

    current_level_nodes = level_0_nodes
    level = 0
    
    while current_level_nodes is not empty:
        next_level_nodes = []
        # Parallel job for each dense grid node at current level
        for node in current_level_nodes:
            parent_node = Node()
            # Compute aggregates from children
            parent_node.min_color = min(child.min_color for child in node.children)
            parent_node.max_color = max(child.max_color for child in node.children)
            parent_node.avg_color = avg(child.avg_color for child in node.children)
            
            # Set child offsets
            # Note: Offsets are 8-bit relative to base_address
            # This requires sequential export to linear buffer, handled by node_map
            for i, child in enumerate(node.children):
                parent_node.offsets[i] = compute_offset(child, parent_node.base_address)
            
            next_level_nodes.append(parent_node)
            node_map[(level + 1, parent_node.index)] = parent_node
            
        current_level_nodes = next_level_nodes
        level += 1
        
    return root_node

def build_tree_second_pass(root_node):
    """
    Compress color attributes using wavelet transform.
    """
    # Parallelize over all nodes
    for node in all_nodes(root_node):
        # Compression is local, requires one level above
        # Preprocessor seeks file handles to determined positions
        node.compressed_min = wavelet_compress(node.min_color, node.parent.min_color)
        node.compressed_avg = wavelet_compress(node.avg_color, node.parent.avg_color)
        node.compressed_max = wavelet_compress(node.max_color, node.parent.max_color)
        
        # Determine sign bit for reconstruction
        if node.compressed_avg > node.compressed_min:
            node.sign_bit = 0
        else:
            node.sign_bit = 1

def reconstruct_voxel(node, compressed_value, parent_node, sibling_node):
    """
    Reconstruct a voxel value from compressed data.
    """
    if node.sign_bit == 0:
        # Formula 1
        reconstructed = (parent_node.max_color - parent_node.avg_color) * compressed_value + sibling_node.avg_color
    else:
        # Formula 2
        reconstructed = (parent_node.avg_color - parent_node.min_color) * compressed_value + sibling_node.min_color
    return reconstructed

def render_hierarchical_dda(camera, root_node):
    """
    Render using hierarchical DDA and backtracking.
    """
    # First traversal: build stack of values
    stack = []
    current_node = root_node
    while not is_leaf(current_node):
        stack.append(current_node)
        # Scale down and modulate voxel position
        next_node = get_child(current_node, camera.direction)
        if next_node is null:
            break # Skip empty space
        current_node = next_node
    
    # Process leaf
    if is_leaf(current_node):
        render_voxel(current_node, camera)

    # Subsequent accesses: backtrack to start node
    # Avoid full root traversal
    start_node = backtrack_from_stack(stack, camera.current_position)
    # Traverse from start_node
```

## Operating characteristics
*   **Compression Ratio:** Achieved 0.7 TB compressed size from 2.5 TB of input data (3.7 TB decoded size), at 45 dB PSNR [S1].
*   **Performance:** Rendered 8K images at 33 ms per frame [S1].
*   **Hardware:** Tested on AMD Radeon PRO W6800 GPU, AMD Ryzen Threadripper PRO 3955WX, 1536 GB RAM [S1].
*   **Memory Management:** All data except the bottom level is stored in GPU memory. The bottom level is registered as host memory [S1].
*   **Strengths:** Enables interactive visualization of terabyte-scale data. Efficient skipping of empty space via hierarchical DDA.
*   **Limitations:** Parallelization is challenging due to the need for sequential export of children to linear buffers and recursive color attribute dependencies. The 8-bit offset constraint limits the branching factor $N$ to a maximum of 6.

## Background
The Compact Pointer Tree was introduced in the paper "Compression and Interactive Visualization of Terabyte Scale Volumetric RGBA Data with Voxel-scale Details" [S1]. It addresses the need for interactive visualization of extremely large volumetric datasets that exceed standard memory capacities. The method combines hierarchical data structures with wavelet-based color compression to achieve high compression ratios while maintaining visual fidelity.

## Sources
[S1] Compression and Interactive Visualization of Terabyte Scale Volumetric RGBA Data with Voxel-scale Details, https://doi.org/10.1145/3532719.3543256, fetched 2026-10-29
