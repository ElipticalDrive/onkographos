# Morton Key Vertex Neighbor Iteration

Also called: Vertex Neighbor Iteration via Morton Keys

## Overview
Morton Key Vertex Neighbor Iteration is a method for efficiently identifying and processing the eight corner vertices of leaf nodes in a sparse voxel octree by leveraging the spatial locality of Morton (Z-order) keys. It solves the problem of dual-grid traversal, where each vertex must be processed exactly once despite being shared by up to eight adjacent leaf nodes. It differs from standard recursive neighbor finding by operating on a pre-computed or iterated list of vertex keys, using bitwise arithmetic to resolve adjacency and handle level mismatches without tree traversal recursion.

## How it works
### Inputs and Outputs
- **Inputs**:
  - A sparse voxel octree implementation providing `node_exists(key)`, `is_subdivided(key)`, and `parent()` operations.
  - A list of `vertex_keys`: an array of `MortonKey` representing the 8 vertices of a leaf node, where invalid vertices are marked with `MortonKey::none()`.
  - The level of the leaf node (`vertex_lv`).
- **Outputs**:
  - A list of resolved neighbor keys for the valid vertices of the leaf. Each entry is either a valid `MortonKey` of an existing node at the same level (or an ancestor if the direct neighbor is missing) or `MortonKey::none()`.

### Data Structures
- **MortonKey**: A struct wrapping a `u64`. The internal representation starts with `1` for the root node, appending 3 bits per level for the child index (X, Y, Z).
  - `pub struct MortonKey(pub u64);`
  - `MortonKey::none()` returns `Self(0)`.
  - `MortonKey::root()` returns `Self(1)`.
- **HashedOctree**: A hash map-based storage for the octree.
  - `pub struct HashedOctree<T: Copy> { values: DashMap<MortonKey, T, RandomState>, }`
- **OctreeIndex**: Bitflags for child indices.
  - `RIGHT = 1 << 0`
  - `TOP = 1 << 1`
  - `FOREMOST = 1 << 2`

### Constants
The method relies on three 64-bit dilation constants to perform bitwise arithmetic on Morton keys. These constants have bits set at positions corresponding to the X, Y, and Z axes in the Morton interleaving.

- `DIL_X = 0b001001001001001001001001001001001001001001001001001001001001001u64`
- `DIL_Y = 0b010010010010010010010010010010010010010010010010010010010010010u64`
- `DIL_Z = 0b100100100100100100100100100100100100100100100100100100100100100u64`

### Formulas
The core of the method is "dilated integer addition" and "dilated integer subtraction." These operations allow moving along grid axes in Morton space using standard integer arithmetic, masked by the dilation constants.

1.  **Dilated Integer Addition** (used to compute vertex keys from a leaf key):
    $$
    vertex\_key = \left( (key.0 \mid \neg DIL\_X) + (vertex\_i \ \& \ DIL\_X) \right) \ \& \ DIL\_X \ \mid \ \left( (key.0 \mid \neg DIL\_Y) + (vertex\_i \ \& \ DIL\_Y) \right) \ \& \ DIL\_Y \ \mid \ \left( (key.0 \mid \neg DIL\_Z) + (vertex\_i \ \& \ DIL\_Z) \right) \ \& \ DIL\_Z
    $$
    Where:
    - $key.0$ is the Morton key of the leaf node.
    - $vertex\_i$ is the vertex index (0–7).
    - $\mid$ is bitwise OR, $\neg$ is bitwise NOT, $\&$ is bitwise AND, $+$ is integer addition.

2.  **Dilated Integer Subtraction** (used to compute neighbor leaf keys from a vertex key):
    $$
    key_{node\_i} = \left( (dc \ \& \ DIL\_X) - (node\_i \ \& \ DIL\_X) \right) \ \& \ DIL\_X \ \mid \ \left( (dc \ \& \ DIL\_Y) - (node\_i \ \& \ DIL\_Y) \right) \ \& \ DIL\_Y \ \mid \ \left( (dc \ \& \ DIL\_Z) - (node\_i \ \& \ DIL\_Z) \right) \ \& \ DIL\_Z
    $$
    Where:
    - $dc$ is the dilated coordinate of the vertex, computed as $vertex\_k.0 \ll (vertex\_lv - vertex\_k.level()) \times 3$.
    - $node\_i$ is the neighbor index (0–7).

### Algorithm Steps
The method processes a list of vertex keys to find unique neighbors.

1.  **Iterate Vertices**: Enumerate over the `vertex_keys` array. Stop when `vertex_k` equals `MortonKey::none()`. Let the current index be `vertex_i` and the key be `vertex_k`.
2.  **Compute Neighbor Candidates**: Call `vert2leaf(vertex_k, vertex_lv)` to get an array `neighbours` of 8 candidate `MortonKey` values for the adjacent nodes.
3.  **Filter Neighbors**: For each neighbor index `neighbour_i` from 0 to 7:
    a.  If `neighbour_i == vertex_i`, skip (this is the node itself).
    b.  Let `neighbour_k = neighbours[neighbour_i]`.
    c.  If `!octree.node_exists(neighbour_k)`, continue to the next neighbor.
    d.  If `octree.is_subdivided(neighbour_k)` (the neighbor is deeper than the current leaf), return `false` from the filter for the *entire vertex*. This ensures the deeper node processes the shared vertex.
    e.  If `neighbour_i < vertex_i`, return `false` from the filter for the *entire vertex*. This ensures that if two nodes share a vertex, only the one with the higher index (or specific tie-breaking rule) processes it.
4.  **Resolve Keys**: If the vertex passes the filter:
    a.  Initialize an array `keys: [MortonKey; 8]`.
    b.  For each `neighbour_i` from 0 to 7:
        i.  Let `neighbour_k = neighbours[neighbour_i]`.
        ii. While `neighbour_k != MortonKey::none()` AND `!octree.node_exists(neighbour_k)`, set `neighbour_k = neighbour_k.parent()`.
        iii. Store `neighbour_k` in `keys[neighbour_i]`.
    c.  Return `keys`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
DIL_X = 0b001001001001001001001001001001001001001001001001001001001001001
DIL_Y = 0b010010010010010010010010010010010010010010010010010010010010010
DIL_Z = 0b100100100100100100100100100100100100100100100100100100100100100

def leaf2vert(key: MortonKey) -> tuple[u32, list[MortonKey]]:
    """
    Returns the level and 8 vertex keys for a leaf node.
    """
    level = key.level()
    level_key = 1 << (3 * level)
    keys = [None] * 8
    
    for vertex_i in range(8):
        # Dilated integer addition
        vertex_key = (
            ((key.0 | ~DIL_X) + (vertex_i & DIL_X)) & DIL_X |
            ((key.0 | ~DIL_Y) + (vertex_i & DIL_Y)) & DIL_Y |
            ((key.0 | ~DIL_Z) + (vertex_i & DIL_Z)) & DIL_Z
        )
        
        # Check validity: must be within level bounds and not on axis boundaries
        if (vertex_key >= (level_key << 1)) or \
           ((vertex_key - level_key) & DIL_X) == 0 or \
           ((vertex_key - level_key) & DIL_Y) == 0 or \
           ((vertex_key - level_key) & DIL_Z) == 0:
            keys[vertex_i] = MortonKey.none()
        else:
            keys[vertex_i] = MortonKey(vertex_key)
            
    return level, keys

def vert2leaf(vertex_k: MortonKey, vertex_lv: u32) -> list[MortonKey]:
    """
    Returns the 8 neighbor leaf keys for a vertex.
    """
    # Dilate the vertex key to the target level
    dc = vertex_k.0 << ((vertex_lv - vertex_k.level()) * 3)
    keys = [None] * 8
    
    for node_i in range(8):
        # Dilated integer subtraction
        keys[node_i] = MortonKey(
            ((dc & DIL_X) - (node_i & DIL_X)) & DIL_X |
            ((dc & DIL_Y) - (node_i & DIL_Y)) & DIL_Y |
            ((dc & DIL_Z) - (node_i & DIL_Z)) & DIL_Z
        )
    return keys

def process_vertex_neighbors(vertex_keys: list[MortonKey], octree: HashedOctree):
    """
    Iterates over vertex keys and resolves unique neighbors.
    """
    results = []
    
    # Iterate while vertex_k is not none
    for vertex_i, vertex_k in enumerate(vertex_keys):
        if vertex_k == MortonKey.none():
            break
            
        # Get candidate neighbors
        neighbours = vert2leaf(vertex_k, vertex_k.level()) # Note: source uses vertex_lv, assumed equal to key.level() here for leaf context
        
        # Filter logic
        should_process = True
        for neighbour_i in range(8):
            if neighbour_i == vertex_i:
                continue
                
            neighbour_k = neighbours[neighbour_i]
            
            # Skip if neighbor doesn't exist
            if not octree.node_exists(neighbour_k):
                continue
                
            # Skip if neighbor is subdivided (deeper)
            if octree.is_subdivided(neighbour_k):
                should_process = False
                break
                
            # Tie-breaking: process only if neighbour_i >= vertex_i
            # Source says: if neighbour_i < vertex_i, return false.
            if neighbour_i < vertex_i:
                should_process = False
                break
        
        if not should_process:
            continue
            
        # Resolve keys
        keys = [None] * 8
        for neighbour_i in range(8):
            neighbour_k = neighbours[neighbour_i]
            while neighbour_k != MortonKey.none() and not octree.node_exists(neighbour_k):
                neighbour_k = neighbour_k.parent()
            keys[neighbour_i] = neighbour_k
            
        results.append(keys)
        
    return results
```

### Edge Cases and Pitfalls
- **Invalid Vertex Keys**: A vertex key is invalid if it falls outside the level's range (`vertex_key >= level_key << 1`) or if any dilated axis difference is zero. These return `MortonKey::none()` [S1, unofficial].
- **Concurrency**: When using `HashedOctree` (backed by `DashMap`), concurrent access may deadlock if a mutable reference is held while calling `value()` [S1, unofficial].
- **Tie-Breaking**: The condition `neighbour_i < vertex_i` ensures that shared vertices between nodes of the same level are processed only once. The specific index comparison depends on the iteration order of the vertex keys [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(V)$ where $V$ is the number of valid vertices. For each vertex, neighbor resolution involves a constant number of bitwise operations and a while-loop for parent traversal. The parent traversal depth is bounded by the octree depth.
- **Memory Complexity**: $O(1)$ auxiliary space per vertex (storing 8 keys). The octree storage is external.
- **Strengths**: Avoids recursive tree traversal for neighbor finding. Uses fast bitwise arithmetic. Efficient for sparse grids where many neighbors are missing.
- **Limitations**: Requires pre-computed or easily derivable vertex keys. Relies on the octree implementation providing efficient `node_exists` and `is_subdivided` checks.

## Background
The method is described in the `dmc` (Dense Marching Cuber) project by aleokdev. It is used for dual-grid traversal in volume rendering, specifically for extracting isosurfaces from sparse voxel octrees. The approach leverages the properties of Morton keys to map spatial adjacency to arithmetic operations.

## Sources
[S1] aleokdev/dmc, https://github.com/aleokdev/dmc, fetched 2026-10-27 (unofficial)
