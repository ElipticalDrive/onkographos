# Brick-of-Bricks Decomposition
Also called: Flat hierarchy of bricks

## Overview
Brick-of-Bricks Decomposition is a storage format where a volume is stored as a flat hierarchy of bricks of size $n^3$ to improve cache-coherence in volume traversal [S1, unofficial][S2, book]. It arises from not fully expanding an octree into single-voxel leaf nodes [S1, unofficial][S2, book]. It sets itself apart from fully expanded octrees by grouping voxels into contiguous blocks for raycasting very large volumes [S1, unofficial][S2, book].

## How it works
**Inputs:** An octree structure where nodes are not necessarily expanded to single-voxel leaf nodes [S1, unofficial][S2, book].
**Outputs:** A flat array of bricks of size $n^3$, along with an indexing scheme to map hierarchical positions to memory addresses for the 8-neighborhood required for trilinear interpolation [S1, unofficial][S2, book].

**Data Structures:**
1. **Brick Array:** A 1D array storing voxel data grouped into contiguous blocks of $n^3$ [S1, unofficial][S2, book].
2. **Hierarchy Index:** A structure mapping the octree topology to the flat brick array, enabling fast lookup of the 8-neighborhood for trilinear interpolation [S1, unofficial][S2, book].

**Steps:**
1. **Construct Hierarchy:** Build an octree where non-leaf nodes may store aggregate metrics (e.g., average, standard deviation, minimum-maximum range, Lipschitz range, or error measures like root mean square) [S1, unofficial][S2, book].
2. **Flatten to Bricks:** Traverse the octree and group nodes into bricks of size $n^3$ to form a flat hierarchy [S1, unofficial][S2, book].
3. **Indexing Scheme:** Implement an efficient indexing scheme to quickly find the memory address of voxels in the 8-neighborhood required for trilinear interpolation [S1, unofficial][S2, book].
4. **Raycasting:** During traversal, use the indexing scheme to access voxel data for shading and interpolation [S1, unofficial][S2, book].

**Parameters:**
* $n$: The side length of a brick in voxels. Typical values are not given in the sources [S1, unofficial][S2, book].

**Edge Cases and Pitfalls:**
* **Boundary Handling:** The indexing scheme must correctly handle bricks at the boundaries of the volume or irregular octree structures [S1, unofficial][S2, book].
* **Interpolation:** Trilinear interpolation requires access to 8 neighboring voxels. If a neighbor crosses a brick boundary, the indexing scheme must correctly locate the adjacent brick and the relative offset within it [S1, unofficial][S2, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel(brick_array, brick_offsets, brick_id, x, y, z, n):
    """Access a voxel at local coordinates (x, y, z) in brick_id."""
    # Calculate linear offset within the brick (row-major order)
    offset = x + n * y + n * n * z
    # Retrieve base address for the brick from the hierarchy index
    base = brick_offsets[brick_id]
    return brick_array[base + offset]

def get_neighbor(brick_array, brick_offsets, brick_id, x, y, z, n, dx, dy, dz):
    """Access a neighbor voxel, handling brick boundaries."""
    nx, ny, nz = x + dx, y + dy, z + dz
    # Check if neighbor stays within current brick
    if 0 <= nx < n and 0 <= ny < n and 0 <= nz < n:
        return get_voxel(brick_array, brick_offsets, brick_id, nx, ny, nz, n)
    # Sources do not specify the exact algorithm for determining the adjacent brick ID when crossing boundaries [S1, unofficial][S2, book]
    # Standard practice computes global coordinates, divides by n to find the neighbor brick ID, then wraps local coordinates modulo n [G]
    return None
```

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Performance:** Improves cache-coherence in volume traversal compared to fully expanded octrees [S1, unofficial][S2, book].
**Strengths:** Better cache performance for large volumes due to flat storage [S1, unofficial][S2, book].
**Limitations:** Requires an efficient indexing scheme for neighbor access [S1, unofficial][S2, book].

## Background
Brick-of-Bricks Decomposition was introduced by Parker et al. for raycasting very large volumes [S1, unofficial][S2, book]. It arises from the observation that octrees do not have to be balanced [S1, unofficial][S2, book] nor fully expanded into single-voxel leaf nodes [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
