# Face Visibility Check

Also called: Full Cube Face Culling

## Overview
Face Visibility Check determines whether a specific voxel face needs to be meshed based on its own visibility state, the neighbor's visibility state, and whether the face has already been processed [S1, S2]. It solves the problem of redundant geometry generation by culling hidden faces between two opaque full cubes [S2]. Unlike complex partial-shape culling which checks area coverage, this method relies on boolean opacity states to decide if a face is visible [S2].

## How it works
**Inputs and Outputs**
- **Inputs**:
  - `voxels`: Array of voxel data implementing a `Voxel` trait with an `is_opaque()` method [S1].
  - `voxels_shape`: Shape of the voxel grid [S1].
  - `face`: An `OrientedBlockFace` defining the orientation (normal `n`, axes `u`, `v`, sign `n_sign`) [S1].
  - `visited`: A mutable boolean array tracking processed faces to prevent duplicates [S1].
  - `current_index`: Linear index of the current voxel [S1].
- **Outputs**: `bool` indicating if the face should be meshed (true) or culled (false) [S1].

**Data Structures**
- `FaceStrides`: Contains `n_stride`, `u_stride`, `v_stride`, and `visibility_offset` [S1].
  - `visibility_offset`: Calculated as `n_stride` if `n_sign > 0`, otherwise `0u32.wrapping_sub(n_stride)` [S1]. This offset points to the adjacent voxel sharing the face.
- `OrientedBlockFace`: Contains `n_sign`, `permutation`, `n`, `u`, `v` [S1].

**Steps**
1. **Initialization**: Fill the `visited` array with `false` for the current slice processing [S1].
2. **Check Current Voxel State**: If the current voxel is `Empty` (not opaque) or the current face index is already marked `visited`, return `false` (cull) [S1].
3. **Retrieve Neighbor**: Calculate the linear index of the adjacent voxel using the `visibility_offset` from `FaceStrides` [S1].
4. **Check Neighbor State**:
   - If the adjacent voxel is `Empty`, return `true` (mesh the face, as it borders empty space) [S1].
   - If the adjacent voxel is `Translucent` AND the current voxel is `Opaque`, return `true` (mesh the face) [S1].
   - If the adjacent voxel is `Opaque`, return `false` (cull the face) [S1].
5. **Full Cube Culling Logic**: For two opaque full cubes, if both the current face and the neighbor's opposite face are opaque, cull the face [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def face_visibility_check(voxels, voxels_shape, face, visited, current_index):
    # 1. Check if current voxel is empty or already visited
    if not voxels[current_index].is_opaque():
        return False
    if visited[current_index]:
        return False

    # 2. Calculate neighbor index using visibility offset
    # Derived from S1: visibility_offset is n_stride if n_sign > 0 else -n_stride
    # Assuming face provides necessary stride info or we compute it
    n_stride = voxels_shape.linearize(face.n.to_array())
    if face.n_sign > 0:
        visibility_offset = n_stride
    else:
        visibility_offset = 0u32.wrapping_sub(n_stride)
    
    neighbor_index = current_index + visibility_offset

    # 3. Check neighbor state
    if not voxels[neighbor_index].is_opaque():
        # Neighbor is Empty or Translucent
        # If Empty, we mesh. If Translucent and current is Opaque, we mesh [S1]
        return True
    
    # 4. Both are opaque -> Cull
    return False

def opposite(face):
    # S2
    if face == Face::North: return Face::South
    if face == Face::South: return Face::North
    if face == Face::East: return Face::West
    if face == Face::West: return Face::East
    if face == Face::Up: return Face::Down
    if face == Face::Down: return Face::Up
```

**Parameters**
- `visited`: Must be initialized to `false` before processing a slice [S1].

**Edge Cases and Pitfalls**
- **Transparent Voxels**: Faces between two transparent voxels are not meshed by this logic [S1]. The source notes a TODO to extend the `IsOpaque` trait for different transparency levels [S1].
- **Partial Shapes**: This method assumes full cubes. For partial shapes (stairs, slabs), boolean face solidity is insufficient because part of a face may be covered while another is open [S2].

## Operating characteristics
- **Time Complexity**: $O(1)$ per face check [S1, S2].
- **Memory Complexity**: $O(N)$ for the `visited` array, where $N$ is the number of voxels or faces [S1].
- **Strengths**: Simple to implement; effectively removes redundant vertices and indices for opaque blocks [S2].
- **Limitations**: Does not handle partial shapes correctly; requires additional logic for transparency levels [S1, S2].

## Background
The method originates from voxel meshing libraries like `block-mesh-rs` [S1] and discussions on face culling for voxel cubes [S2]. It is a foundational step in generating efficient meshes from voxel data by eliminating internal faces between opaque blocks.

## Sources
[S1] GitHub - bonsairobo/block-mesh-rs: Generate voxel block meshes in Rust ..., https://github.com/bonsairobo/block-mesh-rs, fetched 2026-10-27 (unofficial)
[S2] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
