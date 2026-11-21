# Chunk Neighbor Retrieval

Also called: GetAdjacentChunks, GetAllNeighborChunks

## Overview
Chunk Neighbor Retrieval is a method for determining the spatial coordinates of neighboring chunks relative to a center chunk in a chunk-based voxel data structure. It solves the problem of identifying which adjacent chunks must be accessed to handle boundary conditions, such as rendering, physics, or meshing, without duplicating data. It differs from [Padded Chunk Data Structure](4918-padded-chunk-data-structure.md) by avoiding memory overhead for redundant boundary voxels, and from [Chunk Border Neighbor Access](2860-chunk-border-neighbor-access.md) by explicitly returning coordinate offsets rather than providing a read-only view trait.

## How it works
The method takes a center chunk coordinate and returns an array of neighbor chunk coordinates. Two primary variants exist: face-adjacent (6-neighbor) and full surrounding (26-neighbor).

**Inputs:**
- `ChunkCoord`: A 3D integer coordinate $(x, y, z)$ representing the center chunk.

**Outputs:**
- `Neighbors`: An array of 3D integer coordinates.
  - For 6-neighbor mode: size 6.
  - For 26-neighbor mode: size 26.

**Data Structures:**
- `ChunkCoord`: A struct or tuple containing three integer fields: `x`, `y`, `z`.

**Steps:**

### 6-Neighbor Retrieval
1. Initialize an empty array `Neighbors` of size 6.
2. Add the coordinate $(x+1, y, z)$ to `Neighbors`.
3. Add the coordinate $(x-1, y, z)$ to `Neighbors`.
4. Add the coordinate $(x, y+1, z)$ to `Neighbors`.
5. Add the coordinate $(x, y-1, z)$ to `Neighbors`.
6. Add the coordinate $(x, y, z+1)$ to `Neighbors`.
7. Add the coordinate $(x, y, z-1)$ to `Neighbors`.

### 26-Neighbor Retrieval
1. Initialize an empty array `Neighbors` of size 26.
2. Loop $X$ from $-1$ to $1$.
3. Loop $Y$ from $-1$ to $1$.
4. Loop $Z$ from $-1$ to $1$.
5. If $X=0$, $Y=0$, and $Z=0$, continue to the next iteration (skip the center chunk).
6. Add the coordinate $(x+X, y+Y, z+Z)$ to `Neighbors`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_face_neighbors(chunk_coord):
    """
    Retrieves the 6 face-adjacent neighbor chunk coordinates.
    :param chunk_coord: Tuple (x, y, z)
    :return: List of tuples
    """
    x, y, z = chunk_coord
    neighbors = []
    
    # Add face-adjacent neighbors
    neighbors.append((x + 1, y, z))
    neighbors.append((x - 1, y, z))
    neighbors.append((x, y + 1, z))
    neighbors.append((x, y - 1, z))
    neighbors.append((x, y, z + 1))
    neighbors.append((x, y, z - 1))
    
    return neighbors

def get_all_neighbors(chunk_coord):
    """
    Retrieves all 26 surrounding neighbor chunk coordinates.
    :param chunk_coord: Tuple (x, y, z)
    :return: List of tuples
    """
    x, y, z = chunk_coord
    neighbors = []
    
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                # Skip the center chunk itself
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                neighbors.append((x + dx, y + dy, z + dz))
                
    return neighbors
```

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Out-of-Bounds:** The method returns coordinates that may not exist in the world (e.g., negative chunk indices if the world starts at 0). The caller must handle these, often by checking against a world boundary or treating them as "void" or "unloaded" [S2, unofficial].
- **Degenerate Input:** If `chunk_coord` is invalid (e.g., NaN), behavior is undefined. [G] Implementers should validate input coordinates before calling.

## Subtopics

### World Function / Neighbor Query
A variant that abstracts neighbor access through a global "world function" rather than returning raw coordinates. It is used in voxel engines to simplify chunk updates and boundary checks without padding overhead [S2, unofficial].
- **Definition:** Uses a global world function to fetch voxel data by coordinates, abstracting chunk boundaries and handling out-of-bounds or unloaded regions during mesh generation [S2, unofficial].
- **Steps:**
  1. Define a world function that accepts block coordinates [S2, unofficial].
  2. Compute chunk coordinates from block coordinates [S2, unofficial].
  3. Check the target chunk for the requested block [S2, unofficial].
  4. Return block ID, 0 for air, or -2 for out-of-bounds/unloaded [S2, unofficial].
  5. During mesh generation, call the world function instead of checking local chunk bounds [S2, unofficial].
- **Parameters:**
  - Return value for air: 0 [S2, unofficial].
  - Return value for out-of-bounds/unloaded: -2 [S2, unofficial].
- **Pitfalls:** Requires adding conditional checks to the meshing logic, but avoids duplicating voxel data across chunk boundaries [S2, unofficial].
- **Usage:** Used in a voxel engine to simplify chunk updates and boundary checks without padding overhead [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ for both 6-neighbor and 26-neighbor retrieval, as the number of neighbors is constant.
- **Memory Complexity:** $O(1)$ auxiliary space for the output array.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with the number of neighbors requested, but since the count is fixed, it is effectively constant time.
- **Strengths:** Simple to implement; no memory overhead for padding; flexible (can return 6 or 26 neighbors).
- **Limitations:** Does not handle world boundaries automatically; caller must manage out-of-bounds checks.
- **When to use:** When memory efficiency is critical and the caller can handle boundary logic.
- **When not to use:** When a read-only view trait is preferred for semantic uniformity of internal vs. boundary voxels ([Chunk Border Neighbor Access](2860-chunk-border-neighbor-access.md)).

## Background
The method originates from community discussions and implementations in voxel game development, particularly in the context of efficient chunk-based world generation and meshing. It is documented in the `draquel/VoxelWorlds` repository [S1, unofficial] and discussed in the `r/VoxelGameDev` subreddit [S2, unofficial].

## Sources
[S1] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S2] Tips for culling inner faces? : r/VoxelGameDev, https://www.reddit.com/r/VoxelGameDev/comments/210u58/tips_for_culling_inner_faces/, fetched 2026-10-27 (unofficial)
