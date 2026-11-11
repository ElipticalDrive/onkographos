# Voxel Hash for Gaussian Splat Indexing

Also called: VoxelHash

## Overview
Voxel Hash is a sparse data structure that maps each 3D Gaussian Splat to the voxel cell containing its center $\mu$, enabling efficient lookup of splats within specific voxel regions for operations like carving or stacking [S1, unofficial]. It differs from sibling structures by maintaining a direct bidirectional mapping between splat indices and voxel keys, allowing for rapid identification of splats to remove or modify within a target voxel [S1, unofficial].

## How it works
**Inputs:** A list of 3D Gaussian Splats, each defined by a center position $\mu$ and an index. A `VoxelGrid` instance defining the spatial bounds and resolution.
**Outputs:** A `VoxelHash` instance containing the mapping data.

**Data Structures:**
The `VoxelHash` class contains the following fields [S1, unofficial]:
*   `grid`: `readonly VoxelGrid` – The spatial grid configuration.
*   `stats`: `readonly VoxelHashStats` – Statistics about the hash state.
    *   `splatCount`: Total number of splats indexed.
    *   `voxelCount`: Number of occupied voxel cells.
    *   `maxSplatsInAnyVoxel`: Maximum number of splats in a single voxel.
    *   `meanSplatsPerVoxel`: Average number of splats per occupied voxel.
*   `keys`: `readonly ReadonlyArray<string>` – Sorted list of occupied voxel keys (ascending).
*   `cells`: `private readonly ReadonlyMap<string, Uint32Array>` – Maps a voxel key (string) to an array of splat indices (uint32).
*   `splatToCell`: `private readonly ReadonlyMap<number, string>` – Maps a splat index (number) to its voxel key (string).

**Build Process:**
1.  Initialize `cellsBuilder` as an empty `Map<string, number[]>` and `splatToCell` as an empty `Map<number, string>` [S1, unofficial].
2.  Iterate through all splats using a callback providing the splat `index` and center `center` [S1, unofficial].
3.  For each splat, compute the voxel coordinates $\{i, j, k\}$ by calling `grid.worldToVoxel(center)` [S1, unofficial].
4.  Generate the voxel key by calling `grid.voxelKey(i, j, k)` [S1, unofficial].
5.  Append the splat `index` to the bucket `cellsBuilder[key]` using the `pushUnique` helper [S1, unofficial].
6.  Set `splatToCell.set(index, key)` [S1, unofficial].
7.  After iteration, finalize the structure:
    *   Convert `cellsBuilder` values to `Uint32Array` instances.
    *   Compute `stats` (max/mean splats per voxel, counts).
    *   Sort `keys` ascending.
    *   Construct and return the `VoxelHash` instance [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_voxel_hash(splats, grid):
    cells_builder = {}  # Map<string, List[int]>
    splat_to_cell = {}  # Map[int, string]

    for index, center in splats:
        # Compute voxel coordinates
        i, j, k = grid.world_to_voxel(center)
        
        # Generate key
        key = grid.voxel_key(i, j, k)
        
        # Append index to bucket
        if key not in cells_builder:
            cells_builder[key] = []
        cells_builder[key].append(index)
        
        # Map splat to cell
        splat_to_cell[index] = key

    # Finalize
    keys = sorted(cells_builder.keys())
    cells = {k: Uint32Array(cells_builder[k]) for k in keys}
    
    # Compute stats
    splat_count = len(splats)
    voxel_count = len(keys)
    max_splats = max(len(v) for v in cells_builder.values()) if keys else 0
    mean_splats = splat_count / voxel_count if voxel_count > 0 else 0
    
    stats = VoxelHashStats(
        splat_count=splat_count,
        voxel_count=voxel_count,
        max_splats_in_any_voxel=max_splats,
        mean_splats_per_voxel=mean_splats
    )
    
    return VoxelHash(
        grid=grid,
        stats=stats,
        keys=keys,
        cells=cells,
        splat_to_cell=splat_to_cell
    )

def push_unique(builder, key, index):
    if key not in builder:
        builder[key] = []
    builder[key].append(index)
```

**Edge Cases and Pitfalls:**
*   **Out-of-Bounds Splats:** Splats whose center lies outside the grid's AABB are still indexed under voxel keys that fail `VoxelGrid.contains`. This serves as a signal for the caller to ignore them or expand the grid [S1, unofficial].
*   **Precision:** The mapping relies on `worldToVoxel` and `voxelKey` implementations. Inconsistent coordinate system definitions between the grid and the splat data will lead to incorrect mappings [G].

## Subtopics

### Voxel Hash Chunk Map
A variant that organizes voxels into chunks to improve cache locality and memory management for very large scenes. Choose this when memory fragmentation or large-scale scene management is a concern. [Voxel Hash Chunk Map](12207-voxel-hash-chunk-map.md)

## Operating characteristics
*   **Time Complexity:** Build time is $O(N)$ where $N$ is the number of splats, assuming constant-time hash map operations. Lookup is $O(1)$ average case for both splat-to-voxel and voxel-to-splats.
*   **Memory Complexity:** $O(N + V)$ where $V$ is the number of occupied voxels. The structure stores one entry per splat and one entry per occupied voxel.
*   **Strengths:** Enables efficient "carving" operations by quickly identifying all splats within a target voxel cell [S1, unofficial].
*   **Limitations:** The indexing is based solely on the splat center $\mu$. This means splats with large covariances (large $\sigma$) that extend into neighboring cells are not indexed in those neighbors, which can lead to artifacts like "collateral darkening" or "visible wisps" if used for masking without additional fragment-level checks [S1, unofficial].

## Background
The Voxel Hash structure is used in the context of 3D Gaussian Splatting editing tools, specifically in the `splatcarve` project. It was developed to address the limitations of per-splat masking, which cannot produce clean cube-shaped holes due to the discrete nature of splat centers [S1, unofficial]. The structure supports the backend logic for fragment-level masking, where a `Data3DTexture` is used to store occupancy, but the Voxel Hash provides the necessary index mapping to manage the splat data itself [S1, unofficial].

## Sources
[S1] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
