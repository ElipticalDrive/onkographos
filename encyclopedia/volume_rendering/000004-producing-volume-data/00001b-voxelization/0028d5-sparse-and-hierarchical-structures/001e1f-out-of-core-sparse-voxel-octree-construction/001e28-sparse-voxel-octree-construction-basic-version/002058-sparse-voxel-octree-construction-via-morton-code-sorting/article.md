# Sparse Voxel Octree Construction via Morton Code Sorting

Also called: BVH Construction via Morton Code Sorting, Bottom-up SVO Construction with Morton Sort, Morton Code Sorting SVO Construction, Morton-based octree insertion, Octree Construction via Morton Code Sorting, Z-order octree builder

## Overview
This method constructs a static Sparse Voxel Octree (SVO) from a set of 3D points or voxels by converting them to Z-order (Morton) codes, sorting them for spatial locality, and recursively partitioning the sorted array into octree nodes and leaf voxels [S1, unofficial]. It differs from dynamic insertion methods by pre-sorting all points into Morton order, which guarantees spatial locality and allows $O(1)$ range queries via binary search instead of tree traversal during construction [S1, unofficial].

## How it works
**Inputs and Outputs:** A list of 3D points (or voxel centers) $P = \{p_0, p_1, \dots, p_{N-1}\}$ [S1, unofficial]. An SVO consisting of an array of `octree_node` structures and an array of `octree_voxel` structures, serialized in a specific binary format [S1, unofficial].

**Data Structures:** The output is stored in two contiguous arrays:
1. **`octree_node`** (8 bytes total):
   - `uint32_t info`: Bits 0-7 store the occupancy mask (8 bits for 8 children). Bit 8 is the `is_leaf` flag. Bits 9-31 are padding [S1, unofficial].
   - `uint32_t child_ptr`: If `is_leaf` is 0, this is an index into the `octree_node` array (children). If `is_leaf` is 1, this is an index into the `octree_voxel` array (data) [S1, unofficial].
2. **`octree_voxel`** (4 bytes total):
   - `uint32_t rgb`: Bits 0-7 padding. Bits 8-15 Blue, Bits 16-23 Green, Bits 24-31 Red [S1, unofficial].
3. **Binary Serialization Format**:
   - `size_t num_nodes`
   - `size_t num_voxels`
   - `num_nodes * sizeof(octree_node)` bytes [S1, unofficial]
   - `num_voxels * sizeof(octree_voxel)` bytes [S1, unofficial]

**Parameters:**
- `max_depth`: Maximum recursion depth for the octree. Controls grid resolution. Valid range [1, 12]. Default 8. $2^{\text{max\_depth}}$ defines the side length in voxels [S1, unofficial].

**Algorithm Steps:**
1. **Compute Bounding Box**: Initialize `min_bound` and `max_bound` to the first point $p_0$ [S1, unofficial]. Iterate through all points $p \in P$: $\text{min\_bound} = \min(\text{min\_bound}, p)$, $\text{max\_bound} = \max(\text{max\_bound}, p)$ [S1, unofficial].
2. **Equalize Aspect Ratio**: Compute extent: $\text{extent} = \text{max\_bound} - \text{min\_bound}$ [S1, unofficial]. Find $\text{max\_extent} = \max(\text{extent})$ [S1, unofficial]. Calculate margin: $\text{margin} = (\text{vec3}(\text{max\_extent}) - \text{extent}) \times 0.5$ [S1, unofficial]. Shift bounds: $\text{min\_bound} \leftarrow \text{min\_bound} - \text{margin}$ [S1, unofficial].
3. **Compute Grid Scale**: $\text{scale} = \frac{1 \ll \text{max\_depth}}{\max(\text{max\_extent}, 10^{-6})}$ [S1, unofficial].
4. **Convert to Grid Coordinates**: For each point $p$: $\text{grid\_pos} = \lfloor (p - \text{min\_bound}) \times \text{scale} \rfloor$ [S1, unofficial].
5. **Clamp Grid Coordinates**: $\text{grid\_pos} = \min(\text{grid\_pos}, \text{vec3}((1 \ll \text{max\_depth}) - 1))$ [S1, unofficial].
6. **Encode Morton Codes**: For each `grid_pos` $(x, y, z)$, compute a 64-bit Morton code $M$: $M = 0$. For $i$ from 0 to 11: $M \ |= \ ((x \ \& \ (1 \ll i)) \ll 2i) \ | \ ((y \ \& \ (1 \ll i)) \ll (2i + 1)) \ | \ ((z \ \& \ (1 \ll i)) \ll (2i + 2))$ [S1, unofficial].
7. **Sort and Deduplicate**: Sort the list of Morton codes. Remove duplicates using `std::unique` [S1, unofficial].
8. **Recursive Construction**: Call `insert_children_recursive(min_bound, 0, N, 1)`, where $N$ is the number of unique Morton codes [S1, unofficial].
   **Function `insert_children_recursive(min_coord, start_index, end_index, depth)`:**
   - Compute `half_size` for the current depth: $\text{half\_size} = 1 \ll (\text{max\_depth} - \text{depth})$ [S1, unofficial].
   - Initialize `occupancy_mask = 0`.
   - For each octant $i$ from 0 to 7:
     - Compute child bounds: $\text{child\_min} = \text{min\_coord} + \text{octant\_offset}(i) \times \text{half\_size}$, $\text{child\_max} = \text{child\_min} + \text{half\_size} - 1$ [S1, unofficial].
     - Find occupied range in the sorted Morton array using binary search: $\text{range\_start} = \text{lower\_bound}(M(\text{child\_min}), \text{begin} + \text{start\_index}, \text{begin} + \text{end\_index})$, $\text{range\_end} = \text{upper\_bound}(M(\text{child\_max}), \text{range\_start}, \text{begin} + \text{end\_index})$ [S1, unofficial].
     - If $\text{range\_start} < \text{range\_end}$: Set bit $i$ in `occupancy_mask`. If `depth == max_depth`, add voxel to `children_voxels[depth]`. Else, recurse: `child = insert_children_recursive(child_min, range_start, range_end, depth + 1)` and store result in `children_nodes[depth]` [S1, unofficial].
   - Create `octree_node`: `info` = `occupancy_mask` | (`is_leaf ? 1 : 0`) << 8. `child_ptr` = index to first child or voxel.

**Edge Cases and Pitfalls:**
- Division by zero is prevented by clamping `max_extent` to $10^{-6}$ if it is 0 [S1, unofficial].
- Clamping prevents Morton code overflow. `max_depth` is limited to 12 to ensure coordinates stay $< 4096$ [S1, unofficial].
- `child_max` is decremented by 1 because `z_order(child_max)` must fall within the same octant as `z_order(child_min)` [S1, unofficial].
- Duplicates represent multiple voxels at the same grid position; they are merged into a single leaf [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_svo(points, max_depth=8):
    if not points:
        return [], []

    # 1. Compute Bounding Box
    min_bound = points[0]
    max_bound = points[0]
    for p in points:
        min_bound = min(min_bound, p)
        max_bound = max(max_bound, p)

    # 2. Equalize Aspect Ratio
    extent = max_bound - min_bound
    max_extent = max(extent.x, extent.y, extent.z)
    margin = vec3(max_extent) - extent
    margin = margin * 0.5
    min_bound = min_bound - margin

    # 3. Compute Grid Scale
    scale = (1 << max_depth) / max(max_extent, 1e-6)

    # 4. Convert to Grid Coordinates and Clamp
    grid_positions = []
    limit = (1 << max_depth) - 1
    for p in points:
        gp = floor((p - min_bound) * scale)
        gp.x = min(gp.x, limit)
        gp.y = min(gp.y, limit)
        gp.z = min(gp.z, limit)
        grid_positions.append(gp)

    # 5. Encode Morton Codes
    morton_codes = []
    for gp in grid_positions:
        m = 0
        for i in range(12):
            m |= ((gp.x & (1 << i)) << (2 * i))
            m |= ((gp.y & (1 << i)) << (2 * i + 1))
            m |= ((gp.z & (1 << i)) << (2 * i + 2))
        morton_codes.append(m)

    # 6. Sort and Deduplicate
    morton_codes = sorted(list(set(morton_codes)))
    n = len(morton_codes)

    # Global storage
    nodes = []
    voxels = []

    def encode_morton(x, y, z):
        m = 0
        for i in range(12):
            m |= ((x & (1 << i)) << (2 * i))
            m |= ((y & (1 << i)) << (2 * i + 1))
            m |= ((z & (1 << i)) << (2 * i + 2))
        return m

    def lower_bound(arr, val, lo, hi):
        while lo < hi:
            mid = (lo + hi) // 2
            if arr[mid] < val:
                lo = mid + 1
            else:
                hi = mid
        return lo

    def upper_bound(arr, val, lo, hi):
        while lo < hi:
            mid = (lo + hi) // 2
            if arr[mid] <= val:
                lo = mid + 1
            else:
                hi = mid
        return lo

    def octant_offset(i):
        return vec3(i & 1, (i >> 1) & 1, (i >> 2) & 1)

    def insert_children_recursive(min_coord, start_idx, end_idx, depth):
        half_size = 1 << (max_depth - depth)
        occupancy_mask = 0
        child_indices = []

        for i in range(8):
            child_min = min_coord + octant_offset(i) * half_size
            child_max = child_min + half_size - 1
            
            m_min = encode_morton(child_min.x, child_min.y, child_min.z)
            m_max = encode_morton(child_max.x, child_max.y, child_max.z)
            
            range_start = lower_bound(morton_codes, m_min, start_idx, end_idx)
            range_end = upper_bound(morton_codes, m_max, range_start, end_idx)
            
            if range_start < range_end:
                occupancy_mask |= (1 << i)
                if depth == max_depth:
                    voxel_idx = len(voxels)
                    voxels.append([0, 0, 0, 0]) # Placeholder RGB
                    child_indices.append(voxel_idx)
                else:
                    child_idx = insert_children_recursive(child_min, range_start, range_end, depth + 1)
                    child_indices.append(child_idx)
            else:
                child_indices.append(None)

        node_info = occupancy_mask | ((1 if depth == max_depth else 0) << 8)
        node_idx = len(nodes)
        nodes.append([node_info, 0])
        
        if child_indices:
            nodes[node_idx][1] = child_indices[0]
        else:
            nodes[node_idx][1] = 0

        return node_idx

    root_idx = insert_children_recursive(min_bound, 0, n, 1)
    return nodes, voxels
```

## Subtopics

### [Octree Volume Construction](3332-octree-volume-construction.md)
Constructs a hierarchical octree data structure from a 3D volume grid to accelerate ray marching, where leaf nodes store a bitmask representing the presence of density values in their spatial extent. It differs from sibling methods by explicitly quantizing volume samples into discrete brackets to form compact leaf masks, rather than storing raw density values or using uniform voxel grids.

### [K-D Tree Construction for Volume Rendering](3424-k-d-tree-construction-for-volume-rendering.md)
This method constructs a K-D tree from a Sparse Voxel Octree (SVO) to optimize volume rendering and mesh extraction by partitioning space based on Quadratic Error Function (QEF) variance. It differs from sibling methods like [Octree Volume Construction](3332-octree-volume-construction.md) by explicitly balancing the split plane to minimize the error difference between child nodes, rather than simply splitting at geometric centers or based on occupancy alone. This approach facilitates efficient contouring and mesh extraction by ensuring that each node's QEF solution is representative of its contained geometry.

## Operating characteristics
- **Time Complexity**: $O(N \log N)$ due to sorting the Morton codes, where $N$ is the number of input points. The recursive construction is $O(N)$ as each point is visited once.
- **Memory Complexity**: $O(N)$ for storing the nodes and voxels. The node size is fixed at 8 bytes, and voxel size is 4 bytes.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales well with the number of points, but the depth is limited to 12, which caps the resolution at $4096^3$ voxels.
- **Strengths**: Simple to implement, guarantees spatial locality, efficient for static scenes.
- **Limitations**: Not suitable for dynamic scenes where points are added/removed frequently, as the entire tree must be rebuilt. The fixed depth limits resolution.
- **When to use**: Static volume data, pre-processing for ray marching.
- **When not to use**: Dynamic scenes, real-time updates.

## Background
This method is a standard technique for building spatial acceleration structures for volume rendering and ray tracing. It is based on the Z-order curve (Morton order) which provides good spatial locality. The specific implementation details (8-byte nodes, 4-byte voxels, binary serialization) are from the [rayferric/voxel-engine](https://github.com/rayferric/voxel-engine) [S1, unofficial]. The BVH construction variant is from [mxbrt/svo](https://github.com/mxbrt/svo) [S2, unofficial].

## Sources
[S1] rayferric/voxel-engine, https://github.com/rayferric/voxel-engine, fetched 2026-10-24 (unofficial)
[S2] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
