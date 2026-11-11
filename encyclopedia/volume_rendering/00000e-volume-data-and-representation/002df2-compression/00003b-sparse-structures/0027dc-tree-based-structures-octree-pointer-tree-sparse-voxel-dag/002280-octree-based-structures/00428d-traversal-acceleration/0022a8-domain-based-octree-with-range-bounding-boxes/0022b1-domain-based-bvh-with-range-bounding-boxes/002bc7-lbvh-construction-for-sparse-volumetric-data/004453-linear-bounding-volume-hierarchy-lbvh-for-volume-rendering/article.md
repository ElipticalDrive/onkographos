# Linear Bounding Volume Hierarchy (LBVH) for Volume Rendering
Also called: ZHL19 LBVH

## Overview
Linear Bounding Volume Hierarchy (LBVH) is a hierarchical data structure for sparse volume rendering that enables empty space skipping, adapted from surface ray tracing to uniform grids and targeted for GPU implementation via CUDA [S1, unofficial]. It sets itself apart from SAH-based methods (such as k-d trees) by using a middle-split heuristic on Morton-ordered bricks, offering significantly superior construction performance at the cost of inferior culling properties [S1, unofficial].

## How it works
**Inputs:** A uniform volumetric grid, a transfer function, and the target brick size.
**Outputs:** A hierarchy of axis-aligned bounding boxes (AABBs) representing non-empty bricks, ready for traversal.

**Data Structures:**
*   **Brick:** A sub-volume of size $8^3$ voxels.
*   **Morton Code:** A 30-bit integer representing the spatial position of a brick in 3D space, derived from its $(x, y, z)$ coordinates.
*   **Hierarchy Node:** Contains an AABB and pointers/indices to children. Inner nodes are assembled during the final pass.

**Parameters:**
*   Brick size: $8^3$ voxels [S1, unofficial].
*   Morton code size: 30-bit [S1, unofficial].

**Algorithm Steps:**

1.  **Decomposition:** Decompose the input volume into bricks of size $8^3$.
2.  **Visibility Determination:** Launch a CUDA kernel where each thread processes one voxel. The thread determines if the voxel is visible according to the current transfer function [S1, unofficial].
3.  **Brick Voting:** For each brick, threads responsible for its voxels vote on whether the entire brick is visible. This is achieved by atomically updating a flag in on-device shared memory. If any voxel in the brick is visible, the brick is marked non-empty [S1, unofficial].
4.  **Compaction:** Perform a parallel compaction operation to retain only the non-empty bricks. This step has $O(n)$ work complexity, where $n$ is the total number of bricks [S1, unofficial].
5.  **Morton Code Assignment:** Assign a 30-bit 3D Morton code to each non-empty brick based on its spatial coordinates [S1, unofficial].
6.  **Sorting:** Sort the non-empty bricks using a parallel $O(n)$ GPU sorting algorithm (e.g., from the Thrust library) based on their Morton codes [S1, unofficial].
7.  **Hierarchy Definition:** The sorted Morton order implicitly defines the hierarchy. The structure spatially splits child nodes in the middle [S1, unofficial].
8.  **Split Position Calculation:** Determine the split positions for the hierarchy nodes using Karras’ algorithm [Kar12], which reads split positions directly from the bit codes of the Morton indices [S1, unofficial].
9.  **AABB Assembly:** Traverse the hierarchy from each leaf node up to the root node in a final CUDA kernel. As the traversal proceeds, assemble the axis-aligned bounding box (AABB) for each inner node encountered along the path [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1-3: Visibility and Voting
def process_bricks(volume, transfer_function, brick_size=8):
    # volume: 3D array of voxel densities
    # transfer_function: function mapping density to opacity
    
    # 1. Decompose into bricks (implicit in indexing)
    # 2. Determine voxel visibility
    # Each thread handles one voxel
    voxel_visible = [transfer_function(density) > 0 for density in volume.flatten()]
    
    # 3. Vote for brick visibility using atomic operations
    # Shared memory flag per brick
    brick_flags = [0] * (len(volume) // (brick_size**3))
    
    for brick_idx in range(len(brick_flags)):
        # Check if any voxel in this brick is visible
        # In CUDA, this is done via atomicOr in shared memory
        is_visible = any(voxel_visible[brick_idx * brick_size**3 : (brick_idx + 1) * brick_size**3])
        if is_visible:
            brick_flags[brick_idx] = 1
            
    # Step 4: Compaction
    # Keep only indices where brick_flags is 1
    non_empty_bricks = [i for i, flag in enumerate(brick_flags) if flag == 1]
    
    # Step 5: Morton Code Assignment
    # Convert brick index to 3D coordinates, then to 30-bit Morton code
    def index_to_morton(idx, total_bricks_x, total_bricks_y, total_bricks_z):
        # Decode linear brick index to (bx, by, bz)
        bx = (idx // (total_bricks_y * total_bricks_z)) % total_bricks_x
        by = (idx // total_bricks_z) % total_bricks_y
        bz = idx % total_bricks_z
        # Interleave bits for 3D Morton code (30-bit limit)
        return interleave_bits(bx, by, bz)
        
    morton_codes = [index_to_morton(idx, ...) for idx in non_empty_bricks]
    
    # Step 6: Sorting
    # Sort morton_codes and keep track of original brick indices
    sorted_pairs = sorted(enumerate(morton_codes), key=lambda x: x[1])
    sorted_bricks = [pair[0] for pair in sorted_pairs] # Original indices
    sorted_mortons = [pair[1] for pair in sorted_pairs] # Sorted codes
    
    # Step 7-8: Split Positions (Karras' Algorithm)
    # Split positions are derived from the bit differences in sorted Morton codes
    # This defines the tree structure implicitly
    splits = calculate_splits_karras(sorted_mortons)
    
    # Step 9: AABB Assembly
    # Traverse from leaves to root to compute AABBs
    # Note: The exact tree structure is defined by the splits and sorted order
    hierarchy = build_hierarchy_from_morton_order(sorted_bricks, splits, volume, brick_size)
    
    return hierarchy

def interleave_bits(x, y, z):
    # Standard 3D Morton code bit interleaving
    # Returns a 30-bit integer
    # Implementation details omitted for brevity, assumed standard
    pass

def calculate_splits_karras(morton_codes):
    # Karras' algorithm finds split points by analyzing bit patterns
    # Returns a list of split indices or positions
    pass

def build_hierarchy_from_morton_order(bricks, splits, volume, brick_size):
    # Recursive or iterative construction of nodes
    # Each node stores an AABB covering its bricks
    # AABB is computed from the min/max coordinates of the bricks in the range
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** Morton codes are limited to 30 bits. Ensure the volume resolution and brick count fit within this constraint to avoid overflow or collision [S1, unofficial].
*   **Empty Volumes:** If no voxels are visible, the compaction step results in an empty list. The hierarchy construction must handle this gracefully [S1, unofficial].
*   **Transfer Function Changes:** The visibility determination is tied to the current transfer function. If the transfer function changes significantly, the hierarchy may need to be rebuilt or updated [S1, unofficial].

## Operating characteristics
*   **Construction Time:** Significantly improved construction rates compared to SAH-based methods (like k-d trees) [S1, unofficial]. The algorithm relies on $O(n)$ parallel operations (compaction, sorting, assembly) [S1, unofficial].
*   **Culling Quality:** Inferior culling properties compared to SAH-based methods due to the use of the middle-split heuristic [S1, unofficial].
*   **Hardware:** Designed for GPU implementation via NVIDIA CUDA [S1, unofficial].
*   **Complexity:** The core operations (compaction, sorting, assembly) have $O(n)$ work complexity, where $n$ is the number of bricks [S1, unofficial].
*   **When to Use:** When fast hierarchy construction is prioritized over optimal traversal performance, such as in dynamic scenes or when the transfer function changes frequently [S1, unofficial].
*   **When Not to Use:** When maximum culling efficiency is required and construction time is less critical [S1, unofficial].

## Background
The LBVH algorithm was initially introduced by Lauterbach et al. [LGS*09]. The specific implementation for volume rendering, referred to as ZHL19 LBVH, adapts this algorithm to uniform grids and targets GPU implementation via CUDA [S1, unofficial]. It is part of a family of hierarchical data structures for sparse volume rendering, contrasting with more intricate construction schemes like those in [ZSL18] and [ZSL19] which use SAH-like cost functions [S1, unofficial].

## Sources
[S1] Comparing Hierarchical Data Structures for Sparse Volume Rendering with Empty Space Skipping, https://doi.org/10.36227/techrxiv.11416422.v1, fetched 2026-10-24 (unofficial)
