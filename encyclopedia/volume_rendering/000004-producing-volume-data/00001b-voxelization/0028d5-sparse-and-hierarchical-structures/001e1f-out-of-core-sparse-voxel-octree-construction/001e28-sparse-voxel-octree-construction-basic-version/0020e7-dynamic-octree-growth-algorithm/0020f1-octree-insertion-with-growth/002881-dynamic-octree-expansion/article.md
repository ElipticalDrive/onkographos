# Dynamic Octree Expansion

Also called: Enlarge, EnlargeCorners

## Overview
Dynamic Octree Expansion is the mechanism used to grow a Sparse Voxel Octree when a voxel insertion request falls outside the current bounding volume. It replaces the root node with a new parent that spans a larger volume, allowing the tree to accommodate data without pre-allocating a fixed grid. It differs from static expansion by dynamically allocating a new root and re-parenting the existing tree, whereas sibling methods like [Voxel Placement in Octree](3097-voxel-placement-in-octree.md) assume the tree already has sufficient capacity.

## How it works
**Inputs:**
- `m_size`: Current side length of the octree's bounding box.
- `m_position`: Current world-space position of the octree's origin (typically the corner corresponding to the minimum coordinates).
- `direction`: The axis and sign of expansion (e.g., `xplus`, `yminus`, `zplus`).

**Outputs:**
- A new Octree root node with double the size and updated position.
- A new `OctreeChildren` structure containing the previous root and the new empty child.

**Data Structures:**
- `OctreeChildren`: Contains a bitmask indicating which of the 8 children are valid, and an array `rootChildren[8]` of pointers to child nodes.
- `Octree`: Contains `m_size`, `m_position`, and a pointer to the root `OctreeChildren`.

**Steps:**
1. **Compute Indices and Offset:** Call `EnlargeCorners(direction)` to determine:
   - `oldCornerIdx`: The index (0-7) in the child array where the existing root node will be placed.
   - `newCornerIdx`: The index where the new empty node will be placed.
   - `offset`: The vector displacement required to position the new node relative to the old one.
2. **Initialize Children Array:** Create an array `rootChildren` of 8 pointers, initialized to `nullptr`.
3. **Place Existing Root:** Assign the current root node to `rootChildren[oldCornerIdx]`.
4. **Create New Empty Node:** Instantiate a new empty octree node at `rootChildren[newCornerIdx]`. Its size is `m_size`, its position is `m_position + offset`, and its depth is 1.
5. **Construct Children Struct:** Create a new `OctreeChildren` object. Set its bitmask to include only `oldCornerIdx` and `newCornerIdx`. Store the `rootChildren` array.
6. **Update Position (Negative Directions):** If the expansion direction is negative (`xminus`, `yminus`, or `zminus`), update `m_position` by adding `offset`. This ensures the bounding box origin remains consistent with the new larger volume.
7. **Double Size:** Update `m_size` to `m_size * 2`.
8. **Replace Root:** Create a new `Octree` instance with the new `OctreeChildren`, the updated `m_size`, and the updated `m_position`. This becomes the new root of the tree.

**Pitfalls:**
- The offset calculation for `xplus` in some implementations uses `glm::vec3(m_size, 0, 0)` but may be marked as unused or improperly utilized, leading to incorrect positioning if not verified [S1, unofficial].
- Ensure `offset` is correctly aligned with the expansion direction; incorrect offsets cause gaps or overlaps in the spatial coverage.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_octree_expansion(octree, direction):
    # Step 1: Compute indices and offset
    old_corner_idx, new_corner_idx, offset = enlarge_corners(direction)
    
    # Step 2: Initialize children array
    root_children = [None] * 8
    
    # Step 3: Place existing root
    root_children[old_corner_idx] = octree.root_children
    
    # Step 4: Create new empty node
    new_node_size = octree.m_size
    new_node_position = octree.m_position + offset
    new_empty_node = OctreeNode(size=new_node_size, position=new_node_position, depth=1)
    root_children[new_corner_idx] = new_empty_node
    
    # Step 5: Construct children struct
    # Bitmask has bits old_corner_idx and new_corner_idx set
    bitmask = (1 << old_corner_idx) | (1 << new_corner_idx)
    new_children = OctreeChildren(bitmask=bitmask, children=root_children)
    
    # Step 6: Update position for negative directions
    if direction in [DIRECTION_XMINUS, DIRECTION_YMINUS, DIRECTION_ZMINUS]:
        octree.m_position = octree.m_position + offset
    
    # Step 7: Double size
    octree.m_size = octree.m_size * 2
    
    # Step 8: Replace root
    octree.root_children = new_children
```

## Subtopics

### Node Expansion
Expands a pruned node into its 8 children, setting their occupancy probability to the parent's value. This is the reverse of pruning. The caller must verify the node is pruned (not a leaf at the lowest level) [S2]. This operation is expensive when the tree is nearly empty [S2].

## Operating characteristics
- **Time Complexity:** O(1) for the expansion itself, as it involves a constant number of pointer assignments and one new node allocation. However, subsequent traversals may be affected by the increased depth.
- **Memory Complexity:** O(1) additional memory for the new root children structure and one new empty node.
- **Performance:** Expansion is relatively cheap compared to full tree reconstruction, but frequent expansions can lead to fragmentation if not managed. The operation is necessary for dynamic scenes but should be minimized in static scenarios.
- **Strengths:** Allows for unbounded growth without pre-allocation. Simple to implement.
- **Limitations:** Frequent expansions can lead to a tree with many levels, increasing traversal time. The "expensive operation" note for node expansion suggests that reversing pruning is costly, implying expansion should be done carefully [S2].

## Background
Dynamic Octree Expansion is a fundamental technique in dynamic sparse voxel octree construction, enabling the tree to adapt to varying scene densities. It is commonly used in real-time rendering and collision detection systems where the scene geometry changes over time. The method is implemented in various open-source projects, such as the procgen repository [S1, unofficial], and is conceptually similar to node expansion in octomap libraries [S2].

## Sources
[S1] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
[S2] octomap: octomap::OcTree Class Reference, https://octomap.github.io/octomap/doc/classoctomap_1_1OcTree.html, fetched 2026-10-24
