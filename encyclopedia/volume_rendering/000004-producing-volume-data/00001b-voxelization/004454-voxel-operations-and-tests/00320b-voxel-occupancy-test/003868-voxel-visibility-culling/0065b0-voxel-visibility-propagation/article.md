# Voxel Visibility Propagation
Also called: Cascade Visibility Inheritance

## Overview
Voxel Visibility Propagation is a method for determining the visibility of voxels within a multi-resolution voxel cascade by inheriting occlusion data from coarser levels. It solves the problem of efficiently culling skylight or ambient occlusion contributions in high-resolution voxel grids by leveraging spatial coherence, avoiding expensive per-voxel ray marching. It differs from [Near Plane Visibility Culling](10816-near-plane-visibility-culling.md) by operating on cascade hierarchy rather than camera frustum planes, and from [Obstacle-Flag](14294-obstacle-flag.md) by using multiplicative opacity accumulation rather than binary flags.

## How it works
**Inputs:**
- `voxel_targets`: Array of voxel data, containing opacity values.
- `visibility_targets`: Array storing visibility state, packed as `unorm4x8` (4 channels, 8-bit unsigned normalized).
- `cascade_anchors`: Configuration defining cascade boundaries and sizes.
- `target_cascade`: The index of the cascade currently being processed.
- `first_entry`: Boolean flag indicating if this is the first voxel in the current cascade iteration.

**Outputs:**
- Updated `visibility_targets` array with propagated visibility values.

**Data Structures:**
- `visibility_targets`: Indexed by voxel position within the cascade. Each entry is a `unorm4x8` value. The visibility component is stored in one channel (typically Red), with other channels zeroed.
- `voxel_targets`: Indexed by voxel position. Contains opacity values.

**Algorithm:**
The method iterates through voxels in the specified `target_cascade`. For each voxel, it performs the following steps:

1. **Boundary Check:** If the voxel position `voxel_pos` is outside the valid bounds of the cascade (i.e., any coordinate $\le 0$ or $\ge 32$), skip processing this voxel.
2. **Cascade Inheritance:** If `first_entry` is true:
   - Calculate the outer voxel position in the coarser cascade: `outer_voxel_pos = voxel_pos / 2` (integer division).
   - Compute the index in the coarser cascade: `coarser_index = indexFromCascadePos(target_cascade + 1, outer_voxel_pos)`.
   - Retrieve the visibility value from `visibility_targets[coarser_index]`.
   - Set the current `visibility` to this retrieved value.
   - Set `first_entry` to false.
3. **Opacity Sampling:** Get the current voxel index: `ix_voxel = indexFromCascadePos(target_cascade, voxel_pos)`. Unpack the opacity value from `voxel_targets[ix_voxel]`.
4. **Visibility Update:** Store the current `visibility` value into `visibility_targets[ix_voxel]`, ensuring other channels are zeroed. Update the `visibility` variable for subsequent voxels by multiplying it by the remaining transmittance:
   $$
   visibility_{new} = visibility_{old} \times (1 - opacity)
   $$
   Where $opacity$ is the value unpacked from `voxel_targets`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def propagate_visibility(voxel_targets, visibility_targets, cascade_anchors, target_cascade, first_entry):
    # Iterate through voxels in the cascade
    for voxel_pos in get_voxel_positions(target_cascade):
        # Step 1: Boundary check
        if any(coord <= 0 or coord >= 32 for coord in voxel_pos):
            continue

        # Step 2: Cascade inheritance
        if first_entry:
            outer_voxel_pos = voxel_pos // 2  # Integer division
            coarser_index = index_from_cascade_pos(target_cascade + 1, outer_voxel_pos)
            visibility = visibility_targets[coarser_index]
            first_entry = False
        else:
            visibility = visibility  # Carry over from previous voxel in same cascade

        # Step 3: Opacity sampling
        ix_voxel = index_from_cascade_pos(target_cascade, voxel_pos)
        opacity = unpack_opacity(voxel_targets[ix_voxel])

        # Step 4: Visibility update
        # Store visibility with zeros in other channels
        visibility_targets[ix_voxel] = pack_unorm4x8(visibility, 0, 0, 0)
        
        # Update visibility for next voxel
        visibility = visibility * (1.0 - opacity)
```

**Parameters:**
- `cascade_size`: Typically 32 voxels per dimension per cascade level, as indicated by the boundary check.
- `unorm4x8`: 8-bit unsigned normalized integer per channel, allowing values in $[0, 1]$.

**Edge Cases:**
- **Precision:** Multiplicative accumulation of $(1 - opacity)$ can lead to underflow for deep occlusion chains.
- **Boundary Conditions:** Voxels at the edge of the cascade may not have valid coarser neighbors if the cascade hierarchy is not aligned.
- **Degenerate Input:** If `voxel_targets` contains no opacity data, visibility remains unchanged from the inherited value.

## Subtopics

### [Voxel Cascade Skylight Visibility Sweep](19430-voxel-cascade-skylight-visibility-sweep.md)
A compute-shader-based variant that propagates skylight visibility from coarser to finer cascades using Bresenham line traversal for sampling visibility targets, reducing computational cost compared to direct ray marching.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the cascade, as each voxel is processed once with constant-time operations.
- **Memory Complexity:** $O(N)$ for storing `visibility_targets` and `voxel_targets`.
- **Performance:** Efficient for large voxel grids due to cache-friendly sequential access and avoidance of ray tracing.
- **Strengths:** Leverages spatial coherence; simple to implement; low computational overhead.
- **Limitations:** Accuracy depends on cascade alignment; multiplicative opacity can suffer from precision loss; not suitable for complex light transport beyond direct visibility.
- **When to Use:** Skylight culling, ambient occlusion in voxel-based rendering.
- **When Not to Use:** Complex global illumination, dynamic scenes with frequent voxel updates requiring high precision.

## Background
Originated in the context of voxel cone tracing and voxel-based rendering pipelines, specifically in implementations like Sandalmoth's voxel-cone-tracing project. It addresses the need for efficient visibility culling in multi-resolution voxel grids.

## Sources
[S1] Sandalmoth/voxel-cone-tracing, https://github.com/Sandalmoth/voxel-cone-tracing, fetched 2026-10-26 (unofficial)
