# Bottom-up SVO to DAG Reduction

Also called: SVO to DAG reduction, SVO to DAG transformation, SVO to Directed Acyclic Graph (DAG) Reduction, Sparse Voxel Octree (SVO) to Directed Acyclic Graph (DAG) Reduction, Sparse Voxel DAG Reduction, Reducing a Sparse Voxel Tree to a DAG, SVO reduction

## Overview
Bottom-up SVO to DAG Reduction is a method for compressing a Sparse Voxel Octree (SVO) into a Sparse Voxel Directed Acyclic Graph (SVDAG) by merging identical subtrees into shared nodes [S1, unofficial][S2]. While an SVO efficiently encodes empty regions of space, it stores redundant copies of identical non-empty geometry, leading to high memory consumption at high resolutions [S2]. This method solves that problem by generalizing the tree structure to a DAG, allowing multiple parent nodes to share pointers to the same child instance, thereby encoding identical regions of space efficiently [S2]. It sets itself apart from sibling methods like [Compact pointer tree](844-compact-pointer-tree.md) or [OpenVDB (VDB)](1508-openvdb-vdb.md) by focusing on topological redundancy (identical geometry) rather than just spatial sparsity or pointer compression, achieving lossless compression without decompression overhead during traversal [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A Sparse Voxel Octree (SVO) representing a 3D voxel grid. The SVO may be too large to fit in memory entirely [S1, unofficial][S2].
*   **Output:** A minimal Directed Acyclic Graph (DAG) representing the same geometry with minimal node count.

**Data Structures:**
*   **SVO Node:** Defined by a traversal path from the root [S1, unofficial]. Contains a childmask and pointers to children [S1, unofficial][S2].
*   **DAG Node Layout:**
    *   `childmask`: 8-bit integer encoding the existence of child pointers (bit $i$ is 1 if child $i$ exists, 0 otherwise).
    *   `pointers`: Array of pointers stored consecutively in memory immediately after the mask. Only pointers to non-empty children are stored [S1, unofficial][S2].
    *   **Size:** Between 8 and 36 bytes per node, depending on the number of children (1 to 8) [S1, unofficial][S2].
*   **Indirection Table:** A table mapping each original node's position in a level to its position in the compacted (unique) list of that level [S1, unofficial][S2].

**Algorithm Steps:**
The reduction is performed iteratively from the leaves up to the root [S1, unofficial][S2]. The algorithm is guaranteed to find the smallest possible DAG within $L$ iterations, where $L$ is the maximum level of the octree [S1, unofficial][S2].

1.  **Initialization:** If the SVO is too large to fit in memory, construct the top $L-10$ levels top-down using triangle/cube intersection tests [S1, unofficial]. For each leaf node in this top tree, construct a subtree to the maximum depth using a CPU-assisted algorithm (e.g., depth-peeling) [S1, unofficial]. Reduce each subtree to a DAG independently before inserting it into the top tree, creating a partially reduced DAG [S1, unofficial].
2.  **Leaf Reduction:**
    *   Leaf nodes are uniquely defined by their childmasks (describing 8 voxels) [S1, unofficial][S2]. There are at most $2^8 = 256$ unique leaf nodes [S1, unofficial][S2].
    *   Identify all unique leaf nodes.
    *   Update child-pointers in the level above the leaves to reference these new unique leaf instances [S1, unofficial][S2].
3.  **Iterative Level Reduction:**
    *   For each level $l$ from the leaves up to the root:
        *   Identify all nodes at level $l$.
        *   Two nodes are identical if they have the **same childmask** AND **identical pointers** to their children [S1, unofficial][S2].
        *   **Sorting:** Sort the nodes of level $l$ using their child-pointers (and childmask) as a key. A 256-bit key is used (8 pointers $\times$ 32 bits, or optimized for lower levels) [S1, unofficial][S2].
            *   *Optimization:* For the two lowest levels (where the majority of nodes reside), store subtrees of resolution $4^3$ without pointers in a 64-bit integer to improve sorting performance [S1, unofficial][S2].
        *   **Compaction:** Compact the sorted list to contain only unique nodes. Identical nodes become adjacent in the sorted list [S1, unofficial][S2].
        *   **Indirection:** Maintain a table of indirections. For each original node, store its position in the compacted list [S1, unofficial][S2].
        *   **Pointer Update:** Update the child-pointers of nodes in level $l-1$ using the indirection table to point to the unique instances in level $l$ [S1, unofficial][S2].
    *   Repeat until the root level is processed [S1, unofficial][S2].
4.  **Finalization:** Apply the reduction one last time to the top-SVO (if sub-DAGs were used) to generate the final DAG [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reduce_svo_to_dag(svo_levels):
    """
    Reduces a Sparse Voxel Octree (SVO) to a minimal DAG.
    Input: svo_levels is a list of lists, where svo_levels[l] contains all nodes at level l.
           Level 0 is the root, Level L is the leaves.
    Output: The root node of the resulting DAG.
    """
    num_levels = len(svo_levels)
    
    # Process from leaves (highest level) up to the root (level 0)
    for level in range(num_levels - 1, -1, -1):
        nodes = svo_levels[level]
        
        # Step 1: Sort nodes to group identical ones together
        # Key is the combination of childmask and child pointers
        # In practice, a parallel sort like tbb::parallel_sort is used [S2]
        sorted_nodes = sorted(nodes, key=lambda n: (n.childmask, tuple(n.pointers)))
        
        # Step 2: Compact unique nodes and build indirection map
        unique_nodes = []
        indirection_map = {}  # Maps original node ID to index in unique_nodes
        unique_index = 0
        current_unique = None
        
        for node in sorted_nodes:
            # Check if current node is identical to the last added unique node
            if current_unique is None or (node.childmask == current_unique.childmask and node.pointers == current_unique.pointers):
                current_unique = node
                unique_nodes.append(node)
                indirection_map[node.id] = unique_index
                unique_index += 1
            else:
                # Node is a duplicate, map to the existing unique node
                indirection_map[node.id] = unique_index - 1
        
        # Step 3: Update pointers in the level above
        if level > 0:
            parent_level_nodes = svo_levels[level - 1]
            for parent_node in parent_level_nodes:
                # Update each child pointer to point to the unique instance
                for child_idx in range(8):
                    if parent_node.childmask & (1 << child_idx):
                        child_id = parent_node.pointers[child_idx]
                        new_unique_index = indirection_map[child_id]
                        # Update pointer to reference the unique node's ID/address
                        parent_node.pointers[child_idx] = unique_nodes[new_unique_index].id
        
        # Replace the current level with the unique nodes for the next iteration
        svo_levels[level] = unique_nodes

    return svo_levels[0]
```

**Parameters:**
*   **Max Level ($L$):** The depth of the octree. The algorithm runs in $O(L)$ iterations [S1, unofficial][S2].
*   **Resolution:** Demonstrated at $128K^3$ ($131,072^3$) voxel grids [S2].
*   **Sub-DAG Threshold:** For resolutions over $1K^3$, generate sub-DAGs of that size first to manage memory [S2].

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** The complete SVO may be orders of magnitude larger than the final DAG. The method must support streaming or chunk-based processing (constructing sub-DAGs) to handle scenes that do not fit in RAM or on disk [S1, unofficial][S2].
*   **Attribute Compression:** The method focuses on geometry (visibility). It does not compress material or reflectance properties (color, normal) [S2]. Decoupling attributes is handled by [Voxel Attribute Decoupling](1761-voxel-attribute-decoupling.md) [G].
*   **Precision:** Pointer indirection must be exact to preserve geometry.

## Subtopics

### [Sparse Voxel Directed Acyclic Graph (SVDAG)](962-sparse-voxel-directed-acyclic-graph-svdag.md)
The resulting data structure. It shares identical subtrees, enabling lossless compression and efficient traversal without decompression. Choose this when memory footprint is critical and geometry has high redundancy.

### [Voxel Attribute Decoupling](1761-voxel-attribute-decoupling.md)
A technique to separate attributes from geometry. Use this in conjunction with SVO to DAG reduction to compress attributes independently, as the base reduction method does not handle material properties.

## Operating characteristics
*   **Time Complexity:** The reduction is guaranteed to find the minimal DAG within $L$ iterations [S1, unofficial][S2]. The dominant operation is sorting, which is $O(N \log N)$ per level, where $N$ is the number of nodes at that level [G]. Parallel sorting (e.g., `tbb::parallel_sort`) is used on CPU [S2]. GPU parallelization is suggested using parallel prefix sums for compaction [S1, unofficial][S2].
*   **Memory Complexity:** The final DAG requires significantly less memory than the SVO [S2]. In the EPICTTADEL scene ($128K^3$, 19 billion voxels), the DAG requires 945MB, while the SVO requires 5.1GB (excluding pointers) [S2]. Node count reduction is 1-3 orders of magnitude [S2].
*   **Measured Performance:**
    *   **Reduction Time:** On an Intel Core i7 3930K, reducing a $1K^3$ SVO takes ~33ms, and an $8K^3$ SVO takes ~4.5s [S2].
    *   **Traversal:** On an NVIDIA GTX680, traversal speeds for the EPICTTADEL scene are 170 MRays/sec for ambient occlusion and 240 MRays/sec for shadows [S2].
*   **Strengths:** Lossless compression; no decompression overhead during traversal; handles extremely high resolutions ($>1K^3$) that would otherwise exceed memory limits; effective on irregular data [S1, unofficial][S2].
*   **Limitations:** Does not compress material properties; requires significant memory during construction if not using sub-DAGs; sorting large keys (256-bit) can be costly [S2].
*   **When to Use:** Ray tracing hard/soft shadows, ambient occlusion, and primary rays in high-resolution scenes with redundant geometry [S2].
*   **When Not to Use:** Scenes with no geometric redundancy; dynamic scenes requiring real-time compression (the method is static) [S2].

## Background
The method was introduced by Viktor Kampe, Erik Sintorn, and Ulf Assarsson from Chalmers University of Technology in the paper "High Resolution Sparse Voxel DAGs" [S1, unofficial][S2]. It builds on earlier work by Webber and Dillencourt [1989] on quadtrees and Parker et al. [2003] on 3D voxel data, but distinguishes itself by decoupling voxel content from geometry, allowing successful compression of irregular data [S1, unofficial]. The bottom-up approach was chosen over top-down recursive testing for its efficiency [S1, unofficial]. It is used primarily for GPU ray tracing of visibility queries in extremely high-resolution voxel grids [S2].

## Sources
[S1] High Resolution Sparse Voxel DAGs, https://www.cse.chalmers.se/~uffe/HighResolutionSparseVoxelDAGs.pdf, fetched 2026-10-27 (unofficial)
[S2] High resolution sparse voxel DAGs, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
