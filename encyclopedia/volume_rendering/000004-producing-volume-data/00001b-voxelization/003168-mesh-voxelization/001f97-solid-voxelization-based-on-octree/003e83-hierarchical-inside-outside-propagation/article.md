# Hierarchical Inside/Outside Propagation

Also called: Solid Voxelization Propagation, X-Direction Bit Flip Propagation

## Overview
Hierarchical Inside/Outside Propagation is a post-processing step for solid voxelization that corrects the inside/outside status of voxels across an octree structure by propagating boundary crossings along the +x direction. It solves the problem of inconsistent solid status between adjacent nodes that arises when voxelizing a mesh into an octree, ensuring that the final representation correctly distinguishes interior from exterior space. It differs from [Solid Voxelization via Ray Casting](6829-solid-voxelization-via-ray-casting.md) by avoiding per-voxel ray queries for consistency, instead relying on local neighbor relationships and hierarchical aggregation to determine global solid status.

## How it works
The method operates on an octree where level-0 nodes contain a sub-grid of voxels (typically $4 \times 4 \times 4$, totaling 64 voxels per node). Each node maintains an `inside` flag and, for levels $>0$, a `flip` propagation flag. The algorithm proceeds in two main phases: upward propagation of flip information from level-0 to the root, and downward propagation of the final inside status from the root to level-0.

### Inputs and Outputs
*   **Input**: An octree structure where level-0 nodes have been voxelized (bits set for occupied voxels). Each node has pointers to its +x neighbor (if it exists) and its children (if it is not a leaf).
*   **Output**: The `inside` flag of every node is correctly set. For level-0 nodes, the `inside` flag applies to the entire node; for internal nodes, it applies to the node itself if it has no children, or triggers a flip of children's flags if it has children.

### Data Structures
*   **Node**:
    *   `level`: Integer, $0 \le level \le L_{max}$.
    *   `inside`: Boolean, initial value from coarse ray casting or zero.
    *   `flip`: Boolean, used only for levels $>0$. Initial value `false`.
    *   `children`: Array of 8 child nodes (if not leaf).
    *   `x_neighbor`: Pointer to the immediate +x neighbor node at the same level (if it exists).
    *   `sg_voxels`: Bitmask or array of 64 bits for level-0 nodes.

### Steps

1.  **Initialize Flags**: Set `inside = false` and `flip = false` for all nodes.
2.  **Level-0 Flip Propagation**:
    *   Identify "head" nodes: Level-0 nodes that have no -x neighbor (i.e., they are the start of an x-linked string). This can be done by iterating all level-0 nodes and checking if `x_neighbor` exists for the -x direction. [S1, S2]
    *   For each head node, traverse the +x-linked string of level-0 nodes.
    *   For each node in the string, check if the last bit in its SG voxel column (local x index 3, i.e., the rightmost column of the $4 \times 4 \times 4$ sub-grid) is set. [S1, S2]
    *   If the last bit is set, propagate a flip to the next node in the +x string. Specifically, if the current node's last bit is set, the next node's `inside` status must be inverted relative to the previous state. The sources describe this as "propagating bit flips". More precisely, if a node is at the end of an x-linked string and its SG voxels with local x index 3 are set, it signals a boundary crossing. [S1, S2]
    *   The sources state: "If the last bit in a level-0 node's SG voxel column is set, all SG voxels in the same column belonging to nodes in +x direction must be flipped." This implies that the `inside` flag of subsequent nodes in the chain is toggled. [S1, S2]
    *   After traversing the entire string, the nodes at the end of the string (where no more level-0 nodes abut) will have their SG voxels with local x index 3 in agreement for all four level-0 nodes sharing the same level-1 parent. [S1, S2]

3.  **Aggregation to Level-1**:
    *   For each level-1 node, check its children (level-0 nodes).
    *   If any child is at the end of an x-linked node string and its SG voxels with local x index 3 are set, set the level-1 node's `flip` flag to `true`. [S1, S2]
    *   Initialize `flip = false` for all level-1 nodes.

4.  **Level-1 Flip Propagation**:
    *   Identify head nodes at level-1 (nodes with no -x neighbor at level-1).
    *   Traverse +x-linked strings at level-1.
    *   If a node's `flip` flag is set, flip both the `flip` and `inside` flags of its +x neighbor. [S1, S2]
    *   This propagates the boundary crossing information up the hierarchy.

5.  **Repeat for Higher Levels**:
    *   The process is repeated analogously for all remaining levels up to the root. Propagation to the next coarser level and then within this level is performed for each level $i$ from $0$ to $L_{max}-1$. [S1, S2]

6.  **Downward Inside Propagation**:
    *   Start from the root node (level $L_{max}$).
    *   Propagate the `inside` flag information from the top level down to level-0 nodes. [S1, S2]
    *   For each node of level $i$ that has its `inside` flag set and has children:
        *   Flip the `inside` flags of its children. [S1, S2]
        *   If the node is a level-0 node (leaf), flip the SG voxels directly. However, the sources specify: "For each node of level $i$ that has its inside flag set and has children, we flip these child nodes' inside flags or their SG voxels, respectively." This implies that if a parent is inside, its children's status is inverted. [S1, S2]
    *   A level-$i$ node without children is inside the solid object if its `inside` flag is set, and outside otherwise. [S1, S2]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_inside_outside_propagation(octree):
    """
    Propagates inside/outside status hierarchically.
    Assumes octree nodes have: level, inside (bool), flip (bool, for levels > 0),
    x_neighbor (pointer or None), children (list of 8 or None), sg_voxels (bitmask for level 0).
    """
    L_max = octree.root.level
    
    # Step 1: Initialize flags
    for node in octree.all_nodes():
        node.inside = False
        if node.level > 0:
            node.flip = False
            
    # Step 2: Level-0 Flip Propagation
    # Identify head nodes at level 0
    level_0_nodes = [n for n in octree.all_nodes() if n.level == 0]
    head_nodes_l0 = [n for n in level_0_nodes if n.x_neighbor is None or n.x_neighbor.level != 0 or n.x_neighbor is not the -x neighbor] # Simplified: check if no -x neighbor exists
    
    # More robustly: A node is a head if it has no -x neighbor.
    # We assume x_neighbor points to +x. So we need to check if there is a node pointing to us as its +x neighbor?
    # The sources say "loop over all (even-indexed) level-0 nodes... identifying those with no level-0 -x neighbor".
    # Let's assume we can determine -x neighbor existence.
    
    for node in level_0_nodes:
        if has_no_minus_x_neighbor(node):
            # Start of a string
            current = node
            while current is not None:
                # Check if last bit in SG voxel column is set (local x index 3)
                # Assuming sg_voxels is a 64-bit integer, bits 0-63.
                # Local x index 3 corresponds to the rightmost column.
                # In a 4x4x4 grid, x is the fastest varying? Or slowest?
                # Typically Morton order or similar. Let's assume bit mask for x=3 column.
                if is_last_x_column_set(current.sg_voxels):
                    # Propagate flip to next node
                    if current.x_neighbor is not None:
                        # Flip the inside flag of the neighbor
                        current.x_neighbor.inside = not current.x_neighbor.inside
                current = current.x_neighbor

    # Step 3: Aggregation to Level 1
    level_1_nodes = [n for n in octree.all_nodes() if n.level == 1]
    for node in level_1_nodes:
        # Check if any child is at the end of an x-linked string and has last x column set
        for child in node.children:
            if child.level == 0:
                # Check if child is at end of x-string
                if child.x_neighbor is None or child.x_neighbor.level != 0:
                    if is_last_x_column_set(child.sg_voxels):
                        node.flip = True
                        break

    # Step 4: Level 1 Flip Propagation
    head_nodes_l1 = [n for n in level_1_nodes if has_no_minus_x_neighbor(n)]
    for node in head_nodes_l1:
        current = node
        while current is not None:
            if current.flip:
                if current.x_neighbor is not None:
                    current.x_neighbor.flip = not current.x_neighbor.flip
                    current.x_neighbor.inside = not current.x_neighbor.inside
            current = current.x_neighbor

    # Step 5: Repeat for higher levels
    for level in range(1, L_max):
        current_level_nodes = [n for n in octree.all_nodes() if n.level == level]
        next_level_nodes = [n for n in octree.all_nodes() if n.level == level + 1]
        
        # Aggregate flip from level+1 to level
        for node in current_level_nodes:
            node.flip = False
            for child in node.children:
                if child.level == level + 1:
                    if child.x_neighbor is None or child.x_neighbor.level != level + 1:
                        if child.flip:
                            node.flip = True
                            break
        
        # Propagate flip within level
        head_nodes = [n for n in current_level_nodes if has_no_minus_x_neighbor(n)]
        for node in head_nodes:
            current = node
            while current is not None:
                if current.flip:
                    if current.x_neighbor is not None:
                        current.x_neighbor.flip = not current.x_neighbor.flip
                        current.x_neighbor.inside = not current.x_neighbor.inside
                current = current.x_neighbor

    # Step 6: Downward Inside Propagation
    # Start from root
    propagate_inside_down(octree.root)

def propagate_inside_down(node):
    if node.children is None:
        return
    
    for child in node.children:
        if node.inside:
            child.inside = not child.inside
        propagate_inside_down(child)

def has_no_minus_x_neighbor(node):
    # Implementation detail: Check if any node at the same level has x_neighbor == node
    # Or if node has a -x pointer. The sources imply we identify heads by lack of -x neighbor.
    # This requires a reverse lookup or pre-computed -x pointers.
    # For the purpose of this article, we assume this helper exists.
    return True # Placeholder for actual implementation

def is_last_x_column_set(sg_voxels):
    # Assuming 4x4x4 grid, x index 3 is the last column.
    # Bits for x=3 column.
    # This depends on bit packing.
    # Let's assume bits 0-63.
    # If x is the fastest varying, bits 0-3 are x=0, 4-7 x=1, etc.
    # Then x=3 column is bits 3, 7, 11, ...
    # If y is fastest, x=3 column is every 4th bit starting at 3?
    # The sources do not specify bit packing. We assume a helper function.
    return True # Placeholder
```

### Parameters and Edge Cases
*   **Bit Packing**: The definition of "last bit in SG voxel column" depends on the bit packing order. The sources do not specify the exact bit layout, only that it corresponds to local x index 3. [S1, S2]
*   **Boundary Conditions**: Nodes at the boundary of the voxelization domain may not have +x or -x neighbors. The algorithm handles this by checking for `None` neighbors. [S1, S2]
*   **Parallelism**: The identification of head nodes and the traversal of x-linked strings can be parallelized. [S1, S2]

## Operating characteristics
*   **Time Complexity**: Linear in the number of nodes, $O(N)$, where $N$ is the total number of octree nodes. Each node is visited a constant number of times during upward and downward propagation.
*   **Memory Complexity**: $O(N)$ for storing the octree and flags.
*   **Strengths**: Efficient parallel implementation on GPUs. Avoids expensive ray casting for consistency. [S1, S2]
*   **Limitations**: Requires x-linked neighbor pointers, which may increase memory overhead. The correctness depends on the initial voxelization and the correct identification of head nodes.

## Background
The method was introduced in the paper "Fast Parallel Surface and Solid Voxelization on GPUs" by Michael Schwarz et al. (2010). It is part of a broader framework for GPU-based voxelization that includes ray casting for initial inside/outside determination and hierarchical propagation for consistency. [S1, S2]

## Sources
[S1] Fast parallel surface and solid voxelization on GPUs, https://doi.org/10.1145/1882261.1866201, fetched 2026-11-03
[S2] PDF Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
