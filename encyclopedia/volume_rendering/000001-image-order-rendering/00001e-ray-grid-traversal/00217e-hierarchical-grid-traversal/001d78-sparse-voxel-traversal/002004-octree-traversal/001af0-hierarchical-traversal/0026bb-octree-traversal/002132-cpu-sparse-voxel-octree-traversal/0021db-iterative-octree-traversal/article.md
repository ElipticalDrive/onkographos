# Iterative Octree Traversal

Also called: Octree Ray Tracing (Software), Octree Traversal Algorithm, OctreeSWRaytracer, SVO-workbench raycaster, stack-based traversal

## Overview
Iterative Octree Traversal is a CPU-based ray traversal algorithm for Sparse Voxel Octrees (SVO) that uses a fixed-size stack to manage traversal state, handling valid internal nodes, leaf nodes, and empty space by casting rays to the next occupied voxel [S2, unofficial]. It employs a two-phase approach: a register-based DDA with bit-masked coordinate tracking for upper octree levels, and a slab-boundary-based stack traversal for deeper nodes [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin $(o_x, o_y, o_z)$, direction $(d_x, d_y, d_z)$, octree root address, octree bounds $(min_x, min_y, min_z)$ to $(max_x, max_y, max_z)$ [S2].
- Output: Sequence of visited nodes (internal and leaf) for shading or density integration [S1].

**Data Structures:**
- **Cell**: A fixed-sized block representing an octant, implemented as a simple array of eight elements. Each element contains a pointer to a descendent cell or a null pointer if the sub-octant is empty [S1].
- **Stack Item (`stack_item_t`)**: Contains slab boundaries and node state:
  - `tnum tx0, ty0, tz0`: Entry slab boundaries (fixed-point).
  - `tnum tx1, ty1, tz1`: Exit slab boundaries (fixed-point).
  - `int adr`: Node address in memory.
  - `char idx`: Child index (0-7) of the current node within its parent.
- **State Struct (`value`)**: Used in some implementations (e.g., [S3, unofficial]):
  - `m_node_address` (uint32_t): Current node address.
  - `m_child_num` (uint32_t): Next child to visit.
  - `m_morton_code` (uint32_t): Current Morton code.
- **XYZ Variable**: A register variable containing the three coordinates of the first-step voxel. Its width in bits equals the number of levels in the first step [S1].
- **Mask Variable**: Initially set with higher-order bits for each coordinate ($0\ x\ x\ ...\ x\ 0\ y\ y\ ...\ y\ 0\ z\ z\ ...\ z$) and lower bits as $0\ 1\ 0\ ...\ 0\ 0\ 1\ 0\ ...\ 0\ 0\ 1\ 0\ ...\ 0$. Used to filter coordinate bits and control octree level [S1].

**Parameters:**
- `tnum_scale`: Fixed-point scaling factor for slab coordinates, typically 10000 [S2, unofficial].
- `STACK_SIZE`: Maximum depth of the traversal stack, typically 100 [S2, unofficial].
- `terminLevel`: Stop level for traversal, typically 10 [S2, unofficial].

**Steps:**

1. **Initialization**:
   - If any ray direction component is zero, add 0.001 to avoid division by zero [S2, unofficial].
   - If a direction component is negative, negate the origin and direction components for that axis, and set the corresponding bit in `dir_mask` ($x=4, y=2, z=1$) [S2, unofficial].
   - Compute slab entry and exit distances for the octree bounds:
     $$tx_0 = (min_x - o_x)/d_x, \quad tx_1 = (max_x - o_x)/d_x$$
     $$ty_0 = (min_y - o_y)/d_y, \quad ty_1 = (max_y - o_y)/d_y$$
     $$tz_0 = (min_z - o_z)/d_z, \quad tz_1 = (max_z - o_z)/d_z$$
     [S2, unofficial]
   - Scale all slab boundaries by `tnum_scale` to convert them to integer coordinates (`tnum`) [S2, unofficial].
   - Initialize 3D DDA variables: Set `XYZ` and `mask`. Set `cell` to octree root. Push `cell` to stack. Set `i_c=0`. Set `end condition = mask << 1`. Set `XYZ_ant = XYZ` [S1].
   - Initialize traversal state: Set stack index and level to 0, mark as first node, get root node entry, and decode child validity/leaf status into arrays [S2, unofficial].

2. **Main Loop**:
   - While `((XYZ and end condition) == 0)` (i.e., inside octree and no carry bit set):
     - **Ascend**:
       - `XYZ_ant = XYZ_ant XOR XYZ` [S1].
       - `mask = mask << 1` [S1].
       - While `(XYZ_ant and mask)` is non-zero:
         - Pop from stack [S1].
         - `mask = mask >> 1` [S1].
     - **Descend**:
       - `mask = mask >> 1` [S1].
       - Pop `cell` [S1].
       - Separate X, Y, Z from `XYZ` [S1].
       - `bool = TRUE` [S1].
       - While `bool`:
         - Push `cell` [S1].
         - Calculate octant index `i` from Z, Y, X bits:
           - If `(Z and mask) != 0`, `i = i + 4` [S1].
           - If `(Y and mask) != 0`, `i = i + 2` [S1].
           - If `(X and mask) != 0`, `i = i + 1` [S1].
         - If `(mask and 1) == 0` (not last level):
           - If `cell[i] != 0`:
             - `mask = mask >> 1` [S1].
             - `cell = cell[i]` [S1].
           - Else:
             - `bool = FALSE` [S1].
         - Else:
           - `bool = FALSE` [S1].
         - If `cell[i] != 0` and not last level:
           - Initialize second step [S1].
           - Execute second step [S1].
     - Execute 3D DDA [S1].

3. **Second Step (`procSubtree2`)**:
   - Compute child slab boundaries by selecting either the current slab boundary or the midpoint based on the child index bits, then flip indices using `dir_mask` [S2, unofficial].
   - Calculate child entry and exit slab distances, then cull the child if its exit distance is less than or equal to the current traversal start distance (`t_start`) [S2, unofficial].
   - If the child is valid and not culled:
     - Push its slab boundaries and address onto the stack [S2, unofficial].
     - Update `t_start` [S2, unofficial].
     - Continue traversal [S2, unofficial].
   - Otherwise, backtrack or terminate [S2, unofficial].

4. **Next Node Logic**:
   - If `m_child_num >= 8`, pop from stack [S3, unofficial].
   - If stack empty, set `m_finished = true` and return empty [S3, unofficial].
   - Otherwise, increment `m_child_num` of the new current node [S3, unofficial].
   - Check if null, leaf, or reached stop level. If so, increment `m_child_num` and prepare to return the result [S3, unofficial].
   - If the node is an address (internal), push current state to stack, update current state to the child's address (`val`), reset `m_child_num` to 0, and shift `m_morton_code` left by 3. Continue loop [S3, unofficial].
   - If the node is a leaf or reached stop level (and not null), return the current state as the result [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iterative_octree_traversal(ray_origin, ray_direction, octree_root, octree_bounds, levels_first_step=3):
    dir_mask = 0
    # Handle zero and negative direction components
    for axis, mask_bit in [('x', 4), ('y', 2), ('z', 1)]:
        if ray_direction[axis] == 0:
            ray_direction[axis] = 0.001
        if ray_direction[axis] < 0:
            ray_origin[axis] = -ray_origin[axis]
            ray_direction[axis] = -ray_direction[axis]
            dir_mask |= mask_bit

    # Compute slab boundaries
    min_x, min_y, min_z = octree_bounds[0]
    max_x, max_y, max_z = octree_bounds[1]
    tx0 = (min_x - ray_origin.x) / ray_direction.x
    tx1 = (max_x - ray_origin.x) / ray_direction.x
    ty0 = (min_y - ray_origin.y) / ray_direction.y
    ty1 = (max_y - ray_origin.y) / ray_direction.y
    tz0 = (min_z - ray_origin.z) / ray_direction.z
    tz1 = (max_z - ray_origin.z) / ray_direction.z

    tnum_scale = 10000
    tx0, tx1 = int(tx0 * tnum_scale), int(tx1 * tnum_scale)
    ty0, ty1 = int(ty0 * tnum_scale), int(ty1 * tnum_scale)
    tz0, tz1 = int(tz0 * tnum_scale), int(tz1 * tnum_scale)

    # Initialize XYZ and mask for first step
    xyz = 0
    mask = 0
    for i in range(levels_first_step):
        mask |= (1 << (i * 3 + 2)) | (1 << (i * 3 + 1)) | (1 << (i * 3 + 0))
    end_condition = mask << 1
    xyz_ant = xyz

    stack = [{'tx0': tx0, 'ty0': ty0, 'tz0': tz0, 'tx1': tx1, 'ty1': ty1, 'tz1': tz1, 'adr': octree_root, 'idx': 0}]
    m_child_num = 0
    m_morton_code = 0
    m_finished = False
    t_start = 0.0

    while not m_finished and stack:
        # First step DDA loop
        while (xyz & end_condition) == 0:
            xyz_ant = xyz_ant ^ xyz
            mask = mask << 1
            # Ascend octree
            while (xyz_ant & mask) != 0:
                stack.pop()
                mask = mask >> 1
            # Descend octree
            mask = mask >> 1
            cell = stack.pop()
            # Separate X, Y, Z from xyz (conceptual)
            bool_flag = True
            while bool_flag:
                stack.append(cell)
                i = ((cell['z'] & mask) >> 2) + ((cell['y'] & mask) >> 1) + ((cell['x'] & mask) >> 0)
                if (mask & 1) == 0:  # Not last level
                    if cell['children'][i] != 0:
                        mask = mask >> 1
                        cell = cell['children'][i]
                    else:
                        bool_flag = False
                else:
                    bool_flag = False
                if cell['children'][i] != 0 and (mask & 1) == 0:
                    # Initialize second step
                    pass
            # Execute 3D DDA
            pass

        # Second step / procSubtree2 logic
        if m_child_num >= 8:
            stack.pop()
            if not stack:
                m_finished = True
            else:
                m_child_num += 1
                continue

        current = stack[-1]
        node = octree_read(current['adr'] + m_child_num)
        if node is None or node.is_leaf or reached_stop_level:
            m_child_num += 1
            continue

        # Internal node
        stack.append(current)
        current['adr'] = node.val
        m_child_num = 0
        m_morton_code = (m_morton_code << 3) | m_child_num

        # Compute child slab boundaries
        # Select current slab boundary or midpoint based on child index bits
        # Flip indices using dir_mask
        # Calculate child entry/exit distances
        # Cull if exit <= t_start
        # Push valid/culled child to stack, update t_start
```

## Subtopics

### ESVO (Efficient Sparse Voxel Octrees)
A port of the 'Efficient Sparse Voxel Octrees' algorithm, specifically excluding contour generation and beam optimization features [S4, unofficial]. It is an octree-based traversal method that is a near 1:1 port of the original ESVO implementation [S4, unofficial]. Node encoding was changed: relative pointers are 'backwards' to allow trees to be built without intermediate buffers (parent nodes appear after children), and far pointers are signaled by a reserved upper range rather than a dedicated bit [S4, unofficial]. Memory cost is approximately 4 bytes per node [S4, unofficial]. Nodes are very small (4 bytes each) and localized, with a z-curve layout resulting from building the tree in depth-first order [S4, unofficial]. Performance is underwhelming because it takes many extra iterations to ascend and descend the tree and advance positions by DDA [S4, unofficial]. Editing cost is approximately O(log m), leveraging path copying for persistent data structures [S4, unofficial]. On Sponza 2k, ESVO achieves 95.0 Mrays/s for primary rays and 66.2 Mrays/s for primary + 1 diffuse bounce [S4, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**:
  - On Sponza 2k (499k * 8³ voxels): ESVO achieves 95.0 Mrays/s for primary rays and 66.2 Mrays/s for primary + 1 diffuse bounce [S4, unofficial].
  - On Eco House 2k (448k * 8³ voxels): ESVO achieves 67.1 Mrays/s for primary rays and 52.7 Mrays/s for primary + 1 diffuse bounce [S4, unofficial].
  - On Forest Lake 2k (807k * 8³ voxels): ESVO achieves 67.1 Mrays/s for primary rays and 52.7 Mrays/s for primary + 1 diffuse bounce [S4, unofficial].
- **Scaling**: Not given in the sources.
- **Strengths**: Small node size (4 bytes) and localized memory access due to z-curve layout [S4, unofficial].
- **Limitations**: Underwhelming performance due to many extra iterations for ascension/descension and DDA advancement [S4, unofficial].
- **When to use**: When memory bandwidth is low and node size is critical [S4, unofficial].

## Background
The method is based on the work of [S1] Discrete Ray-Tracing of Huge Voxel Spaces, which presents an octree traversal algorithm for huge voxel spaces. The implementation details are also found in [S2, unofficial] skyfex/svo-workbench, [S3, unofficial] loryruta/voxelizer, and [S4, unofficial] dubiousconst282/VoxelRT.

## Sources
[S1] Discrete Ray-Tracing of Huge Voxel Spaces, https://doczz.net/doc/7276913/discrete-ray-tracing-of-huge-voxel-spaces, fetched 2026-10-25
[S2] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S3] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S4] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
