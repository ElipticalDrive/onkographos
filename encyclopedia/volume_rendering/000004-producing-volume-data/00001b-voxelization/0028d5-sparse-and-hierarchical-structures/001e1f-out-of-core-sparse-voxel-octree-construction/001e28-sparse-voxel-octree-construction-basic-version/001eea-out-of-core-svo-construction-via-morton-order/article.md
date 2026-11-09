# Out-of-Core SVO Construction via Morton Order

Also called: Buffer-flushing SVO construction, Moctree construction, Morton Queueing and SVO Creation

## Overview
This method constructs a Sparse Voxel Octree (SVO) from voxel grid data sorted in Morton order by reading voxels in groups of eight, generating parent nodes, and flushing buffers to disk to manage memory constraints [S2, unofficial]. It differs from point-based global illumination methods (e.g., Tabellion EGSR2011) by leveraging the fact that voxel data spans the whole grid space, which eliminates the need for a top-down search to find parent nodes [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of voxel-attribute pairs (Morton index and attribute values) sorted in Morton order.
*   **Output:** An SVO stored on disk, where the final flushed node is the root [S2, unofficial].

**Data Structures:**
1.  **Morton Queue:** Stores voxel-attribute pairs from the current batch and unprocessed pairs from previous batches [S1]. It allows extraction of voxels usable for SVO creation in the current iteration [S1].
2.  **Stitch Queue Buffer:** Stores intermediate SVO nodes and their attributes required for processing the last immediate parent node on the current hierarchy level [S1].
3.  **Level Buffers:** Buffers storing generated parent voxels at each level [S2, unofficial].

**Parameters:**
*   **Buffer Size:** $8 \times \text{octree\_depth} \times \text{node\_size}$ bits [S2, unofficial].
*   **Subdivision Threshold:** $K_{vox/tri}$, a user-defined parameter for triangle subdivision [S1].

**Steps:**
1.  **Sort and Batch:** Sort triangles by their lower bound Morton index $m_{min}(T_i)$ [S1]. Determine a stop index $m_{e\_min}(T_{j\_min})$ for each batch, which indicates the first voxel in the prior batch that cannot be used for node creation of the SVO [S1].
2.  **Voxelization and Queueing:** For each triangle, apply subdivision if the bounding box area exceeds $K_{vox/tri} \cdot A_{vox}$ [S1]. Generate voxel-attribute pairs and copy them into the Morton queue [S1].
3.  **Extract Processable Voxels:** Extract voxel-attribute pairs from the Morton queue for SVO creation. Extraction stops at the stop index of the current batch [S1].
4.  **Determine Parent Indices:** For each hierarchy level, determine parent indices for all nodes by dividing the Morton index by the child count (8 for SVO) [S1].
5.  **Compact Indices:** Compact parent indices to unique values [S1].
6.  **Parallel Node Creation:** Each GPU thread processes all child nodes of one unique parent index [S1].
    *   Calculate voxel attributes.
    *   Calculate bitmasks: execute a modulo division by 8 for all Morton indices of its child nodes to represent non-empty children with 1 and empty children with 0 [S1].
7.  **Stitch and Finalize:** Maintain intermediate nodes in the stitch queue buffer [S1]. A node can only be finalized if all child nodes have themselves been finalized [S1].
8.  **Flush:** Store generated parent voxels at each level in a buffer [S2, unofficial]. Flush each buffer to disk when it is full [S2, unofficial]. The final node flushed becomes the root node [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
CHILD_COUNT = 8
BUFFER_SIZE_BITS = 8 * octree_depth * node_size

# Data Structures
mortons = [] # List of (morton_index, attributes)
stitch_queue = [] # List of partial nodes waiting for siblings

def process_batch(batch_voxels, stop_index):
    """
    Process a batch of voxels to build SVO levels.
    """
    # 1. Extract processable voxels up to stop_index
    # Note: In a real implementation, mortons would be a persistent queue
    processable = []
    for i, (m, attr) in enumerate(mortons):
        if i >= stop_index:
            break
        processable.append((m, attr))
    
    # Remove processed voxels from the main queue
    del mortons[:len(processable)]
    
    # 2. Determine parent indices for the current level
    # Parent Morton index = Child Morton index // 8
    parent_indices = {}
    for m, attr in processable:
        parent_m = m // CHILD_COUNT
        if parent_m not in parent_indices:
            parent_indices[parent_m] = []
        parent_indices[parent_m].append((m, attr))
        
    # 3. Create nodes for unique parents
    new_level_nodes = []
    for parent_m, children in parent_indices.items():
        bitmask = 0
        node_attrs = compute_attributes(children) # Implementation detail [G]
        
        # Calculate bitmask: 1 for non-empty, 0 for empty
        # Each child's position in the octant is m % 8
        for child_m, _ in children:
            child_pos = child_m % CHILD_COUNT
            bitmask |= (1 << child_pos)
            
        # Create node structure
        node = {
            'morton': parent_m,
            'bitmask': bitmask,
            'attributes': node_attrs,
            'children': children # Keep for stitching if not all finalized
        }
        
        # Check if this node can be finalized
        # A node is finalized if all its children are leaves or fully processed
        # For simplicity, we assume if we are building bottom-up, 
        # we might need to wait for siblings from previous batches in stitch_queue
        if is_finalized(node, stitch_queue):
            new_level_nodes.append(node)
        else:
            stitch_queue.append(node)
            
    # 4. Flush if buffer full
    if len(new_level_nodes) * node_size_bits > BUFFER_SIZE_BITS:
        flush_to_disk(new_level_nodes)
        new_level_nodes = []
        
    return new_level_nodes

def is_finalized(node, stitch_queue):
    """
    Check if all children of a node are finalized.
    This requires checking if sibling nodes exist in the stitch queue
    or have been flushed.
    """
    # Logic to verify all 8 octants are accounted for [G]
    return True # Placeholder for complex stitching logic [G]

def flush_to_disk(nodes):
    """
    Write nodes to disk storage.
    """
    for node in nodes:
        write_node(node)
```

**Edge Cases and Pitfalls:**
*   **Finalization:** A node can only be finalized if all child nodes have themselves been finalized. The stitch queue buffer must maintain intermediate nodes across batches [S1].
*   **Grid Spanning:** The method assumes the octree spans the whole grid space, meaning 8 voxels always share the same parent. This eliminates the need for a top-down search to find the parent node [S2, unofficial].

## Operating characteristics
*   **Memory:** Uses limited memory constraints defined by the buffer size $8 \times \text{octree\_depth} \times \text{node\_size}$ bits [S2, unofficial].
*   **Performance:** Parallelizable on GPU using sorted Morton indices [S1].
*   **Strengths:** No atomic operations required for grid-based parallel voxelization [S1]. No top-down search needed for parent lookup [S2, unofficial].
*   **Limitations:** Requires pre-allocation of memory based on upper bound of voxels generated by triangles [S1].

## Background
This method was presented by Jeroen Baert in 2012 [S2, unofficial]. It is similar to techniques presented by Eric Tabellion at EGSR2011 for point-based global illumination, but adapted for voxel data [S2, unofficial].

## Sources
[S1] Grid-free out-of-core voxelization to sparse voxel octrees on GPU, https://doi.org/10.1145/2790060.2790067, fetched 2026-11-05
[S2] Out-Of-Core construction of Sparse Voxel Octrees - Jeroen Baert's Blog, https://www.forceflow.be/2012/07/24/out-of-core-construction-of-sparse-voxel-octrees/, fetched 2026-10-30 (unofficial)
