# Face Visibility Check

Also called: Full Cube Face Culling

## Overview
Face Visibility Check determines whether a specific voxel face needs to be meshed based on its own visibility state, the neighbor's visibility state, and whether the voxel has already been processed [S1, S2, unofficial]. It solves the problem of redundant geometry generation by culling hidden faces between two opaque full cubes [S2, unofficial]. Unlike complex partial-shape culling which checks area coverage, this method relies on boolean opacity states to decide if a face is visible [S2, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**:
  - `voxels`: Array of voxel data implementing a `Voxel` trait with an `is_opaque()` method [S1, unofficial].
  - `voxels_shape`: Shape of the voxel grid [S1, unofficial].
  - `face`: An `OrientedBlockFace` defining the orientation (normal `n`, axes `u`, `v`, sign `n_sign`) [S1, unofficial].
  - `visited`: A mutable boolean array tracking processed faces to prevent duplicates [S1, unofficial].
  - `current_index`: Linear index of the current voxel in the array [S1, unofficial].
- **Outputs**: `bool` indicating if the face should be meshed (`true`) or culled (`false`) [S1, unofficial].

**Data Structures**
- `FaceStrides`: Contains `n_stride`, `u_stride`, `v_stride`, and `visibility_offset` [S1, unofficial].
  - `visibility_offset`: Calculated as `n_stride` if `n_sign > 0`, otherwise `-n_stride` [S1, unofficial]. This offset points to the adjacent voxel sharing the face [S1, unofficial].
- `OrientedBlockFace`: Contains `n_sign`, `permutation`, `n`, `u`, `v` [S1, unofficial].

**Steps**
1. Return `false` if the current voxel is `Empty` (not opaque) or if the voxel index is already marked as `visited` [S1, unofficial].
2. Retrieve the adjacent voxel using the voxel's stride plus the visibility offset [S1, unofficial].
3. Determine the opposite face of the neighbor to check [S2, unofficial].
4. Return `true` if the adjacent voxel is `Empty` [S1, unofficial].
5. Return `true` if the adjacent voxel is `Translucent` AND the current voxel is `Opaque` [S1, unofficial].
6. Return `false` if the adjacent voxel is `Opaque` [S1, unofficial].
7. For two opaque full cubes, if both the current face and the neighbor's opposite face are opaque, cull the face [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def face_visibility_check(voxels, voxels_shape, face, visited, current_index):
    # 1. Check current voxel state
    if not voxels[current_index].is_opaque():
        return False
    if visited[current_index]:
        return False

    # 2. Calculate neighbor index using stride and visibility offset
    n_stride = voxels_shape.linearize(face.n.to_array())
    visibility_offset = n_stride if face.n_sign > 0 else -n_stride
    neighbor_index = current_index + visibility_offset

    # [G] Boundary guard: out-of-bounds neighbors are treated as empty space
    if neighbor_index < 0 or neighbor_index >= len(voxels):
        return True

    # 3. Determine opposite face (helper for partial shapes, symmetric for full cubes)
    def opposite(f):
        if f == "North": return "South"
        if f == "South": return "North"
        if f == "East": return "West"
        if f == "West": return "East"
        if f == "Up": return "Down"
        if f == "Down": return "Up"
    _ = opposite(face.n)

    # 4. Check neighbor state
    if not voxels[neighbor_index].is_opaque():
        # Neighbor is Empty or Translucent
        # If Empty, we mesh. If Translucent and current is Opaque, we mesh [S1, unofficial]
        return True

    # 5. Both are opaque -> Cull
    return False
```

**Parameters**
- `visited`: Must be initialized to `false` for the current slice before processing [S1, unofficial].

**Edge Cases and Pitfalls**
- **Transparent Voxels**: Faces between two transparent voxels are not meshed by this logic [S1, unofficial]. The source notes a TODO to extend the `IsOpaque` trait for different transparency levels [S1, unofficial].
- **Partial Shapes**: This method assumes full cubes [S2, unofficial]. For partial shapes (stairs, slabs), boolean face solidity is insufficient because part of a face may be covered while another is open [S2, unofficial].
- **Boundary Conditions**: Failing to check array bounds for `neighbor_index` causes out-of-bounds reads or index wrapping, which incorrectly culls or emits faces at chunk edges [G].

## Operating characteristics
- **Strengths**: Simple to implement; effectively removes redundant vertices, indices, and rasterization work for opaque blocks [S2, unofficial].
- **Limitations**: Does not handle partial shapes correctly; requires additional logic for transparency levels [S1, S2, unofficial].

## Background
The method is used in voxel meshing implementations to optimize geometry generation by eliminating internal faces between opaque blocks [S1, S2, unofficial]. It forms the core culling rule for full-cube voxel rendering before extending to partial shapes [S2, unofficial].

## Sources
[S1] GitHub - bonsairobo/block-mesh-rs: Generate voxel block meshes in Rust ..., https://github.com/bonsairobo/block-mesh-rs, fetched 2026-10-27 (unofficial)
[S2] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
