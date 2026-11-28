# Hardware Accelerated Voxel Raycasting with Compact Descriptors

Also called: Compact Descriptor Octree Traversal, Octree Raycasting with Descriptor Buffer

## Overview
This method performs ray traversal of a Sparse Voxel Octree (SVO) by encoding the tree structure into a compact descriptor buffer, allowing the GPU to traverse the hierarchy without fetching individual node pointers. It builds on [Octree Iteration with Culling](11319-octree-iteration-with-culling.md) by replacing the CPU-driven stack management with a hardware-friendly, bit-packed traversal state that minimizes memory bandwidth. Unlike traditional SVO raycasting which may require multiple texture lookups per step, this approach uses a single descriptor lookup per level descent, optimizing for parallel execution on modern graphics hardware.

## How it works
### Inputs and Outputs
**Inputs:**
- Ray origin ($O$) and direction ($D$) in world space.
- Octree dimensions ($N$), assumed to be a power of two.
- Compact descriptor buffer containing the SVO structure.
- Voxel data buffer containing density/color values.

**Outputs:**
- Hit position, normal, and material data, or a failure status.

### Data Structures
The method relies on specific lookup tables and state structures defined in the source implementation [S1, unofficial].

**Lookup Tables:**
Two constant arrays of 8 `uint8_t` values are used for bit manipulation during traversal:
1. `mask_8`: Maps child index (0-7) to a single bit mask.
   $$ \text{mask\_8} = \{0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80\} $$
2. `count_mask_8`: Maps child index (0-7) to a cumulative count mask (used for calculating offsets).
   $$ \text{count\_mask\_8} = \{0x01, 0x03, 0x07, 0x0F, 0x1F, 0x3F, 0x7F, 0xFF\} $$

**Traversal State:**
The state is maintained in a stack-like structure to handle the hierarchy:
- `idx_stack`: Array storing the bit mask of the current child index at each scale level.
- `parent_stack`: Array storing the descriptor value (or parent pointer) for each level.
- `parent_stack_index`: Array storing the index offset into the child descriptor array for each level.
- `scale`: Current depth level in the octree.
- `parent_stack_position`: Pointer to the current top of the stack.
- `oct_pos`: Current voxel position in the grid.
- `jump_power`: Distance multiplier for stepping through the grid, derived from the current scale.

### Algorithm Steps
1. **Initialization**:
   - Calculate `delta_t`, the distance to cross one voxel in each axis:
     $$ \Delta t_x = \frac{1.0}{|D_x|}, \quad \Delta t_y = \frac{1.0}{|D_y|}, \quad \Delta t_z = \frac{1.0}{|D_z|} $$
   - Determine `voxel_step` direction:
     $$ \text{voxel\_step}_x = \begin{cases} 1 & \text{if } D_x > 0 \\ -1 & \text{if } D_x < 0 \end{cases} $$
     (Similarly for $y, z$).
   - Calculate initial intersection distances to the first voxel boundary:
     $$ t_x = \Delta t_x \cdot ( \lceil O_x \rceil - O_x ) \cdot \text{voxel\_step}_x $$
     (Similarly for $y, z$).
   - Correct for sign mirroring if $t < 0$:
     $$ t_x = t_x - \Delta t_x \cdot \text{step}(t_x < 0) $$
     (Similarly for $y, z$).
   - Initialize `jump_power`:
     $$ \text{jump\_power} = \log_2(N) - \text{scale} $$
   - Initialize `oct_pos` to the floor of the ray origin.

2. **Main Traversal Loop**:
   While `scale` $\ge 0$:
   a. **Determine Face Crossed**:
      Find the axis with the minimum intersection distance:
      $$ \text{face\_mask}_x = (t_x \le \min(t_y, t_z)) $$
      (Similarly for $y, z$).
   b. **Update Intersection Distances**:
      $$ t_x = t_x + \Delta t_x \cdot |\text{face\_mask}_x| $$
      (Similarly for $y, z$).
   c. **Step Voxel Position**:
      $$ \text{voxel\_pos} = \text{voxel\_pos} + \text{voxel\_step} \cdot \text{face\_mask} \cdot \text{jump\_power} $$
   d. **Update Index Stack**:
      XOR the current scale's index stack with the mask corresponding to the crossed face:
      $$ \text{idx\_stack}[\text{scale}] \oplus= \text{this\_face\_mask} $$
      Where `this_face_mask` is `mask_8[0]`, `mask_8[1]`, or `mask_8[2]` depending on the axis.
   e. **Check Validity**:
      Retrieve the parent descriptor and check if the child indicated by `idx_stack[scale]` exists:
      $$ \text{is\_valid} = (\text{parent\_stack}[\text{parent\_stack\_position}] \gg 16) \& \text{mask\_8}[\text{idx\_stack}[\text{scale}]] $$
   f. **Handle Invalid or Leaf**:
      If `!is_valid` or `scale` is 0 (leaf):
      - If invalid, pop the stack:
        $$ \text{jump\_power} *= 2 $$
        $$ \text{scale}-- $$
        $$ \text{parent\_stack\_position}-- $$
        Update `current_descriptor` from `parent_stack`.
      - If valid and not leaf, traverse down:
        $$ \text{jump\_power} /= 2 $$
        $$ \text{scale}++ $$
   g. **Descent Logic (if valid and not leaf)**:
      - Calculate child count offset:
        $$ \text{count} = \text{count\_bits}((\text{uint8\_t})(\text{parent\_stack}[\text{parent\_stack\_position}] \gg 16) \& \text{count\_mask\_8}[\text{idx\_stack}[\text{scale}]]) - 1 $$
      - Update parent stack index for next level:
        $$ \text{parent\_stack\_index}[\text{parent\_stack\_position} + 1] = \dots + \text{count} $$
      - Update `oct_pos` based on the child index bits:
        If the x-bit of the child index is set:
        $$ \text{oct\_pos}_x += \text{jump\_power} / 2 $$
        $$ \text{idx\_stack}[\text{scale}] |= \text{idx\_set\_x\_mask} $$
        (Repeat for y and z axes).
   h. **Sample Voxel Data**:
      Fetch data at `voxel_pos`. If `voxel_data != 0`, return hit.

### Parameters
- `jump_power`: Initial jump multiplier based on octree depth and current scale.
- `intersection_t`: Distance to the next voxel boundary intersection on each axis.

### Edge Cases and Pitfalls
- **Far Pointers**: When the relative distance exceeds `0x8000`, far pointers are used. The descriptor is ORed with `far_bit_mask` and the relative distance to the far pointer block [S1, unofficial].
- **Precision**: Floating point precision issues may arise near voxel boundaries; the sign correction step is critical for robustness [S1, unofficial].
- **Empty Branches**: The validity check handles empty branches by popping the stack, ensuring no invalid memory accesses occur.

## Operating characteristics
- **Time Complexity**: $O(\log N)$ for traversal depth, but the number of steps depends on the sparsity and ray length. Each step involves constant-time bit operations.
- **Memory Complexity**: $O(N^3)$ for the voxel data, but the descriptor buffer is compact, storing only non-empty nodes.
- **Performance**: Optimized for GPU execution due to minimal branching and memory accesses. The use of compact descriptors reduces bandwidth compared to traditional pointer-based octrees.
- **Strengths**: High performance on hardware with good bit-manipulation support; efficient memory usage for sparse volumes.
- **Limitations**: Requires pre-processing to build the compact descriptor buffer; less flexible for dynamic updates compared to linked-list based SVOs.

## Background
This method is derived from the "voxel-raycaster" project by Mitchell Hansen [S1, unofficial]. It represents an implementation of hardware-accelerated raycasting techniques specifically tailored for sparse voxel octrees, leveraging GPU capabilities for efficient traversal. The approach is part of the broader effort to make real-time volumetric rendering feasible on consumer hardware.

## Sources
[S1] GitHub - MitchellHansen/voxel-raycaster: Hardware accelerated voxel ray ..., https://github.com/MitchellHansen/voxel-raycaster, fetched 2026-10-25 (unofficial)
