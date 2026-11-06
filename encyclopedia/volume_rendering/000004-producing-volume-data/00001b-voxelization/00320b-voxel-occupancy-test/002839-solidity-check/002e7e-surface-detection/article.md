# Surface Detection


## Overview
Surface Detection determines if a voxel coordinate lies on the boundary of a solid volume by checking its 6-connected neighborhood for non-solid neighbors [S1, unofficial]. It distinguishes itself from its parent, Solidity Check, by inverting the logic: while the parent confirms if a voxel is solid, this method confirms if it is exposed to air [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A pointer to a `libvxl_map` structure, and integer coordinates $(x, y, z)$.
- Output: A boolean value (`true`/`1` if on surface, `false`/`0` otherwise).

**Data Structures:**
The method operates on `struct libvxl_map` [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
struct libvxl_map {
    size_t width, height, depth;
    struct libvxl_chunk* chunks;
    size_t* geometry;
    size_t streamed;
};
```
The `geometry` array is used to store solid/air status, accessed via the parent method `libvxl_map_issolid` [S1, unofficial].

**Algorithm Steps:**
1. **Null Check:** If the `map` pointer is `NULL`, return `false` [S1, unofficial].
2. **Neighbor Iteration:** Check the 6 immediate neighbors in the cardinal directions: $+y, -y, +x, -x, +z, -z$ [S1, unofficial].
3. **Solidity Query:** For each neighbor coordinate $(x', y', z')$, call `libvxl_map_issolid(map, x', y', z')` [S1, unofficial].
4. **Surface Determination:** If any neighbor returns `false` (is not solid), the current voxel is on the surface. Return `true` immediately [S1, unofficial].
5. **Completion:** If all 6 neighbors are solid, return `false` [S1, unofficial].

**Formula:**
The condition for a voxel at $(x,y,z)$ being on the surface is defined as [S1, unofficial]:
$$
\text{onsurface}(x,y,z) = \neg \text{solid}(x,y+1,z) \lor \neg \text{solid}(x,y-1,z) \lor \neg \text{solid}(x+1,y,z) \lor \neg \text{solid}(x-1,y,z) \lor \neg \text{solid}(x,y,z+1) \lor \neg \text{solid}(x,y,z-1)
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def libvxl_map_onsurface(map, x, y, z):
    # [G] Standard guard for null pointer check
    if map is None:
        return False
    
    # Define 6-connected neighborhood offsets: +y, -y, +x, -x, +z, -z
    offsets = [
        (0, 1, 0), (0, -1, 0),
        (1, 0, 0), (-1, 0, 0),
        (0, 0, 1), (0, 0, -1)
    ]
    
    for dx, dy, dz in offsets:
        nx, ny, nz = x + dx, y + dy, z + dz
        # Call parent method to check if neighbor is solid
        # [G] Implementation of libvxl_map_issolid is assumed to handle bounds/geometry lookup
        if not libvxl_map_issolid(map, nx, ny, nz):
            return True
            
    return False
```

**Parameters:**
- `map`: The volume data structure.
- `x, y, z`: Integer coordinates of the voxel to test.

**Edge Cases and Pitfalls:**
- **Null Map:** The method explicitly handles a null map pointer by returning `false` [S1, unofficial].
- **Boundary Conditions:** The behavior for neighbors outside the map bounds depends on the implementation of `libvxl_map_issolid` (typically treated as non-solid or handled via wrapping, as per the parent article) [S1, unofficial].

## Subtopics

### Voxel Air Setting
Sets a voxel to air, updates the geometry bit, and restores surface status to previously covered neighbors [S1, unofficial].
- **Definition:** A modification operation that converts a solid voxel to air and updates the surface flags of adjacent voxels that were previously internal [S1, unofficial].
- **Steps:**
    1. Check bounds [S1, unofficial].
    2. Record `surface_prev[6]` for each neighbor: `true` if the neighbor is not solid, or if it is solid and on surface [S1, unofficial].
    3. Call `libvxl_map_setair_internal` and set the geometry bit to 0 [S1, unofficial].
    4. For each neighbor, if `!surface_prev[i]` and the neighbor is now on surface, call `libvxl_map_set_internal` with `DEFAULT_COLOR` [S1, unofficial].
- **Pitfall:** `surface_prev` defaults to `true` if the neighbor is not solid, preventing false restoration of air blocks [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$, as it performs a fixed number of 6 neighbor checks.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Fast boundary detection for voxel-based rendering or collision.
- **Limitations:** Dependent on the correctness and performance of the underlying `libvxl_map_issolid` implementation.

## Background
Originated in the `libvxl` library by xtreme8000 [S1, unofficial]. It is part of the voxel map manipulation suite, specifically following the Solidity Check step in the voxel processing pipeline [S1, unofficial].

## Sources
[S1] libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
