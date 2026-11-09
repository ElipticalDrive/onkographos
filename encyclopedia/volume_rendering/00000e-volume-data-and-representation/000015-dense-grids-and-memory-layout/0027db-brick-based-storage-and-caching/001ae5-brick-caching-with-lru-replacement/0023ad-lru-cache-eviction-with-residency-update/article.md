# LRU Cache Eviction with Residency Update

## Overview
This method couples LRU brick cache eviction with hierarchical residency octree updates [S1]. When the cache lacks a free slot, it evicts the least recently accessed brick and resets bitmask entries in affected leaf and non-leaf nodes to reflect the loss of resident data [S1]. It differs from standard LRU caches by synchronizing cache state with octree metadata after each eviction [S1].

## How it works
**Inputs:**
- A brick cache containing resident bricks [S1].
- An LRU tracking structure mapping bricks to their access recency [S1].
- A residency octree where each node maintains a bitmask indicating which resolution levels have at least one resident brick in its spatial extent [S1].
- The spatial extent and resolution level of the evicted brick [S1].

**Outputs:**
- The cache state with one fewer brick (the LRU brick removed) [S1].
- The residency octree with updated bitmasks reflecting the removal [S1].

**Data Structures:**
- **Cache Slot:** Stores brick ID, spatial AABB, resolution level, and access timestamp/pointer [G].
- **LRU List:** Doubly linked list of cache slots ordered by access time (head = least recent, tail = most recent) [G].
- **Octree Node:**
  - `bitmask`: Integer or bitset where bit $i$ is 1 if any brick at resolution level $i$ within this node's spatial extent is resident [S1].
  - `children`: Array of child nodes (for non-leaf nodes) [G].
  - `spatial_extent`: AABB defining the node's 3D volume [G].
  - `parent`: Pointer to the parent node (for upward propagation) [G].
  - `resident_bricks`: List of bricks currently cached under this node's spatial extent [G].

**Algorithm Steps:**
1. **Eviction Trigger:** If no cache slot is available, evict the least recently accessed brick using an LRU scheme [S1].
2. **Remove from Cache:** Remove the victim brick from the cache storage and LRU list [S1].
3. **Identify Affected Leaf Nodes:** Check all leaf nodes whose spatial extent overlaps the evicted brick's spatial extent [S1].
4. **Update Leaf Node Bitmasks:** For each such leaf node, check if it is still partially mapped after removal by verifying if any other brick in the same resolution level is resident in the cache [S1]. If no other brick in that resolution level is resident, reset the corresponding bit in the node's bitmask to indicate it is no longer partially mapped [S1].
5. **Propagate Updates to Non-Leaf Nodes:** Update non-leaf nodes by setting their bitmasks to the bitwise OR of their child nodes' bitmasks [S1]. As soon as a node’s bitmask remains unchanged by this operation, terminate the upwards propagation of the update [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evict_lru_and_update_residency(cache, lru_list, octree_root, evicted_brick):
    # Step 1 & 2: Evict LRU brick
    if not lru_list.is_empty():
        victim = lru_list.pop_head()
        cache.remove(victim.brick_id)
    else:
        return

    # Step 3: Identify affected leaf nodes
    affected_leaves = []
    def collect_leaves(node):
        if node.is_leaf():
            if node.spatial_extent.overlaps(evicted_brick.spatial_extent):
                affected_leaves.append(node)
        else:
            for child in node.children:
                collect_leaves(child)
    collect_leaves(octree_root)

    # Step 4: Update leaf node bitmasks
    for leaf in affected_leaves:
        level = evicted_brick.resolution_level
        # Check if any other brick at this resolution level is resident in this leaf
        is_resident = any(brick.level == level for brick in leaf.resident_bricks)
        if not is_resident:
            leaf.bitmask &= ~(1 << level)
            propagate_update(leaf)

def propagate_update(node):
    parent = node.parent
    if parent is None:
        return

    # Recalculate parent bitmask
    new_parent_bitmask = 0
    for child in parent.children:
        new_parent_bitmask |= child.bitmask

    if new_parent_bitmask != parent.bitmask:
        parent.bitmask = new_parent_bitmask
        propagate_update(parent)
```

**Parameters:**
- Cache capacity: Not given in the sources. [G]
- Bitmask size: Determined by the maximum number of resolution levels in the octree [S1].

**Edge Cases:**
- **Precision:** Bitwise operations assume integer bitmasks. Ensure no overflow if resolution levels exceed standard integer width [G].
- **Boundaries:** Spatial overlap checks must be inclusive of brick boundaries to avoid missing adjacent bricks [G].
- **Degenerate Input:** If the evicted brick was the only resident brick for a resolution level in a leaf, the bit must be reset [S1]. If the leaf had no bricks at that level initially, the bit is already 0 [G].

## Subtopics

### GPU Residency Budget Fix
A variant addressing memory pile-up in WebGL2 environments by setting a 1.25 GB residency budget, deliberately below the plan budget, to ensure LRU evicts old bricks during refocus [S2, unofficial]. Previous behavior allowed an 8 GB GPU residency request against a 2 GB plan budget, causing the LRU to never evict and pile bricks on top of old ones (e.g., 1.67 GB across 177 bricks on `hoa_heart` L0) [S2, unofficial]. This fix forces eviction by capping the total resident data size [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Strengths:** Maintains strict consistency between cache state and rendering metadata by resetting bitmask entries only when necessary [S1].
- **Limitations:** Requires checking leaf nodes for spatial overlap and propagating bitmask updates up the tree after each eviction [S1].
- **When to use:** Mixed-resolution multi-volume rendering requiring accurate residency tracking for ray-guided traversal [S1].
- **When not to use:** Not given in the sources. [G]

## Background
Developed as part of the Residency Octree approach for scalable web volume rendering, this method integrates LRU cache management with hierarchical residency tracking [S1]. It supports mixed-resolution multi-volume rendering where ray-guided traversal relies on accurate bitmask information to skip empty regions or access resident data [S1]. The GPU Residency Budget Fix variant addresses specific implementation issues in WebGL2 environments, as documented in unofficial repository updates [S2, unofficial].

## Sources
[S1] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S2] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
