# Solidity Check

Also called: Void Check with Boundary Handling

## Overview
Solidity Check determines whether a specific voxel coordinate in a 3D world is occupied by solid geometry or is empty space (void). It serves as the final step in voxel occupancy testing, resolving whether a point lies within a solid volume. It differs from general occupancy tests by explicitly handling boundary conditions, coordinate wrapping, and chunk-based data structures to prevent rendering artifacts at world edges.

## How it works
**Inputs:**
- `x, y, z`: Integer world voxel coordinates.
- `map`: A reference to the voxel map structure containing dimensions and chunk data.

**Outputs:**
- `bool`: `True` if the voxel is solid (occupied), `False` if it is void (empty).

**Data Structures:**
The implementation relies on a chunked map structure.
- `libvxl_map`: Contains `width`, `height`, `depth` (dimensions), `chunks` (array of chunk pointers), `geometry` (lookup table), and `streamed` (streaming state).
- `libvxl_chunk`: Contains `blocks` (pointer to block data), `length`, and `index`.
- `libvxl_block`: Contains `position` and `color` (uint32_t).
- `libvxl_span`: Packed struct with `length`, `color_start`, `color_end`, `air_start` (uint8_t each).
- Coordinate key packing (libvxl): `pos_key(x,y,z) = ((y)<<20) | ((x)<<8) | (z)`. This assumes 12 bits for X, 12 bits for Y, and 8 bits for Z.

**Algorithm Steps:**
1. **Boundary Check (Z-axis):**
   - If `z < 0`, return `False` (void).
   - If `map` is null or `z >= map->depth`, return `True` (solid). This treats out-of-bounds Z as solid to prevent holes.

2. **Chunk Index Calculation:**
   - Calculate the chunk index for the world position using a helper function `get_chunk_index(x, y, z)`.
   - If `chunk_index == -1` (out of bounds in X/Y), return `False` (void). *Note: Source [S1, unofficial] treats out-of-bounds chunks as non-void (solid) in one context, but [S2, unofficial] implies boundary handling. We follow [S2, unofficial] for the primary logic where `z<0` is void, but [S1, unofficial] explicitly states "Out-of-bounds chunks are treated as non-void (solid)". However, [S1, unofficial] also says "If chunk_index is -1 ... return False (not void)". This is a contradiction in [S1, unofficial]. We prioritize [S2, unofficial] for the boundary logic consistency: Z bounds are solid, X/Y bounds are handled by chunk lookup. If chunk lookup fails, it is typically void or solid depending on implementation. [S1, unofficial] says return False (void). [S2, unofficial] does not explicitly state the X/Y out-of-bounds return value in the snippet, but implies wrapping or bounds checking. We will follow [S1, unofficial]'s explicit step: return False.*

3. **Coordinate Wrapping and Local Index:**
   - For valid chunks, calculate the local voxel index within the chunk.
   - X and Y coordinates wrap around the map dimensions using modulo: `x_wrapped = x % map->width`, `y_wrapped = y % map->height`.
   - Z coordinate does **not** wrap.
   - Calculate local index: `local_index = (x_wrapped % CHUNK_SIZE) + (z % CHUNK_SIZE) * CHUNK_SIZE + (y_wrapped % CHUNK_SIZE) * (CHUNK_SIZE * CHUNK_SIZE)`.
   - *Note: [S2, unofficial] uses `key_discardz` and bit shifts for lookup in some contexts, but the step description in [S1, unofficial] uses modulo arithmetic for local index. We follow [S1, unofficial]'s step description for the local index calculation.*

4. **Occupancy Test:**
   - Access the voxel data at `local_index` within the chunk's voxel array (or block structure).
   - If `chunk_voxels[local_index]` is non-zero, return `True` (solid).
   - Else, return `False` (void).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solidity_check(x, y, z, map):
    # Step 1: Z-axis boundary check
    if z < 0:
        return False  # Void
    if map is None or z >= map.depth:
        return True   # Solid (boundary)

    # Step 2: Chunk index calculation
    chunk_index = get_chunk_index(x, y, z)
    
    # Step 3: Out-of-bounds chunk check
    if chunk_index == -1:
        return False  # Void (based on [S1, unofficial] step 3)

    # Step 4: Coordinate wrapping
    # X and Y wrap around map dimensions
    x_wrapped = x % map.width
    y_wrapped = y % map.height
    # Z does not wrap, but is already validated against depth

    # Step 5: Local voxel index calculation
    # Assuming CHUNK_SIZE is a constant or defined in map
    CHUNK_SIZE = 16  # Typical value, not given in sources. [G]
    CHUNK_AREA = CHUNK_SIZE * CHUNK_SIZE
    
    local_x = x_wrapped % CHUNK_SIZE
    local_y = y_wrapped % CHUNK_SIZE
    local_z = z % CHUNK_SIZE
    
    voxel_index = local_x + (local_z * CHUNK_SIZE) + (local_y * CHUNK_AREA)
    
    # Step 6: Access chunk data
    chunk = map.chunks[chunk_index]
    
    # Access voxel data. The structure of chunk_voxels depends on the implementation.
    # [S1, unofficial] implies a direct array access. [S2, unofficial] uses blocks and spans.
    # We assume a simplified voxel array for the check as per [S1, unofficial]'s step.
    # If using [S2, unofficial]'s block structure, we would need to decode the span.
    # For this implementation, we follow [S1, unofficial]'s direct check.
    
    if chunk.voxels[voxel_index] != 0:
        return True   # Solid
    else:
        return False  # Void
```

**Parameters:**
- `CHUNK_SIZE`: Not given in the sources. [G] Typical value is 16.

**Edge Cases:**
- `z < 0`: Always void.
- `z >= depth`: Always solid.
- `chunk_index == -1`: Void (based on [S1, unofficial]).
- Coordinate wrapping: X and Y wrap, Z does not.

## Subtopics

### [Surface Detection](5803-surface-detection.md)
Surface detection determines whether a voxel or particle lies on the boundary of a solid or fluid volume. In voxel-based representations, it identifies voxels that have at least one non-solid neighbor in the 6-connected neighborhood. In particle-based fluid simulations, it identifies boundary particles by estimating occlusion on a spherical screen cover, distinguishing interior particles (shadowed by neighbors) from boundary particles (with unshadowed screen areas). It differs from general solidity checks by specifically evaluating the *neighborhood* state rather than the local occupancy, serving as a prerequisite for surface tension, boundary rendering, or collision detection.

## Operating characteristics
- **Time Complexity:** O(1) for chunk lookup and local index calculation, assuming constant-time array access.
- **Memory Complexity:** O(1) additional memory.
- **Strengths:** Efficient boundary handling prevents rendering holes. Chunk-based structure allows for sparse world representation.
- **Limitations:** Coordinate wrapping for X/Y may cause unexpected behavior at world edges if not intended. Z-axis is not wrapped, requiring explicit depth management.
- **When to use:** When determining if a specific voxel is occupied for rendering or collision.
- **When not to use:** When surface properties or neighborhood information is required (use Surface Detection).

## Background
The method originates from voxel-based rendering systems, specifically those used in Minecraft-like games and the libvxl library. It is used to efficiently determine voxel occupancy in large, chunked 3D worlds.

## Sources
[S1] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S2] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
