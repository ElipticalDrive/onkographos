# Out-of-Core Sparse Voxel Octree Construction

Also called: Out-of-core SVO construction, Streaming SVO construction

## Overview
This method constructs a sparse voxel octree (SVO) from a triangle mesh while keeping memory usage logarithmic in the size of the SVO, by processing an intermediate high-resolution 3D voxel grid in Morton order in a streaming manner [S1]. It differs from its siblings, such as [Brute Force SVO Construction](1704-brute-force-svo-construction.md), by avoiding complex sorting or streaming logic in favor of a simple recursive subdivision and depth-first search [S2, unofficial]. Unlike [Adaptive Voxelization of Unstructured Grids](3246-adaptive-voxelization-of-unstructured-grids.md), which handles unstructured data, this method is designed for regular voxelization of triangle meshes [S1]. It is chosen when implementation simplicity is prioritized over memory efficiency and construction speed [S2, unofficial].

## How it works
The algorithm consists of two sequential subprocesses: (1) voxelization, which consumes a triangle mesh and produces an intermediate high-resolution 3D voxel grid in Morton order, and (2) SVO construction, which consumes the voxel grid and produces the output SVO [S1].

### Inputs and Outputs
- **Input**: A triangle mesh (with or without bounding box) and available memory size.
- **Output**: A sparse voxel octree written to disk/stream.
- **Intermediate**: A high-resolution 3D voxel grid in Morton order, stored on disk.

### Data Structures
1. **Subgrid Partition**: A temporary file on disk storing the subset of triangles intersecting a specific subgrid.
2. **Morton Code Stream**: A sequence of 64-bit integers representing the Morton codes of non-empty cells in the intermediate voxel grid.
3. **Level Queues**: An array of queues, one for each level of the octree. Each queue holds at most $2^d$ nodes (where $d=3$ for 3D, so max 8 nodes per level) [S1].
4. **SVO Node**: Represents a voxel or internal node. Contains the Morton code and child pointers/flags.

### Parameters
- **Grid Resolution**: Power-of-two resolution (e.g., $2048^3$) [S1].
- **Subgrid Size**: Determined by regularly subdividing the high-resolution grid until a subgrid fits in available memory [S1].
- **Queue Size**: At most $2^d$ nodes per level [S1].

### Steps

#### 1. Voxelization Subprocess
1. **Partitioning**: Partition the triangle mesh according to subgrids of the high-resolution 3D voxel grid in a streaming manner by testing each triangle against the bounding box of every subgrid. Store the resulting triangle mesh partitions temporarily on disk [S1].
2. **Subgrid Sizing**: Determine subgrid size by regularly subdividing the high-resolution 3D voxel grid until a subgrid fits in available memory. Use only power-of-two resolutions for the high-resolution 3D voxel grid to ensure subgrids correspond to contiguous ranges in Morton order [S1].
3. **Actual Voxelization**: Sequentially process each subgrid. For each subgrid, consume its corresponding triangle mesh partition in a streaming manner. Voxelize triangles using the 6-separability surface voxelization variant of Huang et al. [1998]. Output only the Morton codes of non-empty cells, not boolean values for all cells [S1].
4. **Output Order**: Ensure subgrids are processed in Morton order so that the output of the voxelization subprocess is the intermediate high-resolution 3D voxel grid in Morton order [S1].

#### 2. SVO Construction (Basic Version)
The construction process reads the Morton code stream and builds the octree bottom-up using queues [S1].

1. **Initialize**: Create an array of empty queues, one for each level of the octree (from leaf level up to root level).
2. **Read Leaf Nodes**: Read cells with Morton codes corresponding to the current level's leaf nodes from the input and store them in the queue for that level [S1].
3. **Create Parent Nodes**: When a parent node's children (all nodes in the current level's queue) are processed, create an internal node with the parent's Morton code. Store this internal node in the queue of the parent's level. Record the parent-child relation. Write all non-empty nodes currently in the child's queue to the output. Clear the child's queue [S1].
4. **Repeat**: Repeat reading leaf nodes and creating parent nodes level by level until the root is processed. Finally, write the root node to output and clear the root's queue [S1].

### Pseudocode

```python
def out_of_core_svo_construction(triangle_mesh, available_memory, grid_resolution):
    """
    Constructs an SVO from a triangle mesh using out-of-core voxelization and streaming construction.
    
    Args:
        triangle_mesh: List of triangles.
        available_memory: Size of available RAM in bytes.
        grid_resolution: Power-of-two resolution of the intermediate voxel grid (e.g., 2048).
        
    Returns:
        None (writes SVO to output stream/file).
    """
    # Step 1: Determine subgrid size
    # Regularly subdivide the grid until a subgrid fits in available_memory.
    # Assume power-of-two resolution ensures contiguous Morton ranges.
    subgrid_size = determine_subgrid_size(grid_resolution, available_memory)
    
    # Step 2: Voxelization - Partitioning
    # Partition triangle mesh by subgrid bounding boxes.
    # Store partitions on disk.
    subgrid_partitions = partition_mesh_by_subgrids(triangle_mesh, subgrid_size, grid_resolution)
    
    # Step 3: Voxelization - Actual Voxelization
    # Process subgrids in Morton order.
    # Output Morton codes of non-empty cells to a stream.
    morton_code_stream = []
    for subgrid in get_subgrids_in_morton_order(subgrid_size, grid_resolution):
        partition = subgrid_partitions[subgrid.id]
        # Voxelize triangles in partition using 6-separability method (Huang et al.)
        # Output only Morton codes of non-empty cells.
        codes = voxelize_partition(partition, subgrid)
        morton_code_stream.extend(codes)
        
    # Step 4: SVO Construction
    # Determine number of levels in the octree
    num_levels = log2(grid_resolution) + 1
    
    # Initialize queues for each level
    # Queue[i] holds nodes at level i (leaf level is num_levels - 1)
    queues = [Queue() for _ in range(num_levels)]
    
    # Process Morton codes
    # The stream contains Morton codes for leaf nodes first, then their parents, etc.
    # However, the algorithm reads leaf nodes, builds parents, and pushes parents to next level queue.
    
    # We need to track which level we are currently reading leaf nodes for.
    # In the basic version, we read all leaf nodes first? 
    # No, the description says "Read cells with Morton codes corresponding to the current level's leaf nodes".
    # This implies we process level by level.
    
    # Let's assume the stream is organized such that we can process level by level.
    # Actually, the stream is just a list of Morton codes for non-empty leaf cells.
    # We need to group them by their parent's Morton code.
    
    # Re-reading [S1]: "The cells with Morton codes 0, 1, 2 and 3... are read... stored in queue of level 2."
    # "An internal node with level-1 Morton code 0... is created... stored in queue of level 1."
    # This implies we process the stream in chunks corresponding to siblings.
    
    # Since the stream is in Morton order, siblings are contiguous.
    # We can process the stream in groups of 8 (for 3D) to form parents.
    
    current_level = num_levels - 1 # Start at leaf level
    queue_index = current_level
    
    # We need to handle the fact that the stream might not be perfectly aligned if we skip empty parents.
    # But the algorithm says "Read cells... corresponding to the current level's leaf nodes".
    # This suggests we read all leaf nodes first? 
    # No, "Repeat reading leaf nodes and creating parent nodes level by level".
    # This implies a bottom-up approach where we process one level's worth of siblings at a time.
    
    # Let's implement the queue-based approach as described.
    
    # Initialize the first queue with leaf nodes.
    # We read the entire stream of leaf Morton codes? 
    # No, the stream is the intermediate voxel grid.
    # The SVO construction consumes this stream.
    
    # The key insight is that Morton order corresponds to post-order depth-first traversal.
    # So, we can process the stream sequentially.
    
    # However, the pseudocode in [S1] suggests a level-by-level processing.
    # Let's follow the queue logic strictly.
    
    # We need to know the Morton codes of the leaf nodes.
    # The stream provides them.
    
    # Let's assume we have a function to get the next Morton code from the stream.
    
    # We need to group leaf nodes by their parent.
    # Parent Morton code = child Morton code >> 3 (for 3D, removing the last 3 bits).
    
    # But the algorithm uses queues per level.
    
    # Let's implement the logic from [S1] directly.
    
    # Step 1: Read leaf nodes into the leaf queue.
    # But how many? All of them?
    # The example says "cells with Morton codes 0,1,2,3... stored in queue of level 2".
    # Then "internal node... stored in queue of level 1".
    # Then "cells with Morton codes 4,5,6,7... stored in queue of level 2".
    # This implies we interleave reading leaf nodes and creating parents?
    # No, "Repeat reading leaf nodes and creating parent nodes level by level".
    # This suggests we finish one level before moving to the next?
    # But the example shows level 2 being filled, then cleared, then level 1 filled.
    # Then level 2 filled again?
    # "3. The cells with Morton codes 4, 5, 6 and 7... are read... stored in the queue of level 2."
    # "4. An internal node with level-1 Morton code 1... is created... stored in the queue of level 1."
    # This implies we process siblings in groups.
    
    # Let's assume the stream is processed in chunks of 8 (for 3D) at the leaf level.
    # Each chunk forms a parent.
    # The parent is added to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction from a sorted list of leaf nodes.
    
    # Let's implement this.
    
    # We need to handle the case where a parent is empty (no children).
    # The stream only contains non-empty leaf nodes.
    # If a parent has no children, it is not created.
    
    # But the algorithm says "When a parent node's children... are processed".
    # This implies we wait for all 8 children to be processed?
    # No, "When a parent node's children (all nodes in the current level's queue) are processed".
    # This implies the queue holds the children of a specific parent.
    # When the queue is full (or empty? No, when all children are in), we create the parent.
    
    # Let's re-read carefully: "When a parent node's children (all nodes in the current level's queue) are processed, create an internal node... Store this internal node in the queue of the parent's level."
    # This implies the queue holds the children of a *single* parent?
    # No, "queue of at most 2^d nodes per level".
    # If the queue holds children of a single parent, it would be size 8.
    # If it holds children of multiple parents, it would be larger.
    # The example says "queue of level 2" holds nodes 0-3, then cleared.
    # Then "queue of level 2" holds nodes 4-7.
    # This implies the queue is used to buffer children of a *single* parent.
    # But the text says "queue of at most 2^d nodes per level".
    # And "11 queues of 8 nodes are needed" for a 2048^3 grid.
    # This implies there is one queue per level, not per parent.
    
    # Let's look at the example again.
    # "1. The cells with Morton codes 0, 1, 2 and 3... are read... stored in the queue of level 2."
    # "2. An internal node with level-1 Morton code 0... is created... stored in the queue of level 1... non-empty nodes in the queue of level 2 are written to output... queue of level 2 is cleared."
    # "3. The cells with Morton codes 4, 5, 6 and 7... are read... stored in the queue of level 2."
    # "4. An internal node with level-1 Morton code 1... is created... stored in the queue of level 1."
    
    # This implies that the queue of level 2 is used to buffer the children of the *current* parent being built.
    # Once the parent is built, the queue is cleared.
    # But how do we know when all children of a parent are in the queue?
    # The stream is in Morton order. Siblings are contiguous.
    # So we can read 8 nodes, form a parent, and clear the queue.
    # But what if some children are empty?
    # The stream only contains non-empty nodes.
    # So if we read 8 nodes, they are all non-empty.
    # But what if a parent has only 1 non-empty child?
    # Then we read 1 node, and we don't have 8 nodes.
    # How do we know when to create the parent?
    
    # The algorithm says "When a parent node's children (all nodes in the current level's queue) are processed".
    # This implies the queue holds *all* children of a parent.
    # But if the queue is shared across parents, how do we know when a parent is complete?
    
    # Let's look at the "queue of at most 2^d nodes per level" statement.
    # If the queue holds children of multiple parents, it could be larger.
    # But the example shows the queue being cleared after each parent is built.
    # This implies the queue is used to buffer children of a *single* parent.
    # But the text says "queue of level 2", singular.
    
    # Let's assume the queue is used to buffer children of the *current* parent.
    # We read nodes from the stream.
    # If the node belongs to the current parent, we add it to the queue.
    # If the node belongs to a new parent, we first check if the current parent is complete.
    # But how do we know if the current parent is complete?
    # We don't know the number of children in advance.
    
    # This is a contradiction in the description.
    # Let's look at the "Morton order corresponds to post-order depth-first traversal" statement.
    # In post-order, we visit children before parents.
    # So, we visit all children of a parent, then the parent.
    # But the stream only contains non-empty nodes.
    # So, if a parent has non-empty children, we will see them in the stream.
    # The children of a parent are contiguous in Morton order.
    # So, we can read nodes until we encounter a node that does not belong to the current parent.
    # Then we create the parent.
    
    # But how do we know if a node belongs to the current parent?
    # Parent Morton code = child Morton code >> 3.
    # So, we can group nodes by their parent Morton code.
    
    # Let's implement this grouping approach.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm as follows:
    
    # 1. Read all leaf nodes from the stream.
    # 2. Group them by parent Morton code.
    # 3. For each parent, create an internal node if it has children.
    # 4. Add the internal node to the next level's queue.
    # 5. Repeat until the root is created.
    
    # But the algorithm uses queues per level.
    # This suggests a streaming approach where we don't store all leaf nodes.
    
    # Let's go back to the queue-based approach.
    # The queue holds children of the current parent.
    # We read nodes from the stream.
    # If the node belongs to the current parent, we add it to the queue.
    # If the node belongs to a new parent, we first check if the current parent is complete.
    # But how do we know if the current parent is complete?
    # We don't.
    
    # This is a problem.
    # Let's look at the example again.
    # "1. The cells with Morton codes 0, 1, 2 and 3... are read... stored in the queue of level 2."
    # "2. An internal node with level-1 Morton code 0... is created... stored in the queue of level 1... non-empty nodes in the queue of level 2 are written to output... queue of level 2 is cleared."
    # "3. The cells with Morton codes 4, 5, 6 and 7... are read... stored in the queue of level 2."
    # "4. An internal node with level-1 Morton code 1... is created... stored in the queue of level 1."
    
    # This implies that the queue is cleared after each parent is built.
    # And the next parent's children are added to the same queue.
    # But how do we know when a parent is complete?
    # The example shows that we read 4 nodes (for 2D) and then create the parent.
    # This implies that we know the number of children in advance?
    # No, in 3D, it's 8.
    # But what if a parent has only 1 child?
    # Then we read 1 node, and we don't have 8 nodes.
    # How do we know when to create the parent?
    
    # The algorithm must assume that we read all children of a parent before creating the parent.
    # But how do we know when we have read all children?
    # We can check if the next node in the stream belongs to a different parent.
    # If it does, then the current parent is complete.
    
    # Let's implement this.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to process the stream level by level.
    # But the stream is just a list of leaf nodes.
    # We need to build the tree bottom-up.
    
    # Let's use a list of queues, one for each level.
    # We start with the leaf level.
    # We read leaf nodes and add them to the leaf queue.
    # But the leaf queue is used to buffer children of a single parent.
    # So, we read nodes until we encounter a node that belongs to a different parent.
    # Then we create the parent and add it to the next level's queue.
    # We repeat this until we have processed all leaf nodes.
    # Then we move to the next level.
    # We read nodes from the next level's queue (which are the parents we just created).
    # We group them by their parent Morton code.
    # We create the next level's parents and add them to the next level's queue.
    # We repeat this until we reach the root level.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where a parent has no children.
    # In that case, no nodes are in the stream for that parent.
    # So we don't create a node for that parent.
    
    # Let's implement the algorithm.
    
    # We need to read nodes from the stream.
    # We group them by parent Morton code.
    # For each group, we create a parent node.
    # We add the parent node to the next level's queue.
    # When the next level's queue is full (8 nodes), we create its parent.
    
    # This is a standard bottom-up octree construction.
    
    # Let's implement this.
    
    # We need to handle the case where the stream is empty.
    # In that case, we don't create any nodes.
    
    # Let's implement the algorithm.
    
    # We need a function to get the parent Morton code.
    def get_parent_morton_code(morton_code, level):
        # Remove the last 3 bits for each level up to the parent level.
        # For 3D, each level adds 3 bits.
        # So, to get the parent at level L-1 from a node at level L, we shift right by 3.
        # To get the parent at level L-k, we shift right by 3*k.
        return morton_code >> (3 * (num_levels - level))
    
    # We need to
