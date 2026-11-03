# Compact pointer tree

Also called: Hierarchical Voxel Compression with Wavelet Transform, Terabyte-scale volumetric compression

## Overview
The Compact Pointer Tree is a tree-based data structure for volumetric RGBA data that enables interactive visualization of terabyte-scale datasets by compressing color values using wavelet transforms and allowing on-demand voxel reconstruction [S1]. It sets itself apart from sibling sparse structures by storing $N^3$ children per internal node (where $N \in [2, 6]$) and using 8-bit offsets relative to a 32-bit base address, which allows for compact storage at the cost of requiring sequential child export during construction [S1]. Unlike [Pointer Compression](1770-pointer-compression.md) which generally reduces pointer size, this method integrates compression into the tree hierarchy itself, using local wavelet transforms on color attributes [S1].

## How it works
### Inputs and Outputs
**Inputs:** Terabyte-scale volumetric RGBA data [S1].
**Outputs:** A compressed tree structure stored on disk/GPU memory, enabling interactive rendering at 8K resolution [S1].

### Data Structures
**Internal Node Layout:**
*   **Children:** $N^3$ child pointers. Each pointer is an 8-bit offset relative to a 32-bit base address stored within the node [S1]. If a child is null, it is marked as such [S1]. The maximum branching factor $N$ is 6 because $6^3 = 216$ fits within 8 bits (max value 255) [S1].
*   **Color Attributes:** Minimum color, average color, and maximum color [S1]. These are used for wavelet compression [S1].
*   **Sign Bit:** A stored bit indicating which reconstruction formula to use [S1].

**Bottom Level:**
*   Arrays of voxel color values [S1].
*   Supports storage of values starting from 4-bit channels [S1].

### Construction Algorithm
The construction occurs in two passes [S1].

**First Pass (Bottom-Up Structure Building):**
1.  Initialize a hashmap with keys `(level, index.x, index.y, index.z)` to track created nodes [S1].
2.  Build Level 0 nodes:
    *   Process $N$ images per thread to build a Level 0 node [S1].
    *   Compute max, min, average, and child masks for the children [S1].
    *   Store node data via hashmap lookup [S1].
3.  Build higher levels recursively:
    *   Launch jobs equal to the dense grid node count for the next level [S1].
    *   Threads iterate over children to compute max, min, average, and child masks [S1].
    *   Repeat until the root node is reached [S1].
    *   *Pitfall:* Children exporting to a linear buffer at each level must be sequential [S1]. This is handled by the hashmap during the first pass to manage node creation efficiently, allowing parallel processing of different branches [S1].

**Second Pass (Compression):**
1.  Parallelize over nodes at any level to compress max, min, and average values [S1].
2.  For each node:
    *   The preprocessor seeks file handles to determined positions [S1].
    *   Recursively calculate the wavelet compressed value [S1]. This is local; it only requires knowledge of the level above [S1].
3.  After compression, the structure is ready for traversal [S1].

### Traversal (Rendering)
1.  Implement hierarchical DDA to skip empty space [S1].
2.  Scale down and modulate voxel position at each level while descending [S1].
3.  Keep a stack of computed values from the first traversal to the bottom [S1].
4.  For subsequent accesses, backtrack the structure to find the node to start descending from, avoiding full root traversal [S1].
5.  Store all data except the bottom level on GPU memory [S1].
6.  Register host memory for the bottom level [S1].

### Reconstruction Formula
To reconstruct a value $x_i^r$ from a compressed value $x_i^c$, one of two formulas is selected based on a stored sign bit [S1]:

$$
x_i^r = (max_{i+1} - ave_{i+1})x_i^c + ave_{i-1}
$$

or

$$
x_i^r = (ave_{i+1} - min_{i+1})x_i^c + min_{i-1}
$$

Where:
*   $x_i^r$: Reconstructed value.
*   $x_i^c$: Compressed value.
*   $min_{i-1}, ave_{i-1}$: Min and average color attributes of the child level (level $i-1$).
*   $min_{i+1}, ave_{i+1}, max_{i+1}$: Min, average, and max color attributes of the parent level (level $i+1$).

This is executed recursively from the root to the bottom to reconstruct voxel values [S1].

### Parameters
*   **N (Branching Factor):** The number of children per internal node along each axis. Range: 2 to 6. Constraint: Maximum $N$ is 6 because node offsets are compressed to 8 bits [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_compact_pointer_tree(input_data, N=6):
    # First Pass: Build Structure
    node_map = {}
    level_0_nodes = []
    
    # Parallelize over threads to build Level 0
    for thread_id in range(num_threads):
        children_data = get_children_data(thread_id)
        node = {
            'max_color': max(children_data),
            'min_color': min(children_data),
            'ave_color': mean(children_data),
            'child_masks': compute_child_masks(children_data),
            'base_address': 0,
            'children': [None] * (N**3),
            'sign_bit': 0
        }
        key = (0, thread_id.x, thread_id.y, thread_id.z)
        node_map[key] = node
        level_0_nodes.append(node)

    # Build higher levels recursively
    current_level_nodes = level_0_nodes
    level = 0
    while not is_root(current_level_nodes):
        level += 1
        next_level_nodes = []
        # Launch jobs equal to dense grid node count
        for job_id in range(dense_grid_node_count):
            children = get_children(current_level_nodes, job_id)
            node = {
                'max_color': max(children),
                'min_color': min(children),
                'ave_color': mean(children),
                'child_masks': compute_child_masks(children),
                'base_address': 0,
                'children': [None] * (N**3),
                'sign_bit': 0
            }
            key = (level, job_id.x, job_id.y, job_id.z)
            node_map[key] = node
            next_level_nodes.append(node)
        current_level_nodes = next_level_nodes

    # Second Pass: Compression
    # Parallelize over nodes at any level to compress max, min, and average values
    for node in node_map.values():
        parent_node = get_parent_node(node)
        node['compressed_max'] = wavelet_compress(node['max_color'], parent_node)
        node['compressed_min'] = wavelet_compress(node['min_color'], parent_node)
        node['compressed_ave'] = wavelet_compress(node['ave_color'], parent_node)
        node['sign_bit'] = 0 if node['max_color'] - node['ave_color'] >= 0 else 1

    return node_map

def reconstruct_voxel(node_map, target_coords):
    current_node = get_root(node_map)
    coords = target_coords

    for level in range(max_level, -1, -1):
        child_idx = get_child_index(coords, level)
        if current_node['child_masks'][child_idx] is None:
            return None

        base_addr = current_node['base_address']
        offset = current_node['children'][child_idx]
        next_node = node_map[base_addr + offset]

        if level > 0:
            if next_node['sign_bit'] == 0:
                val = (next_node['max_color'] - next_node['ave_color']) * next_node['compressed_ave'] + next_node['ave_color']
            else:
                val = (next_node['ave_color'] - next_node['min_color']) * next_node['compressed_min'] + next_node['min_color']
        current_node = next_node

    return current_node['voxels'][coords]

def render_hierarchical_dda(camera, node_map):
    # Store all data except bottom level on GPU memory
    # Register host memory for bottom level
    stack = []
    current_node = get_root(node_map)
    coords = camera.initial_coords

    while current_node is not None and not is_leaf(current_node):
        stack.append(current_node)
        coords = scale_and_modulate(coords, current_node)
        child_idx = get_child_index(coords, current_node['level'])
        base_addr = current_node['base_address']
        offset = current_node['children'][child_idx]
        current_node = node_map[base_addr + offset]

    # Subsequent accesses backtrack using stack
    return stack
```

### Subtopics
*   [Bottom-up SVO to DAG Reduction](898-bottom-up-svo-to-dag-reduction.md): Reduces SVOs to DAGs for compression. Choose when memory reduction is prioritized over traversal speed.
*   [SparseGrid](904-sparsegrid.md): Hash-based sparse storage. Choose for uniform access patterns and simpler implementation.
*   [Hash Grids](936-hash-grids.md): Hash-based sparse storage. Choose for GPU-friendly, uniform access.
*   [Implicit Octree Storage](939-implicit-octree-storage.md): Stores octrees implicitly. Choose when pointer overhead must be eliminated entirely.
*   [Pointerless Sparse Voxel Directed Acyclic Graph (PSVDAG)](949-pointerless-sparse-voxel-directed-acyclic-graph-psvdag.md): DAG-based with no pointers. Choose for maximum compression and ray tracing.
*   [Sparse Voxel DAG Ray Traversal](1147-sparse-voxel-dag-ray-traversal.md): Traversal for DAGs. Choose when ray tracing is the primary workload.
*   [Implicit Octree Traversal](1220-implicit-octree-traversal.md): Traversal for implicit octrees. Choose for pointerless traversal.
*   [OpenVDB (VDB)](1508-openvdb-vdb.md): VDB-style sparse storage. Choose for general-purpose volumetric editing and simulation.
*   [Pointer Compression](1770-pointer-compression.md): General pointer compression. Choose when adapting existing tree structures.
*   [ExaBricks Hierarchy](2182-exabricks-hierarchy.md): Brick-based hierarchy. Choose for large-scale scientific visualization.
*   [Time-Variant Volume Rendering with Octrees](2232-time-variant-volume-rendering-with-octrees.md): Octrees for time-varying data. Choose for temporal datasets.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Construction requires parallelization to handle terabyte-scale data [S1]. Traversal uses hierarchical DDA to skip empty space [S1].
*   **Memory Complexity:** Compresses 2.5 TB of data to 0.7 TB (3.7 TB decoded) at 45 dB PSNR [S1].
*   **Scaling:** Not given in the sources.
*   **Measured Performance:** Rendered 8K at 33 ms per frame on an AMD Radeon PRO W6800 GPU, AMD Ryzen Threadripper PRO 3955WX, 1536 GB RAM, Windows 10, Vulkan 1.2, HIP through Orochi [S1].
*   **Strengths:** Enables interactive visualization of terabyte-scale data; supports on-demand voxel reconstruction; high compression ratio [S1].
*   **Limitations:** Construction requires parallelization challenges to be solved (sequential child export); maximum branching factor limited to 6 due to 8-bit offsets [S1].
*   **When to use:** When visualizing terabyte-scale volumetric data interactively is required [S1].
*   **When not to use:** When higher branching factors are needed or when construction time is critical and parallelization overhead is too high [S1].

## Background
The method is described in the paper "Compression and Interactive Visualization of Terabyte Scale Volumetric RGBA Data with Voxel-scale Details" [S1]. It was developed to address the challenge of visualizing extremely large volumetric datasets interactively [S1]. The method builds on hierarchical DDA [Museth 2021] to accelerate content on GPU while streaming [S1].

## Sources
[S1] Compression and Interactive Visualization of Terabyte Scale Volumetric RGBA Data with Voxel-scale Details, https://doi.org/10.1145/3532719.3543256, fetched 2026-10-29
