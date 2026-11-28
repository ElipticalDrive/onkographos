# Nearest Neighbor SDF Tree Traversal

Also called: SDF Tree Query

## Overview
Nearest Neighbor SDF Tree Traversal is an algorithm for computing the signed distance from a query point to a scene represented by a hierarchical Signed Distance Function (SDF) tree [S1, unofficial]. It solves the problem of efficiently evaluating complex, potentially infinite or large-scale SDF scenes by traversing a dual-buffer structure (downtree and uptree) that supports boolean operations and geometric primitives [S1, unofficial]. Unlike standard octree traversal which samples voxel data, this method computes exact or approximate analytical distances using union pruning and continuous axis-aligned (CAA) cloning, making it suitable for ray marching in implicit surface rendering [S1, unofficial].

## How it works
The algorithm maintains a stack of frames, one per tree level, to track the state of the traversal. It processes the downtree (structural hierarchy) and uptree (operation evaluation) in a single pass, using pruning to skip irrelevant subtrees.

### Inputs and Outputs
- **Input**: A query point $\mathbf{p}$ in world space; a hierarchical SDF tree structure containing `downtree_buffer` and `uptree_buffer`.
- **Output**: The signed distance $d$ from $\mathbf{p}$ to the scene surface.

### Data Structures
The core state is held in a `LevelStackEntry` array (`point_stack`), indexed by tree level.

**LevelStackEntry Layout**:
- `branch_points`: `[Vec4; 2]` (or `[Point; 2]`). Stores the current query point and a secondary point for branching logic.
- `branch_dists`: `[f32; 2]` (or `[f64; 2]`). Stores the computed distance for the left and right branches of the current node. Initialized to $\infty$.
- `fill_idx`: `usize`. Index into `branch_dists` indicating how many valid distance values are stored for the current level.

**Tree Node Layout (Downtree Block)**:
- `level`: `u32`. Depth in the tree.
- `len`: `usize`. Number of subsequent uptree blocks associated with this node.
- `parent_is_union`: `bool`. Flag indicating if the parent node is a union operation (enables pruning).
- `is_primitive`: `bool`. True if this node represents a geometric primitive (e.g., sphere).
- `op_code`: `u32`. Operation type (e.g., union, intersection, primitive type).
- `op_specific`: Struct containing operation parameters (e.g., smooth radius, CAA clone vectors).
- `bounding_box`: `SdfBoundingBox`. Axis-aligned or oriented bounding box of the node.
- `other_box`: `SdfBoundingBox`. The bounding box of the sibling node in a binary operation (used for pruning).

**Tree Node Layout (Uptree Block)**:
- `level`: `u32`. Depth in the tree.
- `op_code`: `u32`. Operation type (e.g., union, intersection).
- `op_specific`: Struct containing operation parameters (e.g., `smooth_radius` for smooth unions).

### Numbered Steps

1.  **Initialization**:
    - Set `dt_index = 0`, `ut_index = 0`, `last_dt_level = 0`.
    - Initialize `point_stack[1]` (root level):
        - `branch_points = [point, Vec4::ZERO]`
        - `branch_dists = [INFINITY, INFINITY]`
        - `fill_idx = 0`
    - Initialize `last_ut_level = u32::MAX` (or equivalent max value).

2.  **Main Traversal Loop**:
    While `dt_index < sdf_tree.downtree_buffer_len`:
    a.  **Read Downtree Block**:
        Let `dt_block = sdf_tree.downtree_buffer[dt_index]`.
        Let `dt_point = point_stack[dt_block.level].branch_points[0]`.

    b.  **Union Pruning**:
        If `dt_block.parent_is_union` is true:
        - Compute `this_mindist = mindist(dt_block.bounding_box, dt_point)`.
        - Compute `maxdist_sibling = maxdist(dt_block.other_box, dt_point)`.
        - If `this_mindist > 0` AND (`this_mindist > dt_ut_prune_cmp` OR `this_mindist > maxdist_sibling`):
            - Skip this subtree:
                - `dt_index += 1 + dt_block.len`
                - `ut_index += 1 + dt_block.len`
                - Continue to next iteration.

    c.  **Uptree Propagation**:
        If `dt_block.level < last_dt_level`:
        - Enter loop to process pending uptree blocks:
            - Let `ut_block = sdf_tree.uptree_buffer[ut_index]`.
            - If `ut_block.level >= last_ut_level`, break loop.
            - Let `child_frame = point_stack[ut_block.level + 1]`.
            - Let `lbranch_dist = child_frame.branch_dists[0]`.
            - Let `rbranch_dist = child_frame.branch_dists[1]`.
            - Let `ut_frame = point_stack[ut_block.level]`.
            - Compute `result = uptree_dispatch(ut_block.op_code, ut_block.op_specific, lbranch_dist, rbranch_dist)`.
            - Store `ut_frame.branch_dists[ut_frame.fill_idx] = result`.
            - Increment `ut_frame.fill_idx`.
            - Update `last_ut_level = ut_block.level`.
            - Increment `ut_index`.

    d.  **Primitive Case**:
        If `dt_block.is_primitive`:
        - Let `this_frame = point_stack[dt_block.level]`.
        - Compute local point: $\mathbf{p}_{local} = M^{-1} \mathbf{p}$, where $M$ is `dt_block.bounding_box.trans_inverse`.
        - Compute distance: `dist = prim_dispatch(dt_block.op_code, dt_block.op_specific, p_local)`.
        - Store `this_frame.branch_dists[this_frame.fill_idx] = dist`.
        - Increment `this_frame.fill_idx`.
        - Increment `ut_index`.

    e.  **Non-Primitive Case**:
        If not primitive:
        - Let `child_frame = point_stack[dt_block.level + 1]`.
        - Compute child points: `child_points = downtree_dispatch(dt_block.op_code, dt_block.op_specific, dt_block.bounding_box.trans_inverse * dt_point)`.
        - Store `child_frame.branch_points = child_points`.
        - Reset `child_frame.fill_idx = 0`.
        - Reset `child_frame.branch_dists = [INFINITY, INFINITY]`.

    f.  **Increment**:
        - Update `last_dt_level = dt_block.level`.
        - Increment `dt_index`.

3.  **Finish Propagation**:
    While `ut_index < sdf_tree.uptree_buffer_len`:
    - Repeat steps 2c (Uptree Propagation) logic for remaining uptree blocks.

4.  **Return Result**:
    - Return `point_stack[1].branch_dists[0]`.

### Formulas

**Mindist for Bounding Box**:
Computes the distance from point $\mathbf{p}$ to bounding box defined by matrix $M$ and scale $s$.
$$
\mathbf{q}_{local} = (|M^{-1}\mathbf{p}| - 1.0) \cdot s
$$
$$
d = \max(0, \|\mathbf{q}_{local}\|) + \max(0, \max(q_x, q_y, q_z)) \cdot \min(0, \dots) \quad \text{(Implementation detail varies)}
$$
Specific implementation from source:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
q_local = (trans.abs() - 1.0) * bbox.scale
d = q_local.max(0).length() + q_local.x.max(q_local.y.max(q_local.z)).min(0.0)
```
Note: The second term in the source implementation `q_local.x.max(q_local.y.max(q_local.z)).min(0.0)` effectively adds 0 if the point is inside the box (since max component is $\le 0$ in local space relative to center? No, local space is normalized. If inside, components are in $[-1, 1]$. The formula seems to be a specific optimization for OBBs or specific bounding box types). For a standard AABB centered at 0 with half-extents $h$, $d = \max(0, |x|/h_x - 1)h_x + \dots$. The source formula uses `trans.abs()` which implies transformation to local space.

**Sphere Primitive Distance**:
$$
d = \|\mathbf{p}_{local}\| - r
$$
where $\mathbf{p}_{local}$ is the point in the primitive's local space and $r$ is the radius.

**CAA Clone Downtree Transform**:
Used for cloning geometry along a lattice.
$$
\mathbf{p}_{new} = \mathbf{p} - \mathbf{d} \cdot \text{round}\left(\frac{\mathbf{p}}{\mathbf{d}}\right) \cdot \text{clamp}\left(\frac{\mathbf{p}}{\mathbf{d}}, \mathbf{n}, \mathbf{p}\right)
$$
where $\mathbf{d}$ is displacement (lattice spacing), $\mathbf{n}$ is neg_limit, $\mathbf{p}$ is pos_limit.
Implementation:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
point - op_specific.vec4s[0] * (point / op_specific.vec4s[0]).round().clamp(op_specific.vec4s[1], op_specific.vec4s[2])
```

### Parameters
- **SdfUnion smooth_radius**: Stored in `op_specific.floats[0]`. Controls the blending radius for smooth unions.
- **SdfCaaClone displacement**: `Vec3`. Defines the lattice spacing for cloning.
- **SdfCaaClone neg_limit**: `Vec3`. Defines the negative boundary of the clone region.
- **SdfCaaClone pos_limit**: `Vec3`. Defines the positive boundary of the clone region.

### Edge Cases and Pitfalls
- **Precision**: Floating-point precision issues may arise in `mindist` calculations for points very close to bounding box boundaries.
- **Empty Subtrees**: Pruning relies on `parent_is_union`. If this flag is incorrect, pruning may skip valid geometry.
- **Stack Depth**: The `point_stack` must be large enough to accommodate the maximum depth of the tree.
- **Infinity Handling**: `branch_dists` initialized to `INFINITY`. If no valid path is found, the result may be `INFINITY`.

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of nodes visited. Pruning reduces the effective $N$ for sparse scenes.
- **Memory Complexity**: $O(D)$ where $D$ is the depth of the tree, for the `point_stack`.
- **Strengths**: Efficient for large-scale scenes with hierarchical structure; supports complex boolean operations and cloning; pruning accelerates traversal.
- **Limitations**: Requires pre-computed downtree and uptree buffers; less suitable for dynamic scenes without rebuilding the tree; precision depends on bounding box approximations.

## Background
The method is described in the `holger-kibur/sdf` repository [S1, unofficial]. It implements a stack-based traversal for SDF trees, supporting union pruning and CAA cloning. The approach is designed for efficient evaluation of hierarchical SDF representations, likely used in real-time rendering or simulation contexts where implicit surfaces are queried frequently.

## Sources
[S1] holger-kibur/sdf, https://github.com/holger-kibur/sdf, fetched 2026-10-27 (unofficial)
