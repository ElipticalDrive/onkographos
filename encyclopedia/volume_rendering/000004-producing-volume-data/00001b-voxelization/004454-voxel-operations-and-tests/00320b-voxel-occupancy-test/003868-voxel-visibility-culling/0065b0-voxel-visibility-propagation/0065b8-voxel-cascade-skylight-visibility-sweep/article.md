# Voxel Cascade Skylight Visibility Sweep

Also called: Skylight Sweep

## Overview
The Voxel Cascade Skylight Visibility Sweep is a compute-shader-based algorithm that propagates skylight visibility information from coarser voxel cascades into finer ones. It solves the problem of efficiently determining which voxels in a high-resolution cascade are occluded by geometry in the scene, using precomputed visibility data from adjacent coarser levels. Unlike direct ray-marching or brute-force occlusion queries, this method leverages spatial coherence and Bresenham line traversal to sample visibility targets, significantly reducing the computational cost of skylight culling in voxel-based rendering pipelines.

## How it works
The method operates on a 2D grid of compute threads, where each thread is responsible for sweeping a line of voxels through a specific cascade level. It reads visibility data from a target cascade and an outer (coarser) cascade, combining them to update the visibility state of the current cascade.

### Inputs and Outputs
*   **Inputs**:
    *   `SkylightVisibility` buffer: Contains `visibility_targets` (uint) for the outer cascade.
    *   `VoxelCascades` buffer: Contains `voxel_targets` (uint) for the current cascade (readonly).
    *   `CommonUBO`: Uniform buffer containing cascade metadata (`cascade_size`, `cascade_mask`, `n_cascades`, `min_voxel_size`, `anchors`).
    *   `InjectUBO`: Uniform buffer containing traversal parameters (`bresenham[64]`, `origin`, `step_a`, `step_b`, `light_direction`, `n_a`, `n_b`, `target_cascade`).
*   **Outputs**:
    *   `VoxelCascades` buffer: Updated `voxel_targets` for the `target_cascade`.

### Data Structures
*   **VoxelCascades**: `std430` buffer, set 0, binding 0. Layout: `uint voxel_targets[]`.
*   **SkylightVisibility**: `std430` buffer, set 1, binding 0. Layout: `uint visibility_targets[]`.
*   **CommonUBO**: `std140` buffer, set 2, binding 0. Fields:
    *   `uvec4 cascade_size`: xyz components used.
    *   `uvec4 cascade_mask`.
    *   `uint n_cascades`.
    *   `float min_voxel_size`.
    *   `vec4 anchors[MAX_CASCADES]`.
*   **InjectUBO**: `std140` buffer, set 2, binding 1. Fields:
    *   `ivec4 bresenham[64]`: Offsets defining the Bresenham line path.
    *   `ivec4 origin`: Starting 3D voxel coordinate.
    *   `ivec4 step_a`: Increment vector for the first grid dimension.
    *   `ivec4 step_b`: Increment vector for the second grid dimension.
    *   `vec4 light_direction`.
    *   `uint n_a`: Grid dimension size A.
    *   `uint n_b`: Grid dimension size B.
    *   `uint target_cascade`: Index of the cascade being written.

### Algorithm Steps
1.  **Thread Identification**: Each compute thread identifies itself using `gl_GlobalInvocationID.x`. If `global_id >= n_a * n_b`, the thread returns immediately.
2.  **Grid Coordinates**: Compute grid coordinates `a` and `b`:
    $$ a = global\_id \pmod{n_a} $$
    $$ b = \lfloor global\_id / n_a \rfloor $$
3.  **Root Voxel Position**: Calculate the starting voxel position for the sweep:
    $$ root = origin + a \cdot step\_a + b \cdot step\_b $$
    (Note: `root` is a 3D vector derived from the 4D components).
4.  **Initialization**:
    *   Set `visibility_accumulator = 1.0`.
    *   Set `first_entry = true` if `target_cascade + 1 < n_cascades`, otherwise `false`.
5.  **Bresenham Sweep Loop**: Iterate `i` from 0 to 63:
    *   Compute current voxel position:
        $$ voxel\_pos = root + bresenham[i] $$
    *   **Boundary Check**: If any component of `voxel_pos` is outside the range $[-32, 31]$, skip to the next iteration. This prevents out-of-bounds memory access.
    *   **Outer Cascade Sampling**: If `first_entry` is true:
        *   Compute outer voxel position:
            $$ outer\_voxel\_pos = voxel\_pos / 2 $$
        *   Compute index in the outer cascade:
            $$ ix = indexFromCascadePos(target\_cascade + 1, outer\_voxel\_pos) $$
        *   Read visibility from the outer cascade:
            $$ visibility = visibility\_targets[ix] $$
        *   Set `first_entry = false`.
    *   **Accumulation**: Update `visibility_accumulator` based on `visibility` and potentially `voxel_targets` (implementation detail: the source notes imply accumulation but do not specify the exact bitwise or arithmetic operation for combining `visibility` with the current voxel's state. Typically, this involves bitwise AND or multiplication depending on whether visibility is binary or continuous. Given the `uint` type, bitwise AND is a strong candidate for occlusion culling). [G]
6.  **Write Back**: Write the final `visibility_accumulator` (or updated voxel state) to `voxel_targets` at the appropriate index for `target_cascade`.

### Parameters
*   `target_cascade`: Index of the voxel cascade currently being written to.
*   `n_cascades`: Total number of voxel cascades in the hierarchy.
*   `bresenham[64]`: Array of 64 `ivec4` offsets defining the Bresenham line path.
*   `origin`: Starting 3D voxel coordinate for the sweep grid.
*   `step_a`: Increment vector for the first grid dimension.
*   `step_b`: Increment vector for the second grid dimension.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume these are available in the shader environment
# gl_GlobalInvocationID.x
# n_a, n_b, target_cascade, n_cascades
# origin, step_a, step_b, bresenham[64]
# visibility_targets (from SkylightVisibility buffer)
# voxel_targets (from VoxelCascades buffer, read/write)

global_id = gl_GlobalInvocationID.x
if global_id >= n_a * n_b:
    return

a = global_id % n_a
b = global_id / n_a  # Integer division

root = origin + a * step_a + b * step_b

visibility_accumulator = 1.0
first_entry = (target_cascade + 1 < n_cascades)

for i in range(64):
    voxel_pos = root + bresenham[i]
    
    # Boundary check: hardcoded limits [-32, 31]
    if (voxel_pos.x < -32 or voxel_pos.x > 31 or
        voxel_pos.y < -32 or voxel_pos.y > 31 or
        voxel_pos.z < -32 or voxel_pos.z > 31):
        continue

    if first_entry:
        outer_voxel_pos = voxel_pos / 2  # Integer division
        ix = indexFromCascadePos(target_cascade + 1, outer_voxel_pos)
        visibility = visibility_targets[ix]
        first_entry = False
        
        # Combine visibility with current voxel state
        # Note: Exact combination logic (bitwise AND vs multiplication) 
        # is not fully specified in the source notes. 
        # Assuming bitwise AND for occlusion culling with uint targets.
        current_voxel = voxel_targets[...] # Index calculation for current voxel needed
        visibility_accumulator = visibility_accumulator & visibility # [G]

# Write result back to voxel_targets for target_cascade
# Index calculation for writing back is not fully specified in the source notes.
voxel_targets[...] = visibility_accumulator # [G]
```

### Edge Cases and Pitfalls
*   **Branch Divergence**: When `first_entry` is true, threads enter the outer cascade sampling block at different loop iterations, causing branch divergence. The author suggests precomputing the outer voxel on the CPU or using a DDA intersection test to mitigate this.
*   **Boundary Checks**: Hardcoded limits of -32 and 31 are used for voxel coordinates. Vectors outside this range are skipped via `continue`, preventing out-of-bounds buffer access. This assumes a fixed voxel coordinate space.
*   **Index Calculation**: The function `indexFromCascadePos` is used but not defined in the notes. It is assumed to map 3D voxel coordinates to a linear index in the `visibility_targets` array based on the cascade's dimensions.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot L)$, where $N$ is the number of threads (grid points) and $L$ is the number of Bresenham steps (64). Each thread performs a constant number of operations per step.
*   **Memory Complexity**: $O(V)$, where $V$ is the total number of voxels across all cascades, stored in `VoxelCascades` and `SkylightVisibility` buffers.
*   **Performance**: The method is efficient for large voxel grids due to its parallel nature and use of Bresenham's algorithm, which minimizes floating-point operations. However, branch divergence from `first_entry` can reduce GPU efficiency.
*   **Strengths**: Low computational overhead per voxel, efficient use of cache due to spatial coherence, and simple implementation.
*   **Limitations**: Fixed boundary checks may not generalize to all scene sizes. Branch divergence can impact performance on some GPU architectures. The exact visibility combination logic is not fully specified, which may require experimentation.

## Background
The Voxel Cascade Skylight Visibility Sweep is part of the voxel cone tracing pipeline, specifically for skylight visibility culling. It was developed as part of the `voxel-cone-tracing` project by Sandalmoth. The method builds on the principles of voxel visibility propagation, extending it to handle skylight occlusion across multiple cascade levels.

## Sources
[S1] Sandalmoth/voxel-cone-tracing, https://github.com/Sandalmoth/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
