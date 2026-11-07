# Voxel Filled Check

Also called: Filled, Intersection Detection, Voxel Occupancy Check

## Overview
Voxel Filled Check determines whether a specific voxel, brick, or chunk within a structured volume grid is occupied by geometry or data [S3, unofficial]. It acts as a binary occupancy test for pre-computed voxel masks [S1, unofficial].

## How it works
The method checks occupancy by accessing a bitmask texture or array associated with a volume block (brick/chunk) [S1, unofficial]. It supports two modes: a metadata-driven check and a direct bitmask check [S2, unofficial].

**Inputs:**
- `coord`: A 3D integer coordinate $(x, y, z)$ representing the voxel position.
- `global_idx`: The global linear index of the voxel in the volume grid.
- `option_.WithMeta()`: A boolean flag indicating if voxel metadata is available and should be used.
- `voxel_metas_`: An array of metadata objects (if metadata mode is active).
- `output_lb_`, `output_ub_`: Lower and upper bounds of the valid output region.
- `voxel_batch`: A bitmask array or texture storing occupancy bits.
- `kBatchSize`: The number of voxels packed into one integer/bitset unit.
- `VX_MASK_EXT`: The extent (side length) of a single mask block (brick/chunk).

**Outputs:**
- `bool`: `true` if the voxel is occupied, `false` otherwise.

**Data Structures:**
- **Bitmask Block**: A texture or array where each element represents a block of voxels. The size is determined by `VX_MASK_EXT` [S1, unofficial].
- **Linear Index**: A 1D index into the bitmask block calculated from 3D coordinates [S1, unofficial].

**Steps:**
1. **Calculate Local Coordinates**: Compute the local position within the mask block using bitwise AND with the mask extent minus one.
   $$ local = coord \ \& \ (VX\_MASK\_EXT - 1) $$
   [S1, unofficial]

2. **Calculate Linear Index**: Convert the 3D local coordinates to a 1D index within the bitmask block.
   $$ mask\_linear\_idx = local.x + local.y \cdot VX\_MASK\_EXT + local.z \cdot VX\_MASK\_EXT \cdot VX\_MASK\_EXT $$
   [S1, unofficial]

3. **Metadata Check (Optional)**: If `option_.WithMeta()` is true, retrieve the metadata object at `coord` from `voxel_metas_` and return the result of its `Filled()` method. [S2, unofficial]

4. **Boundary Check**: If metadata is not used, check if the coordinate lies within the valid output bounds defined by `output_lb_` and `output_ub_`. If the coordinate is out of bounds, return `0`. [S2, unofficial]

5. **Bit Test**: Determine the batch index and bit offset.
   $$ batch\_index = global\_idx / kBatchSize $$
   $$ bit\_offset = global\_idx \ \& \ (kBatchSize - 1) $$
   Retrieve the integer value from `voxel_batch[batch_index]` and test the bit at `bit_offset`. Return `true` if the bit is set, `false` otherwise. [S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_filled_check(coord, global_idx, option_with_meta, voxel_metas, output_lb, output_ub, voxel_batch, k_batch_size, vx_mask_ext):
    # Step 1: Calculate local coordinates within the mask block
    local_x = coord.x & (vx_mask_ext - 1)
    local_y = coord.y & (vx_mask_ext - 1)
    local_z = coord.z & (vx_mask_ext - 1)
    
    # Step 2: Calculate linear index within the bitmask block
    mask_linear_idx = local_x + local_y * vx_mask_ext + local_z * vx_mask_ext * vx_mask_ext
    
    # Step 3: Metadata check
    if option_with_meta:
        meta = voxel_metas[coord]
        return meta.filled()
    
    # Step 4: Boundary check
    if not (output_lb.x <= coord.x <= output_ub.x and
            output_lb.y <= coord.y <= output_ub.y and
            output_lb.z <= coord.z <= output_ub.z):
        return False
    
    # Step 5: Bit test
    batch_index = global_idx // k_batch_size
    bit_offset = global_idx & (k_batch_size - 1)
    
    batch_value = voxel_batch[batch_index]
    is_set = (batch_value >> bit_offset) & 1
    
    return is_set == 1
```

**Parameters:**
- `VX_MASK_EXT`: Defines the brick/chunk dimensions [S1, unofficial].
- `kBatchSize`: Defines the number of voxels packed per integer unit [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Non-Power-of-Two Extents**: The bitwise AND optimization for `local` coordinates requires `VX_MASK_EXT` to be a power of two. If not, modulo arithmetic must be used [G].
- **Out-of-Bounds Access**: Failing to check `output_lb_` and `output_ub_` before accessing `voxel_batch` can lead to memory errors or incorrect occupancy results for voxels near the volume boundary [S2, unofficial].

## Subtopics

### [Voxel Material Classification](3645-voxel-material-classification.md)
Determines if a voxel's material is transparent for ray traversal purposes, allowing rays to pass through. It extends the basic filled check by evaluating material properties (e.g., alpha threshold) rather than just occupancy. It is distinct from [Voxel Filtering](4035-voxel-filtering.md), which handles spatial smoothing or mipmapping, by focusing on discrete categorical classification rather than value interpolation. Choose it when shading requires semantic status (Air, Liquid, Solid) or transparency flags.

### [Voxel Filtering](4035-voxel-filtering.md)
Iterates over all voxels in a grid and returns a list of linear indices for voxels that satisfy a user-defined predicate function. It serves as a general-purpose selection mechanism within the voxelization pipeline, acting as a post-processing step for [Voxel Filled Check](1705-voxel-filled-check.md) to refine occupancy data. Unlike [Voxel Material Classification](3645-voxel-material-classification.md), which categorizes voxels by material type, Voxel Filtering selects voxels based on arbitrary boolean predicates, enabling tasks such as density thresholding or invalid value exclusion. Choose it when post-processing occupancy masks for custom spatial conditions.

### Voxel Solidity Check
A post-sampling validation step that verifies whether a sampled voxel actually contributes to the volume's opacity, used to reduce banding and rendering artifacts [S4, unofficial]. It differs from [Voxel Filled Check](1705-voxel-filled-check.md) by operating on sampled density values rather than pre-computed occupancy bits. Choose it during raycasting when visual quality requires filtering out non-contributing samples.

## Operating characteristics
- **Time Complexity**: $O(1)$ per voxel check [G]. The calculation involves bitwise operations and array lookups [G].
- **Memory Complexity**: Depends on the storage of the bitmask [G]. For a volume of size $N$, the bitmask requires $N/kBatchSize$ integers [G].
- **Strengths**: Low overhead, suitable for dense and sparse volumes [G].
- **Limitations**: Requires pre-computed occupancy masks. Does not handle partial voxel occupancy or continuous geometry directly [G].
- **When to use**: When accelerating ray traversal by skipping empty regions in voxelized volumes [S3, unofficial].
- **When not to use**: When working with unstructured meshes or continuous implicit surfaces without prior voxelization [G].

## Background
The method originates from voxel-based ray tracing implementations, specifically in the vxray project, which uses bitmask textures to represent voxel occupancy [S1, unofficial]. It is implemented as CPU and CUDA functions to detect intersections for determining occupied voxels [S3, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S2] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
[S3] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S4] jose-villegas/VolumeRendering, https://github.com/jose-villegas/VolumeRendering, fetched 2026-10-27 (unofficial)
