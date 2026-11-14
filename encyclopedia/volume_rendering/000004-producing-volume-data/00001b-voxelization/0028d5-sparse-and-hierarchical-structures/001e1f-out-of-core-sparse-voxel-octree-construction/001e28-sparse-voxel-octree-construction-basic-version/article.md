# Sparse Voxel Octree Construction (Basic Version)

Also called: Basic SVO Construction, SVO Construction, basic version

## Overview
This method constructs a Sparse Voxel Octree (SVO) from a stream of non-empty voxel Morton codes by maintaining fixed-size queues per depth level, merging nodes when a queue fills up, and writing non-empty nodes to disk [S3]. It differs from its siblings, such as [Brute Force SVO Construction](1704-brute-force-svo-construction.md) and [Sparse Voxel Octree Construction via Morton Code Sorting](2227-sparse-voxel-construction-via-morton-code-sorting.md), by processing the input in a single streaming pass without requiring full in-memory tree storage or pre-sorting [S3]. Unlike the [Optimized Version](1900-sparse-voxel-octree-construction-optimized-version.md), it processes empty space individually, which results in significantly higher processing time for sparse scenes [S3].

## How it works
**Inputs and Outputs:**
- **Input:** A stream of non-empty voxel leaf nodes, sorted by Morton code (Z-order). Each leaf node contains its Morton code and associated data (e.g., color, density) [S3].
- **Output:** A serialized SVO written to disk (or a buffer), where nodes are stored in Morton order to ensure spatial locality of siblings [S3].

**Data Structures:**
- **Queues:** An array of queues, one for each depth level $d$ from $0$ to $max\_depth$. Each queue holds a fixed number of node descriptors. The size of the queue determines the branching factor or buffer size before flushing to disk.
- **Node Descriptor:** Represents a child pointer. It can be an index to a node in the output stream or a special value indicating an empty subtree [S3].
- **Internal Node:** Contains references to its 8 children. In the basic version, these are stored as offsets or indices into the output stream [S3].

**Steps:**
1. Initialize an empty array of queues `queues` for depths $0$ to $max\_depth$.
2. While the input stream is not empty:
   a. Consume the next leaf node $l$ from the input stream [S3].
   b. Push $l$ onto the queue corresponding to its depth $d$ (typically the maximum depth) [S3].
   c. Set a depth counter $d$ to the current depth of the pushed node [S3].
   d. While $d > 0$ and the queue at depth $d$ is full:
      i. Create a new internal node $p$ [S3].
      ii. Set the children of $p$ to the child offsets of the elements currently in `queues[d]` [S3].
      iii. Write the non-empty elements of `queues[d]` to the output stream [S3].
      iv. Clear `queues[d]` [S3].
      v. Push the new internal node $p$ onto `queues[d-1]` [S3].
      vi. Decrement $d$ by 1 [S3].
3. After the input stream is exhausted, flush any remaining nodes in the queues to the output stream, starting from the highest depth down to 0 [S3].
4. Write the root node reference to the output [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_svo_basic(input_stream, max_depth, queue_capacity):
    # Initialize queues for each depth level
    queues = [ [] for _ in range(max_depth + 1) ]
    
    # Output buffer for serialized nodes
    output = []
    
    # Process input stream
    for leaf in input_stream:
        # Push leaf onto the queue at its depth
        queues[leaf.depth].append(leaf)
        
        d = leaf.depth
        
        # Merge nodes while queue is full and not at root
        while d > 0 and len(queues[d]) >= queue_capacity:
            # Create internal node from the full queue
            internal_node = create_internal_node(queues[d])
            
            # Write non-empty children of the queue to output
            # Note: The basic version writes the queue contents to disk
            # before clearing, ensuring siblings are stored together
            write_nodes_to_output(queues[d], output)
            
            # Clear the current queue
            queues[d].clear()
            
            # Push the new internal node to the parent level
            queues[d-1].append(internal_node)
            
            # Move up one level
            d -= 1
            
    # Flush remaining nodes from highest depth to 0
    for d in range(max_depth, -1, -1):
        if queues[d]:
            if d > 0:
                # If not root, create internal nodes from remaining items
                # This step may need to handle partial queues if capacity wasn't met
                # The basic algorithm assumes full queues trigger merges, 
                # so remaining items at the end are pushed up.
                internal_node = create_internal_node(queues[d])
                write_nodes_to_output(queues[d], output)
                queues[d].clear()
                if d-1 >= 0:
                    queues[d-1].append(internal_node)
            else:
                # Root level
                write_nodes_to_output(queues[d], output)
                queues[d].clear()
                
    return output
```

**Parameters:**
- `max_depth`: The maximum depth of the octree. Typical values depend on the voxel grid resolution [S3].
- `queue_capacity`: The number of nodes that can fit in a queue before triggering a merge. This determines the block size for disk writes [S3].

**Edge Cases and Pitfalls:**
- **Empty Space:** The basic version does not explicitly handle empty space between input leaves. It assumes the input stream contains only non-empty voxels. Empty space is implicitly handled by the structure of the octree (missing children) [S3].
- **Queue Size:** If the queue capacity is too small, frequent disk I/O occurs. If too large, memory usage increases [S3].
- **Partial Queues:** At the end of the stream, queues may not be full. The algorithm must still flush these to disk and merge them up to the root [S3].

## Subtopics

### [Brute Force SVO Construction](1704-brute-force-svo-construction.md)
Brute Force SVO Construction is a top-down algorithm that builds a complete Sparse Voxel Octree (SVO) in main memory by recursively subdividing the world space, followed by a depth-first search to prune empty branches. It differs from its siblings, such as [Out-of-Core SVO Construction via Morton Order](1885-out-of-core-svo-construction-via-morton-order.md) and [Sparse Voxel Octree Construction via Morton Code Sorting](2227-sparse-voxel-construction-via-morton-code-sorting.md), by requiring the entire octree structure to reside in memory during construction and relying on post-construction pruning rather than incremental merging or sorting. This approach is chosen for its implementation simplicity and recursive clarity, but is limited by high memory consumption and poor cache coherence due to scattered node allocation.

### [Out-of-Core SVO Construction via Morton Order](1885-out-of-core-svo-construction-via-morton-order.md)
This method constructs a Sparse Voxel Octree (SVO) from voxel grid data sorted in Morton order by reading voxels in groups of eight, generating parent nodes, and flushing buffers to disk to manage memory constraints. It differs from point-based global illumination methods (e.g., Tabellion EGSR2011) by leveraging the fact that voxel data spans the whole grid space, which eliminates the need for a top-down search to find parent nodes.

### [Sparse Voxel Octree Construction (Optimized Version)](1900-sparse-voxel-octree-construction-optimized-version.md)
This method constructs a Sparse Voxel Octree (SVO) from a stream of non-empty voxel Morton codes by optimizing the handling of empty space. Unlike the basic version, which inserts empty child nodes individually, this approach calculates the number of empty nodes between consecutive input leaves and directly inserts empty parent nodes into higher-level queues. This reduces the number of queue operations significantly, as pushing back $2^d$ empty nodes is equivalent to pushing back a single node at the level above. It builds on [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md).

### [Sparse Voxel Octree Construction via Morton Code Sorting](2227-sparse-voxel-octree-construction-via-morton-code-sorting.md)
This method constructs a static Sparse Voxel Octree (SVO) from a set of 3D points or voxels by converting them to Z-order (Morton) codes, sorting them for spatial locality, and recursively partitioning the sorted array into octree nodes and leaf voxels. It differs from dynamic insertion methods by pre-sorting all points into Morton order, which guarantees spatial locality and allows $O(1)$ range queries via binary search instead of tree traversal during construction.

### [Dynamic Octree Growth Algorithm](2375-dynamic-octree-growth-algorithm.md)
The Dynamic Octree Growth Algorithm constructs a Sparse Voxel Octree (SVO) by dynamically increasing its depth and spatial extent as new voxels are processed, rather than pre-calculating the size from the input data. It handles both positive and negative coordinates by centering the root at the boundary between -1 and 0, ensuring the tree grows to encompass all input data without pre-calculating the scene's bounding box. It differs from static voxelization by adapting resolution locally and handling boundary conditions through recursive growth.

### [Sparse Voxel Octree Construction (Queue-Based Collapse)](2457-sparse-voxel-octree-construction-queue-based-collapse.md)
This method constructs a Sparse Voxel Octree (SVO) from a stream of indexed leaves by maintaining a fixed-size array of queues, one per tree depth level, and collapsing homogeneous subtrees into single leaf or invalid nodes to compress the structure. It differs from brute-force or sorting-based builders by processing leaves in a single pass without requiring full in-memory tree storage or pre-sorting.

### [Sparse Voxel Octree (SVO) Builder](2617-sparse-voxel-octree-svo-builder.md)
The Sparse Voxel Octree (SVO) Builder constructs a hierarchical Sparse Voxel Octree from a flat list of voxelized geometry, establishing parent pointers and face-sharing neighbor links to enable efficient traversal. Unlike [Brute Force SVO Construction](1704-brute-force-svo-construction.md) which may use less structured memory layouts, this method explicitly groups voxels into fixed-size leaf blocks and iteratively merges them into a power-of-two bounding volume, ensuring a dense, contiguous array representation suitable for both CPU and GPU implementations. It serves as the foundational construction step for [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md) by providing the initial tree structure and connectivity data.

### SSVDAG Construction Algorithm
The SSVDAG Construction Algorithm is an out-of-core, bottom-up external-memory algorithm that reduces an SVO to a minimal Symmetry-aware Sparse Voxel DAG (SSVDAG) [S4, unofficial]. It operates by alternating phases at each level of the octree hierarchy: clustering nodes representing similar subtrees (identical up to a similarity transform) and replacing them with a single representative node; replacing pointers in the higher level with tagged pointers to the representative, where the tag encodes the required similarity transform; and sorting the representatives by their reference count to facilitate variable-bit-rate encoding of the pointers [S4, unofficial]. The method assumes the original scene has geometric redundancy, with many subtrees similar with respect to a reflective transformation [S4, unofficial].

## Operating characteristics
- **Time Complexity:** The basic version has a time complexity proportional to the number of non-empty voxels plus the number of internal nodes created. However, it spends significant time processing empty nodes implicitly, making it slower than the optimized version [S3]. The optimized version is over two orders of magnitude faster than the non-optimized version because the non-optimized version spends significant time processing empty nodes [S3].
- **Memory Complexity:** The method uses logarithmic memory usage relative to the size of the SVO, as it processes the voxel grid in a streaming manner and flushes nodes to disk [S3]. On a Dell Precision T7500 workstation (Intel Xeon X5650 CPU, 7200 RPM SATA II disk), the out-of-core method using 128 MB or 1 GB memory is roughly as fast as an in-core method using 8 GB memory, but uses up to 64 times less memory [S3].
- **Strengths:** Low memory footprint, suitable for out-of-core processing, simple implementation [S3].
- **Limitations:** Slower than optimized versions due to handling empty space, requires sorted input [S3].
- **When to use:** When memory is constrained and the input is already sorted by Morton code [S3].
- **When not to use:** When performance is critical and the scene has large empty regions, as the optimized version handles this more efficiently [S3].

## Background
The Sparse Voxel Octree Construction (Basic Version) is part of the broader field of out-of-core volume rendering and hierarchical data structures. It was introduced in the context of out-of-core construction of sparse voxel octrees for large-scale volume rendering [S3]. The method builds on the concept of Morton order (Z-order) to ensure spatial locality of voxels, which is crucial for efficient cache usage and disk I/O [S3]. The basic version serves as a foundation for more optimized variants that handle empty space more efficiently [S1, unofficial].

## Sources
[S1] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
[S2] scarlyle0/voxel-compare, https://github.com/scarlyle0/voxel-compare, fetched 2026-10-27 (unofficial)
[S3] Out-of-Core Construction of Sparse Voxel Octrees, https://graphics.cs.kuleuven.be/publications/BLD13OCCSVO/BLD13OCCSVO_paper.pdf, fetched 2026-10-27
[S4] Symmetry-aware Sparse Voxel DAGs (SSVDAGs) for compression-domain tracing of high-resolution geometric scenes, https://albertojaspe.net/publications/2017-JCGT-ssvdags.pdf, fetched 2026-10-24 (unofficial)
