# Parallel Multiresolution Volume Rendering with Error-Guided Load Balancing

Also called: Error-Guided Tree Traversal, Error-guided hierarchical data distribution, Parallel Multresolution Volume Rendering, Tile-Based Composition, Wavelet-based parallel volume rendering

## Overview
A two-stage parallel algorithm for visualizing large 3D datasets that uses a wavelet tree, error-guided bucketization, and on-the-fly reconstruction to enable interactive multiresolution rendering with minimal inter-processor communication and balanced workload [S1][S2]. It builds on [Hierarchical Data Distribution with Error-Guided Bucketization](6597-hierarchical-data-distribution-with-error-guided-bucketizati.md) by adding a run-time traversal and rendering pipeline that selects nodes based on a user-specified error tolerance, reconstructs data blocks on the fly, and composites partial images via screen-space tiling [S1][S2]. Unlike single-machine multiresolution rendering, it explicitly partitions the wavelet tree to eliminate inter-processor data dependencies at run-time and uses space-filling curves with error-guided bucketization to maintain load balance without redistribution [S1].

## How it works
**Inputs:** Raw 3D dataset (single byte per voxel), user-specified error tolerance $E_{tol}$, number of processors $P$, caching limits (memory $M_{cache}$, disk $D_{cache}$) [S1][S2].
**Outputs:** Final rendered image [S1][S2].

**Data Structures:**
1.  **Wavelet Tree:** Hierarchical structure where leaf nodes are blocks of size $n = 128 \times 128 \times 64$ voxels (1 MB uncompressed) [S1][S2]. Each node stores:
    *   `low_pass`: Low-pass filtered subblock of size $n/8$.
    *   `coeffs`: Wavelet coefficients of size $7n/8$, compressed using run-length and Huffman encoding.
    *   `error`: Hierarchical error metric $E$.
    *   `parent`, `children`: Pointers to tree hierarchy [S1][S2].
2.  **Distribution Units:** The tree is partitioned into units along a Hilbert space-filling curve [S1][S2]. Each processor is assigned a set of units [S1][S2].
3.  **Cache:** LRU cache for reconstructed data blocks, limited to $M_{cache}$ (128 MB) in memory and $D_{cache}$ (256 MB) on disk per processor [S1].

**Preprocessing Stage:**
1.  **Block Subdivision:** Subdivide the original 3D data into leaf blocks of size $n = 2^i \times 2^j \times 2^k$ voxels [S2].
2.  **Wavelet Transform:** Perform a 3D Haar wavelet transform with a lifting scheme on each leaf block [S1][S2]. This produces a low-pass filtered subblock of size $n/8$ and wavelet coefficients of size $7n/8$ [S2].
3.  **Hierarchy Construction:** Collect low-pass filtered subblocks from eight adjacent leaf nodes and group them into a single block of $n$ voxels, forming the low-resolution data for the parent node. Recursively apply the transform and grouping until the root is reached [S2].
4.  **Error Metric Calculation:** Calculate a hierarchical error metric $E$ for each node. The metric must satisfy $E_{parent} \ge E_{child_i}$ for all eight children $i$ [S1][S2]. This guarantees that the parent approximation error is always greater than or equal to its children's errors [S1].
5.  **Compression:** Compress the wavelet coefficients using a lossless scheme (run-length and Huffman encoding) with threshold set to zero [S1][S2].
6.  **Distribution:** Partition the wavelet tree and distribute data blocks among processors along a hierarchical space-filling curve (Hilbert curve) using error-guided bucketization [S1][S2]. Each processor maintains a copy of the wavelet tree skeleton with error values at each node, regardless of data block assignment, to avoid communication during traversal [S1][S2].

**Run-time Stage:**
1.  **Tree Traversal:** If viewing parameters or error tolerance have changed, traverse the wavelet tree front-to-back according to the viewing direction [S1][S2]. Identify nodes with varied resolutions that satisfy the user-specified error tolerance $E_{tol}$ [S1][S2]. The traversal stops at a node if its error $E \le E_{tol}$ [S1].
2.  **Reconstruction:** For each selected node, if the data block is not already reconstructed:
    *   Retrieve the low-pass filtered subblock of size $n/8$ from the parent node. This may require recursive requests to ancestor nodes for low-resolution data [S2].
    *   Obtain high-frequency wavelet coefficients of size $7n/8$ by decoding the bit stream [S2].
    *   Apply the inverse 3D wavelet transform to the grouped low-pass filtered subblock and wavelet coefficients to reconstruct the data block [S2].
    *   Cache the reconstructed block in memory or disk according to the caching policy [S1].
3.  **Parallel Rendering:** Each processor renders the data blocks preassigned to it in parallel [S1][S2]. No data redistribution occurs between processors [S2].
4.  **Tile-Based Composition:**
    *   Partition the screen projection of the volume's bounding box into $P$ tiles, one per processor [S1][S2].
    *   Each processor is assigned one tile [S1].
    *   After rendering a data block, send the resulting partial image to all processors whose tiles overlap with the block's screen projection [S1][S2].
    *   Composite received partial images at each processor to generate the final image for its assigned tile [S1][S2].
5.  **Final Image:** The host processor collects the partial image tiles and creates the final image [S1][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess(raw_data, block_size=(128, 128, 64)):
    """Constructs the wavelet tree and distributes data."""
    leaf_blocks = []
    for block in raw_data.split_into_blocks(block_size):
        low_pass, coeffs = haar_3d_transform(block)
        leaf_blocks.append({
            'low_pass': low_pass,
            'coeffs': compress(coeffs),
            'error': calculate_error(low_pass, block),
            'is_leaf': True
        })
    tree_nodes = leaf_blocks
    while len(tree_nodes) > 1:
        parent_nodes = []
        for i in range(0, len(tree_nodes), 8):
            children = tree_nodes[i:i+8]
            parent_low_pass = group_low_pass(children)
            parent_error = calculate_parent_error(children)
            parent_nodes.append({
                'low_pass': parent_low_pass,
                'coeffs': None,
                'error': parent_error,
                'children': children,
                'is_leaf': False
            })
        tree_nodes = parent_nodes
    root = tree_nodes[0]
    distribution_map = distribute_along_hilbert_curve(root)
    return root, distribution_map

def run_time_render(tree, distribution_map, error_tolerance, view_params, num_processors):
    """Performs parallel multiresolution volume rendering."""
    selected_nodes = []
    traverse_front_to_back(tree, view_params, error_tolerance, selected_nodes)
    tiles = partition_screen_projection(tree.bounding_box, num_processors)
    final_tiles = []
    for proc_id in range(num_processors):
        assigned_blocks = distribution_map.get_blocks_for_processor(proc_id)
        selected_for_proc = [n for n in selected_nodes if n in assigned_blocks]
        partial_images = []
        for node in selected_for_proc:
            data_block = get_or_reconstruct(node)
            img = raycast_block(data_block, view_params)
            overlapping_procs = get_overlapping_processors(proc_id, node.screen_projection, tiles)
            for p_id in overlapping_procs:
                send_image(p_id, img)
        final_tile = composite_images(tiles[proc_id], received_images)
        final_tiles.append(final_tile)
    final_image = assemble_tiles(final_tiles)
    return final_image

def traverse_front_to_back(node, view_params, error_tolerance, selected_nodes):
    if node.error <= error_tolerance:
        selected_nodes.append(node)
        return
    sorted_children = sort_children_by_depth(node.children, view_params)
    for child in sorted_children:
        traverse_front_to_back(child, view_params, error_tolerance, selected_nodes)

def get_or_reconstruct(node):
    if node in cache.memory:
        return cache.memory[node]
    if node in cache.disk:
        data = load_from_disk(node)
        cache.memory[node] = data
        cache.disk.remove(node)
        return data
    low_pass = node.low_pass
    if not node.is_leaf:
        low_pass = node.parent.low_pass
    coeffs = decompress(node.coeffs)
    data_block = inverse_haar_3d_transform(low_pass, coeffs)
    if len(cache.memory) < MAX_MEMORY:
        cache.memory[node] = data_block
    else:
        lru_node = cache.get_lru()
        cache.disk[lru_node] = cache.memory.pop(lru_node)
        cache.memory[node] = data_block
    return data_block

# Helper definitions
def haar_3d_transform(block):
    """Standard 3D Haar wavelet transform with lifting scheme."""
    pass
def compress(coeffs):
    """Run-length and Huffman encoding."""
    pass
def calculate_error(low_pass, block):
    """Calculate approximation error."""
    pass
def group_low_pass(children):
    """Collect low-pass subblocks from 8 children, group into n voxels."""
    pass
def calculate_parent_error(children):
    """Ensure E_parent >= E_child_i."""
    pass
def distribute_along_hilbert_curve(root):
    """Partition tree along Hilbert curve, assign to processors."""
    pass
def partition_screen_projection(bounding_box, num_processors):
    """Partition screen into P tiles."""
    pass
def sort_children_by_depth(children, view_params):
    """Sort children front-to-back based on view direction."""
    pass
def get_overlapping_processors(proc_id, block_proj, tiles):
    """Find processors whose tiles overlap with block projection."""
    pass
def send_image(proc_id, img):
    """Send partial image to processor."""
    pass
def composite_images(tile, images):
    """Composite images for tile."""
    pass
def assemble_tiles(tiles):
    """Assemble final image from tiles."""
    pass
def decompress(coeffs):
    """Run-length and Huffman decoding."""
    pass
def inverse_haar_3d_transform(low_pass, coeffs):
    """Inverse 3D Haar wavelet transform."""
    pass
def raycast_block(data_block, view_params):
    """Standard front-to-back volume raycasting."""
    o, d, step = view_params.origin, view_params.direction, view_params.step
    alpha = 0.0
    color = [0.0, 0.0, 0.0]
    t = 0.0
    while t < view_params.max_depth:
        x = o + t * d
        val = sample_volume(data_block, x)
        a = transfer_function(val)
        color += a * (1.0 - alpha) * val
        alpha += a * (1.0 - alpha)
        t += step
    return color
```

## Subtopics

### [EVERY-2 scheme with greedy optimization](6631-every-2-scheme-with-greedy-optimization.md)
The EVERY-2 scheme with greedy optimization is a parallel multiresolution volume rendering algorithm that uses a greedy strategy to select which wavelet tree nodes store pre-reconstructed data, trading storage space for rendering speed while maintaining load balance across processors. It builds on [Parallel Multiresolution Volume Rendering with Error-Guided Load Balancing](6624-parallel-multiresolution-volume-rendering-with-error-guided-.md) by introducing a selection algorithm that prioritizes nodes for reconstruction based on a benefit function, thereby reducing the runtime reconstruction overhead without significantly increasing communication or storage costs.

## Operating characteristics
**Time Complexity:** Reconstruction cost is bounded by the height of the subtree corresponding to a distribution unit, which is usually small (two or three in experiments) [S2]. Tree traversal is $O(N)$ where $N$ is the number of nodes, but pruning by error tolerance reduces effective $N$ [G].
**Memory Complexity:** Depends on the cache size ($M_{cache}$, $D_{cache}$) and the storage of the wavelet tree skeleton (all processors store the skeleton) [S1][S2].
**Measured Performance:** On 32 1.53GHz AMD Athlon 1800MP processors connected by Myrinet, rendering a 7.5GB 2048 x 2048 x 1920 Richtmyer-Meshkov Instability (RMI) dataset with error tolerance 3,000 took 48.74 seconds (26.35s reconstruction/I/O, 22.36s raycasting, 0.03s composition) [S2]. Parallel CPU utilization was 97.6% for 8 processors, 96.8% for 16 processors, and 87.3% for 32 processors [S2].
**Strengths:** Minimizes run-time communication cost to a minimum; ensures well-balanced workload among processors; allows flexible tradeoff between image quality and rendering speed via error tolerance [S1][S2].
**Limitations:** Requires preprocessing time (about an hour for the RMI dataset on a 2.0GHz Intel Pentium 4) [S2]. Reconstruction overhead exists if data is not cached [S1].
**When to use:** Large 3D datasets (gigabytes) requiring interactive multiresolution visualization on parallel clusters [S1][S2].
**When not to use:** Small datasets where overhead of distribution and tree traversal outweighs benefits; scenarios requiring exact single-resolution rendering without error tolerance control [G].

## Background
Originated by Chaoli Wang, Jinzhu Gao, and Han-Wei Shen from The Ohio State University and Oak Ridge National Laboratory [S1][S2]. An abbreviated version appeared in Eurographics Symposium on Parallel Graphics and Visualization 2004 (EGPGV2004) [S1]. It addresses the challenge of visualizing large-scale scientific data (e.g., Richtmyer-Meshkov Instability) on parallel architectures by combining wavelet-based multiresolution representation with a novel data distribution scheme to eliminate inter-processor dependencies and balance load [S1][S2].

## Sources
[S1] A Parallel Multiresolution Volume Rendering Algorithm for Large Data Sets, https://academicweb.nd.edu/~cwang11/papers/pc05-pmvr.pdf, fetched 2026-10-26
[S2] Parallel Multiresolution Volume Rendering of Large Data Sets with Error-Guided Load Balancing, https://academicweb.nd.edu/~cwang11/papers/pgv04-pmvr.pdf, fetched 2026-10-26
