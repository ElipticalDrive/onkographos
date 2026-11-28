# Interior Voxel Culling

Also called: Enclosed Voxel Skip, Hidden Voxel Removal, Interior Face Culling, Visible Mask Culling, Voxel Neighbor Culling

## Overview
Interior Voxel Culling removes voxels completely surrounded by other occupied voxels on all six faces to reduce instance count and draw calls without visual impact [S1, S2, unofficial]. It differs from view-dependent methods by relying solely on topological connectivity to the grid boundary, ignoring camera projection and ray casting entirely [G].

## How it works
**Inputs:** A 3D voxel grid or list of occupied coordinates, and a chunk dimension size.
**Outputs:** A subset of voxels that expose at least one face to an empty neighbor or the grid boundary [S3, unofficial].

**Data Structures:**
1.  **Occupied Set:** A `HashSet<Long>` storing packed coordinate keys for fast neighbor lookup [S1, unofficial].
2.  **Voxel Grid:** A 3D array `[CHUNK_SIZE][CHUNK_SIZE][CHUNK_SIZE]` of `Voxel` structs, each containing a single `int type` field where `-1` denotes `AIR` [S2, unofficial].
3.  **Enclosed Mask:** A boolean array matching the grid dimensions, initialized to `True` (or `ones_like(filled)`) [S1, S2, unofficial].
4.  **Output Buffer:** `m_positionsLayered`, a `std::vector<glm::vec4>` storing world coordinates and voxel type for the visible result [S2, unofficial].

**Packing Formula:**
$$ \text{key} = (x_{\text{shifted}} \ll 42) \mid (y_{\text{shifted}} \ll 21) \mid z_{\text{shifted}} $$
where $x_{\text{shifted}} = x + 1,000,000$, $y_{\text{shifted}} = y + 1,000,000$, and $z_{\text{shifted}} = z + 1,000,000$ [S1, unofficial].

**Algorithm Steps:**
1.  **Build Occupied Set:** Iterate through all input voxel coordinates $(x, y, z)$, pack each into a unique 64-bit key using the formula above, and insert into the hash set [S1, unofficial].
2.  **Initialize Enclosed Mask:** Create a boolean array `enclosed` with the same dimensions as the voxel grid, initializing all entries to `True` [S1, S2, unofficial].
3.  **Iterate Chunk:** Loop through all voxels in the chunk using three nested loops for $x, y, z$ from $0$ to $\text{CHUNK\_SIZE}-1$ [S1, S2, unofficial].
4.  **Skip Air:** Retrieve the current voxel's type; if it equals $\text{AIR}$ ($-1$), skip to the next iteration [S2, unofficial].
5.  **Check Neighbors:** Verify the 6 immediate spatial neighbors $(\pm x, \pm y, \pm z)$ [S1, S2, unofficial].
    *   *Hash Lookup Method:* Check if all 6 neighbor keys exist in the occupied set [S1, unofficial].
    *   *Array Rolling Method:* For each axis $a \in \{0, 1, 2\}$ and shift $s \in \{1, -1\}$, roll the `filled` array by $s$ along axis $a$, set the entering edge voxels (index $0$ if $s=1$, index $\text{shape}[a]-1$ if $s=-1$) to `False` to treat out-of-bounds as empty, and update `enclosed &= neighbor` [S2, unofficial].
6.  **Mark Enclosed:** If all 6 neighbors are present (or non-air), mark the voxel as fully enclosed [S1, S2, unofficial].
7.  **Filter & Output:** Return `filled & ~enclosed` [S2, unofficial]. Compute world positions by adding chunk offsets to local coordinates, append $(x+\text{offset}, y+\text{offset}, z+\text{offset}, \text{type})$ to `m_positionsLayered`, and set `m_shouldRegen` to `false` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interior_voxel_culling(grid: list[list[list[int]]], chunk_size: int = 64) -> list[tuple[int, int, int, int]]:
    """
    Removes voxels completely surrounded by other voxels on all 6 faces.
    :param grid: 3D array of voxel types (-1 for AIR).
    :param chunk_size: Dimension of the voxel chunk.
    :return: List of (x, y, z, type) for visible voxels.
    """
    if not grid:
        return []

    # Step 1: Build hash set of occupied positions
    occupied_keys = set()
    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                if grid[x][y][z] != -1:  # Skip AIR
                    xi = x + 1_000_000
                    yi = y + 1_000_000
                    zi = z + 1_000_000
                    key = (xi << 42) | (yi << 21) | zi
                    occupied_keys.add(key)

    # Step 2: Initialize enclosed mask
    enclosed = [[True for _ in range(chunk_size)] for _ in range(chunk_size)]
    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                enclosed[x][y][z] = True

    # Step 3 & 4: Iterate and check neighbors
    directions = [
        (1, 0, 0), (-1, 0, 0),
        (0, 1, 0), (0, -1, 0),
        (0, 0, 1), (0, 0, -1)
    ]

    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                if grid[x][y][z] == -1:
                    continue

                is_enclosed = True
                for dx, dy, dz in directions:
                    nx, ny, nz = x + dx, y + dy, z + dz
                    # Treat out-of-bounds as empty
                    if 0 <= nx < chunk_size and 0 <= ny < chunk_size and 0 <= nz < chunk_size:
                        nxi = nx + 1_000_000
                        nyi = ny + 1_000_000
                        nzi = nz + 1_000_000
                        nkey = (nxi << 42) | (nyi << 21) | nzi
                        if nkey not in occupied_keys:
                            is_enclosed = False
                            break
                    else:
                        is_enclosed = False
                        break

                if is_enclosed:
                    enclosed[x][y][z] = True
                else:
                    enclosed[x][y][z] = False

    # Step 5: Filter and output
    visible_voxels = []
    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                if grid[x][y][z] != -1 and not enclosed[x][y][z]:
                    visible_voxels.append((x, y, z, grid[x][y][z]))
    
    return visible_voxels
```

**Parameters:**
*   **Coordinate Offset:** $1,000,000$ [S1, unofficial].
*   **Bit Shifts:** $42$ and $21$ [S1, unofficial].
*   **Chunk Size:** Defaults to $64$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate Range:** The packing scheme assumes coordinates fit in $20$ bits each (range $\approx \pm 500,000$); exceeding this causes collisions or overflow [S1, unofficial].
*   **Boundary Handling:** Accessing neighbors outside the chunk bounds must return $\text{AIR}$ ($-1$) or be treated as empty to prevent out-of-bounds memory access and correctly treat chunk edges as empty space [S2, unofficial].
*   **Precision:** Using $64$-bit integers for keys avoids floating-point precision issues during neighbor lookups [S1, unofficial].

## Subtopics

### Visible Voxel Culling (Outer Surface Extraction)
A rendering optimization that identifies and renders only the exterior voxels of a solid block, culling all interior voxels occluded by the outer shell [S4, unofficial]. For a solid $N \times N \times N$ voxel block, it calculates the number of outer voxels by treating the block as a hollow shell [S4, unofficial]. For a $10 \times 10 \times 10$ block, it yields $486$ outer voxels ($9 \times 9 \times 6$), resulting in $2916$ quads versus $6000$ for a full block [S4, unofficial]. For a $100 \times 100 \times 100$ block, it yields $58806$ outer voxels ($99 \times 99 \times 6$), resulting in $352836$ quads, reducing polygon count to $6\%$ of the original [S4, unofficial]. It differs from naive voxel rendering by eliminating interior geometry entirely rather than relying on per-frame depth testing or occlusion culling, significantly reducing GPU upload and rasterization load [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ to build the hash set and $O(N)$ to check neighbors, where $N$ is the number of voxels [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store the hash set of occupied keys [S1, unofficial].
*   **Strengths:** Reduces draw calls and vertex count significantly by removing fully enclosed geometry [S2, unofficial].
*   **Limitations:** Only removes voxels completely surrounded on all 6 faces; does not handle view-dependent occlusion or hollow objects with internal cavities visible through openings [S1, S2, unofficial].
*   **When to Use:** Pre-processing static voxel data to reduce rendering overhead [S1, unofficial].
*   **When Not to Use:** Hollow objects or structures with internal cavities where interior voxels may be visible through openings [S1, unofficial].

## Background
Interior Voxel Culling is a standard technique in voxel-based rendering pipelines used to optimize performance by removing redundant geometry [G]. It is commonly implemented in real-time voxel renderers and game engines that utilize chunked voxel data structures [S1, S2, unofficial]. The method relies on the principle that fully enclosed voxels do not contribute to visible surface area and can be safely discarded [S1, unofficial].

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] Rydgel/Vokuselu, https://github.com/Rydgel/Vokuselu, fetched 2026-10-27 (unofficial)
[S3] HollaFoil/Voxmat, https://github.com/HollaFoil/Voxmat, fetched 2026-10-28 (unofficial)
[S4] How do people render so many voxels at once?, https://stackoverflow.com/questions/12434908/how-do-people-render-so-many-voxels-at-once, fetched 2026-10-26 (unofficial)
