# Voxel Raycast (Nonzero/Color Channel)

Also called: Hidden Surface Elimination via Ray Casting, Visible Surface Detection, zylann::voxel_raycast with Color Predicate

## Overview
This method performs hidden surface elimination by casting a ray through a binary voxel volume to locate the first voxel where a specific data channel is non-zero [S1, unofficial][S2]. It differs from standard DDA traversal by optimizing for sparse or binary data: it processes voxels in groups of eight packed into a single byte, advancing the ray by whole bytes when the group is empty, thereby skipping large empty regions efficiently [S2]. This approach is distinct from sibling methods that use adaptive edge smoothing or neighbor-averaged interpolation, as it relies on strict bitwise presence checks rather than continuous field evaluation .

## How it works
**Inputs:**
- Ray origin $\mathbf{O}$ and direction $\mathbf{D}$ (normalized).
- Voxel grid data structure containing binary or channel-based voxel values.
- Channel index $c$ to evaluate for non-zero presence.
- Maximum ray distance $d_{max}$.

**Outputs:**
- Hit position $\mathbf{P}_{hit}$.
- Previous position $\mathbf{P}_{prev}$ (position before the hit).
- Distance along ray $d_{hit}$.
- Surface normal $\mathbf{N}$.

**Data Structures:**
The voxel data is stored in a binary array where each byte holds eight voxels. The layout assumes a specific packing order (typically least-significant bit to most-significant bit or vice versa, though the exact bit-to-voxel mapping depends on the implementation's endianness and packing convention) [S2]. A predicate structure `RaycastPredicateColor` holds a reference to the `VoxelData` and the target channel index $c$ [S1, unofficial].

**Steps:**
1. **Initialize Raycast Predicate:** Create a predicate object that encapsulates the voxel data and the channel index $c$. The predicate's evaluation function `operator()` takes a ray state (containing the current hit position) and returns true if the voxel value at that position for channel $c$ is non-zero [S1, unofficial].
2. **Execute Voxel Raycast:** Call the core raycasting routine (e.g., `zylann::voxel_raycast`) with the ray origin, direction, the predicate, maximum distance, and output variables for hit/previous positions and distances [S1, unofficial].
3. **Byte-Level Traversal (Optimization):** Inside the traversal, the algorithm examines the voxel array along the ray's path. For rays aligned with main axes (X, Y, or ±Z), it reads one byte at a time. If the byte value is zero, it advances to the next byte, effectively skipping eight voxels simultaneously [S2].
4. **Hit Detection:** If the byte value is non-zero, the algorithm identifies that a surface voxel exists within that byte [S2].
5. **Voxel Extraction:** The specific surface voxel is retrieved by applying a bitmask to the non-zero byte to isolate the exact voxel corresponding to the ray's current position [S2].
6. **Result Construction:** Upon a successful hit, instantiate a `VoxelRaycastResult` object. Set the position to the hit location, the previous position to the location before the hit, the distance to the hit distance, and compute the normal as the normalized vector from the previous position to the hit position: $\mathbf{N} = \frac{\mathbf{P}_{hit} - \mathbf{P}_{prev}}{\|\mathbf{P}_{hit} - \mathbf{P}_{prev}\|}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_raycast_nonzero_color(ray_origin, ray_direction, voxel_data, channel_index, max_distance):
    """
    Casts a ray to find the first voxel where the specified channel is non-zero.
    Optimized for byte-packed voxel data (8 voxels per byte).
    """
    
    # Define the predicate function
    def predicate(ray_state):
        # Get the current position from the ray state
        pos = ray_state.hit_position
        # Retrieve the voxel value for the specific channel
        # defval is a default value if the position is out of bounds
        voxel_value = voxel_data.get_voxel(pos, channel_index, defval=0)
        # Return True if the value is non-zero
        return voxel_value != 0

    # Initialize output variables
    hit_position = None
    prev_position = None
    hit_distance = None
    
    # Call the underlying voxel raycast implementation
    # This function handles the DDA traversal and calls the predicate
    success = zylann_voxel_raycast_impl(
        ray_origin=ray_origin,
        ray_direction=ray_direction,
        predicate=predicate,
        max_distance=max_distance,
        out_hit_position=hit_position,
        out_prev_position=prev_position,
        out_hit_distance=hit_distance
    )
    
    if success:
        # Compute normal from the step taken
        normal = normalize(hit_position - prev_position)
        
        result = VoxelRaycastResult(
            position=hit_position,
            previous_position=prev_position,
            distance_along_ray=hit_distance,
            normal=normal
        )
        return result
    else:
        return None
```

**Edge Cases and Pitfalls:**
- **Byte Alignment:** The algorithm assumes the voxel array is stored such that each byte holds eight voxels, allowing 8 voxels to be examined simultaneously along main axes [S2]. If the data is not packed this way, the optimization fails or produces incorrect results.
- **Bit Masking:** The specific surface voxel must be correctly extracted from the byte using masking. Incorrect bit ordering (LSB vs MSB) will lead to hitting the wrong voxel within the byte [S2].
- **Precision:** Floating-point precision in ray position calculation may cause jitter or missed hits in sparse volumes if not handled with epsilon checks [G].

## Operating characteristics
- **Time Complexity:** $O(k)$ where $k$ is the number of empty bytes traversed before a hit. In sparse volumes, this is significantly faster than per-voxel traversal.
- **Memory Complexity:** $O(1)$ auxiliary memory, as it operates on the existing voxel data structure.
- **Strengths:** Extremely fast for binary or sparse data where large empty regions are common. Simple to implement if the data format is known.
- **Limitations:** Only works for finding the *first* non-zero voxel. Not suitable for volume rendering requiring integration through the entire ray. Requires specific data packing (8 voxels/byte).
- **When to use:** Hidden surface elimination, collision detection in binary voxel worlds, visible surface detection.
- **When not to use:** Volume rendering with transparency, continuous field evaluation, or when data is not packed in 8-voxel bytes.

## Background
The concept of hidden surface elimination via ray casting in binary voxel models was introduced in the late 1980s, with early implementations focusing on efficient depth cues for real-time rotation [S2]. The specific optimization of examining eight voxels per byte was described in "Hardware Support for the Display and Manipulation of Binary Voxel Models" (1988) [S2]. The modern implementation in `zylann::voxel_raycast` adapts this principle for real-time applications in engines like Godot, integrating it with general-purpose voxel data structures [S1, unofficial].

## Sources
[S1] Zylann/godot_voxel, https://github.com/Zylann/godot_voxel, fetched 2026-10-26 (unofficial)
[S2] Hardware Support for the Display and Manipulation of Binary Voxel Models, https://doi.org/10.2312/eggh/eggh88/103-117, fetched 2026-10-26
