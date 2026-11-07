# Out-of-Core Sparse Voxel Octree Construction

Also called: Out-of-core SVO construction, Streaming SVO construction

## Overview
This method constructs a Sparse Voxel Octree (SVO) from a triangle mesh while keeping memory usage logarithmic in the size of the SVO, enabling the processing of scenes larger than available RAM [S1]. It achieves this by processing an intermediate high-resolution 3D voxel grid in Morton order in a streaming manner [S1]. Unlike [Brute Force SVO Construction](1704-brute-force-svo-construction.md) or [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md) which may require storing large intermediate structures or the full tree in memory, this method ensures that the input is consumed and output is produced in a streaming fashion with logarithmic space complexity [S1]. It differs from [Out-of-Core Octree Construction](9038-out-of-core-octree-construction.md) by building the hierarchy bottom-up from a Morton-ordered voxel grid rather than top-down from primitive partitions [S1, S1].

## How it works
The algorithm consists of two sequential subprocesses: (1) voxelization, which consumes a triangle mesh and produces an intermediate high-resolution 3D voxel grid in Morton order, and (2) SVO construction, which consumes the voxel grid and produces the output SVO [S1].

### Inputs and Outputs
*   **Input**: A triangle mesh (with optional vertex attributes for variants) and the available main memory size.
*   **Output**: A serialized Sparse Voxel Octree (SVO) written to disk or memory stream.
*   **Intermediate**: A high-resolution 3D voxel grid, stored on disk in Morton order.

### Data Structures
*   **Subgrid Partition**: A list of triangle mesh subsets assigned to specific spatial subgrids. Stored temporarily on disk.
*   **Level Queues**: An array of queues, one for each level of the octree. Each queue holds at most $2^d$ nodes (8 for 3D) [S1].
*   **Node**: Represents a voxel or internal octree node. Contains its Morton code and child pointers/indices.

### Parameters
*   **Grid Resolution**: The resolution of the intermediate high-resolution 3D voxel grid. Must be a power of two to ensure subgrids correspond to contiguous ranges in Morton order [S1].
*   **Subgrid Size**: Determined by regularly subdividing the high-resolution 3D voxel grid until a subgrid fits in available memory [S1].

### Steps

#### 1. Voxelization Subprocess
This stage produces the intermediate high-resolution 3D voxel grid in Morton order.

1.  **Partitioning**: Partition the triangle mesh according to subgrids of the high-resolution 3D voxel grid. Test each triangle against the bounding box of every subgrid. Store the resulting triangle mesh partitions temporarily on disk [S1].
2.  **Subgrid Sizing**: Determine subgrid size by regularly subdividing the high-resolution 3D voxel grid until a subgrid fits in available memory. Use only power-of-two resolutions for the high-resolution 3D voxel grid [S1].
3.  **Actual Voxelization**: Sequentially process each subgrid. For each subgrid, consume its corresponding triangle mesh partition in a streaming manner. Voxelize triangles using the 6-separability surface voxelization variant of Huang et al. [1998] [S1]. Output only the Morton codes of non-empty cells, not boolean values for all cells [S1].
4.  **Output Order**: Ensure subgrids are processed in Morton order so that the output of the voxelization subprocess is the intermediate high-resolution 3D voxel grid in Morton order [S1].

#### 2. SVO Construction Subprocess
This stage builds the octree from the Morton-ordered voxel grid using a bottom-up approach. The key insight is that Morton order corresponds to a post-order depth-first traversal of the nodes of the octree [S1].

1.  **Initialize Queues**: Create an array of queues, one for each level of the octree. The maximum number of nodes per queue is $2^d$ (8 for 3D) [S1].
2.  **Read Leaf Nodes**: Read cells with Morton codes corresponding to the current level's leaf nodes from the input and store them in the queue for that level [S1].
3.  **Create Parent Nodes**: When a parent node's children (all nodes in the current level's queue) are processed, create an internal node with the parent's Morton code. Store this internal node in the queue of the parent's level. Record the parent-child relation. Write all non-empty nodes currently in the child's queue to the output. Clear the child's queue [S1].
4.  **Repeat**: Repeat reading leaf nodes and creating parent nodes level by level until the root is processed. Finally, write the root node to output and clear the root's queue [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def out_of_core_svo_construction(triangle_mesh, available_memory):
    """
    Constructs an SVO from a triangle mesh using out-of-core processing.
    """
    # Step 1: Voxelization
    # Determine subgrid size based on available memory
    subgrid_size = determine_subgrid_size(available_memory)
    
    # Partition mesh into subgrids
    subgrid_partitions = partition_mesh_by_subgrids(triangle_mesh, subgrid_size)
    
    # Voxelize each subgrid in Morton order
    intermediate_grid_stream = []
    for subgrid in sorted(subgrid_partitions, key=lambda s: s.morton_start_code):
        # Voxelize triangles in the partition
        # Using 6-separability surface voxelization (Huang et al. [1998])
        voxel_codes = voxelize_triangles(subgrid.triangles, subgrid.bounds)
        
        # Output only non-empty cells in Morton order
        for code in sorted(voxel_codes):
            intermediate_grid_stream.append(code)
            
    # Step 2: SVO Construction
    # Initialize queues for each level
    # Max nodes per queue is 8 (2^3) for 3D
    num_levels = log2(subgrid_size) + 1
    queues = [Queue() for _ in range(num_levels)]
    
    # Process the intermediate grid stream
    # The stream is already in Morton order (post-order traversal)
    current_leaf_code = 0
    leaf_queue_index = num_levels - 1
    
    # We process nodes level by level. 
    # Since Morton order is post-order, we encounter leaves first, then their parents.
    
    # Helper to get parent Morton code from child code
    def get_parent_code(child_code, level):
        # Remove the lowest 3 bits (or appropriate bits for the level)
        # In Morton order, siblings are contiguous.
        # Parent code is child_code >> 3 (for leaf to level-1)
        # More generally, shift by 3 * (leaf_level - current_level)
        shift = 3 * (num_levels - 1 - level)
        return child_code >> shift

    # Process leaf nodes
    leaf_level = num_levels - 1
    leaf_queue = queues[leaf_level]
    
    for code in intermediate_grid_stream:
        leaf_queue.enqueue(code)
        
        # Check if we have a full set of siblings (8 nodes)
        # In Morton order, siblings are contiguous.
        # We can detect a full parent when the current code is the last sibling
        # or by counting nodes in the queue.
        if leaf_queue.size() == 8:
            # Create parent node
            parent_code = get_parent_code(leaf_queue.front(), leaf_level)
            parent_node = Node(morton_code=parent_code)
            
            # Record parent-child relation (implementation detail)
            record_parent_child(parent_node, leaf_queue)
            
            # Write non-empty leaf nodes to output
            while not leaf_queue.is_empty():
                leaf_node = leaf_queue.dequeue()
                write_to_output(leaf_node)
                
            # Enqueue parent to next level
            queues[leaf_level - 1].enqueue(parent_node)
            
    # Process higher levels
    for level in range(leaf_level - 1, -1, -1):
        queue = queues[level]
        next_queue = queues[level - 1] if level > 0 else None
        
        while not queue.is_empty():
            # Collect 8 nodes to form a parent
            siblings = []
            for _ in range(8):
                if not queue.is_empty():
                    siblings.append(queue.dequeue())
                else:
                    # This case should not happen if the tree is complete
                    # or if we handle partial parents correctly.
                    # For sparse trees, we might have fewer than 8 children.
                    # The algorithm assumes full octants for simplicity in basic version.
                    break
            
            if len(siblings) == 8:
                parent_code = get_parent_code(siblings[0].morton_code, level)
                parent_node = Node(morton_code=parent_code)
                record_parent_child(parent_node, siblings)
                
                # Write children to output
                for sibling in siblings:
                    write_to_output(sibling)
                
                if level > 0:
                    next_queue.enqueue(parent_node)
                else:
                    # Root node
                    write_to_output(parent_node)
            else:
                # Handle partial parents if necessary
                # For basic version, we assume full octants or handle sparse leaves
                # by writing them directly if they are the only children.
                # This part depends on specific sparse handling.
                # For simplicity, we assume the input grid covers the full bounding box
                # and we only output non-empty nodes.
                # If a parent has fewer than 8 children, it is still an internal node
                # if it has at least one child.
                pass

    # Finalize output
    close_output_stream()
```

### Edge Cases and Pitfalls
*   **Precision**: Morton codes can become large for high-resolution grids. Use 64-bit integers for Morton codes to avoid overflow [S1].
*   **Boundaries**: Ensure the bounding box of the triangle mesh is given or computed in a streaming manner [S1].
*   **Degenerate Input**: The algorithm assumes the input is a triangle mesh. Non-manifold geometry or degenerate triangles may require preprocessing [S1].
*   **Memory**: The subgrid size must be carefully chosen to fit in available memory. If the subgrid is too large, the algorithm will fail. If too small, performance may degrade due to excessive I/O [S1].

## Subtopics

### [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md)
Sparse Voxel Octree Construction (Basic Version) builds a hierarchical Sparse Voxel Octree (SVO) from a stream of voxelized data, typically Morton codes, by maintaining a set of queues per depth level and merging nodes when queues fill up. It differs from its siblings, such as [Brute Force SVO Construction](1704-brute-force-svo-construction.md), by processing the input in a streaming, bottom-up manner that avoids storing the full octree in memory, and from [Sparse Voxel Octree Construction via Morton Code Sorting](2227-sparse-voxel-octree-construction-via-morton-code-sorting.md) by not requiring a global sort of all points before construction. Trade-offs include higher memory usage compared to optimized variants due to processing empty nodes, but it offers a straightforward implementation for out-of-core construction.

### [Morton-Partitioned Triangle Voxelization](7460-morton-partitioned-triangle-voxelization.md)
Morton-Partitioned Triangle Voxelization is a streaming voxelization pipeline that constructs a Sparse Voxel Octree (SVO) from a triangle mesh by processing a high-resolution 3D grid in Morton order. It partitions the voxel grid into subgrids, assigns triangles to subgrids via axis-aligned bounding box (AABB) overlap, and marks voxels using triangle-box intersection tests. It sets itself apart from [Sparse Voxel Octree Construction (Basic Version)](1694-sparse-voxel-octree-construction-basic-version.md) by managing memory usage logarithmically through out-of-core partitioning, allowing the construction of large SVOs that exceed available RAM.

### [Out-of-Core Octree Construction](9038-out-of-core-octree-construction.md)
An incremental out-of-core algorithm to build an octree where leaves contain the geometry of a given dataset, designed for datasets larger than main memory. It partitions the model into manageable pieces stored on disk, allowing the octree to be constructed directly on a machine with small memory without holding the entire dataset in main memory. Unlike in-core brute-force approaches or algorithms that build from leaves upward, this method builds from the root downward, splitting leaves that exceed a primitive count limit, and ensures that each section insertion reads any affected node at most once.

### Out-of-Core Voxelization with Appearance Data
An extension of the out-of-core SVO construction that handles appearance data (colors, normals, texture coordinates) by interpolating vertex attributes during voxelization and propagating them to higher levels of the octree, provided the data is available locally [S1].

### Sparse-Voxel-Octree (SVO) Voxelization
A variant of voxel cone tracing that uses a Sparse-Voxel-Octree (SVO) instead of a regular 3D texture to store lighting data, reducing GPU memory usage by dynamically instantiating nodes [S2, unofficial]. The voxelization procedure for SVO undergoes three rasterization passes, once for each spatial axis [S2, unofficial]. The SVO structure is a tree-like hierarchy where each internal node has exactly eight children (octants). One node represents a voxel and its children represent the subdivision of that voxel into eight others [S2, unofficial]. Storing lighting data in a sparse octree is significantly more challenging to implement and navigate compared to a regular grid [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: The voxelization process is linear in the number of triangles and the number of voxels. The SVO construction process is linear in the number of non-empty voxels.
*   **Space Complexity**: The SVO construction process has a space complexity that is logarithmic in the size of the sparse voxel octree [S1]. The voxelization process requires memory proportional to the subgrid size, which is determined by available memory.
*   **Measured Performance**: The method was implemented in C++ and tested on a Dell Precision T7500 workstation with an Intel Xeon X5650 CPU and a 7200 RPM SATA II disk. It required approximately 1000 lines of code. The resolution of the intermediate high-resolution 3D voxel grid was set to $2048^3$, requiring 8 GB of memory using a representation of 1 byte per grid cell [S1].
*   **Strengths**: Handles extremely large triangle meshes; logarithmic memory usage; streaming I/O.
*   **Limitations**: Requires power-of-two resolutions; depends on efficient voxelization algorithm (Huang et al. [1998]); may have higher overhead due to I/O compared to in-core methods for small scenes.
*   **When to Use**: When the scene size exceeds available RAM; when building SVOs for large-scale rendering.
*   **When Not to Use**: For small scenes where in-core methods are faster and simpler.

## Background
The method was introduced by Bekaert et al. in 2013 [S1]. It addresses the problem of constructing sparse voxel octrees for very large and highly detailed scenes, where traditional in-core algorithms fail due to memory constraints. The key insight is that the sparse voxel octree construction process can perform its task in a streaming manner with a space complexity that is logarithmic in the size of the sparse voxel octree if the intermediate high-resolution 3D voxel grid is organized in Morton order [S1]. This effectively makes the sparse voxel octree construction process out-of-core: The input is consumed and the output is produced in a streaming manner, and the amount of memory required is logarithmic in the in- and output [S1].

## Sources
[S1] Out-of-Core Construction of Sparse Voxel Octrees, https://graphics.cs.kuleuven.be/publications/BLD13OCCSVO/BLD13OCCSVO_paper.pdf, fetched 2026-10-27
[S2] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
