# Greedy Voxel Mesh Optimization

Also called: CEVoxelGreedyOptimizer
## Overview
Greedy Voxel Mesh Optimization is a greedy algorithm that optimizes voxel mesh generation for 2D tile grids (e.g., Minecraft-like games) by merging adjacent identical tiles into larger rectangular quads to reduce draw calls [S1, unofficial]. It differs from its siblings, [32nd Bit Avoidance Modification](3351-32nd-bit-avoidance-modification.md) and [Sparse Voxelization with Oracle and Parallel Scan](3687-sparse-voxelization-with-oracle-and-parallel-scan.md), by explicitly constructing maximal rectangular regions of uniform tile properties through iterative expansion, rather than relying on bit-packing tricks or parallel scan-based sparse structures [S1, unofficial].
## How it works
**Inputs and Outputs:**
- **Input:** A 2D grid of tiles defined by a bounding box `[mBoundMin, mBoundMax]` (inclusive) [S1, unofficial].
- **Input:** `mIsHaveTileCallback(x, y)`: A function returning `true` if a tile exists at `(x, y)` [S1, unofficial].
- **Input:** `mIsSameTileCallback(tileA.x, tileA.y, tileB.x, tileB.y)`: A function returning `true` if the tile at `(tileA.x, tileA.y)` is identical/mergeable with the tile at `(tileB.x, tileB.y)` [S1, unofficial].
- **Output:** A list of `GreedyOptimizeResult` objects, each representing a merged rectangular mesh quad [S1, unofficial].

**Data Structures:**
- `GreedyOptimizeResult`: A struct containing:
  - `x` (int): Top-left X coordinate [S1, unofficial].
  - `y` (int): Top-left Y coordinate [S1, unofficial].
  - `sizeX` (int): Width of the merged quad [S1, unofficial].
  - `sizeY` (int): Height of the merged quad [S1, unofficial].
- `mProcessAreaDic`: A dictionary mapping `Vector2Int` (coordinates) to `bool`, marking tiles that have already been processed and included in a result [S1, unofficial].
- `mResultList`: A list of `GreedyOptimizeResult` objects [S1, unofficial].
- `mBoundMin`, `mBoundMax`: `Vector2Int` defining the optimization range [S1, unofficial].

**Parameters:**
- `mBoundMin`: Minimum X and Y coordinates of the optimization range [S1, unofficial].
- `mBoundMax`: Maximum X and Y coordinates of the optimization range [S1, unofficial].
- `mIsSameTileCallback`: Callback for tile identity comparison [S1, unofficial].
- `mIsHaveTileCallback`: Callback for tile existence check [S1, unofficial].

**Algorithm Steps:**
1. Initialize `mResultList`, `mProcessAreaDic`, `mBoundMin`, `mBoundMax`, `mIsSameTileCallback`, and `mIsHaveTileCallback` from the input parameters [S1, unofficial].
2. Iterate `y` from `mBoundMin.y` to `mBoundMax.y` inclusive [S1, unofficial].
3. Iterate `x` from `mBoundMin.x` to `mBoundMax.x` inclusive [S1, unofficial].
4. Call `DoProcessTile(x, y)` for each coordinate [S1, unofficial].

**`DoProcessTile(_nowX, _nowY)` Logic:**
1. If `(x, y)` is in `mProcessAreaDic` OR `!mIsHaveTileCallback(_nowX, _nowY)`, return immediately [S1, unofficial].
2. Call `FindSameTileMaxX(_nowX, _nowY, _nowY, mBoundMax.x, out nowMaxX)` to find the maximum X extent of identical tiles in the current row starting from `_nowX` [S1, unofficial].
3. Initialize `nowMaxY` to `_nowY` [S1, unofficial].
4. Loop `checkY` from `_nowY + 1` to `mBoundMax.y` inclusive:
   a. Call `FindSameTileMaxX(_nowX, _nowY, checkY, nowMaxX, out checkMaxX)` [S1, unofficial].
   b. If `!isHaveResult` OR `checkMaxX < nowMaxX`, break the loop [S1, unofficial].
   c. If the loop continues, set `nowMaxY = checkY` [S1, unofficial].
5. Mark all tiles in the rectangle from `(_nowX, _nowY)` to `(nowMaxX, nowMaxY)` as processed in `mProcessAreaDic` [S1, unofficial].
6. Add a new `GreedyOptimizeResult` to `mResultList` with `x=_nowX`, `y=_nowY`, `sizeX=nowMaxX-_nowX+1`, `sizeY=nowMaxY-_nowY+1` [S1, unofficial].

**`FindSameTileMaxX(_startX, _refY, _staticY, _maxSearchX, out _maxX)` Logic:**
1. Initialize `isHaveResult = false` and `_maxX = _startX` [S1, unofficial].
2. Loop `nowX` from `_startX` to `_maxSearchX` inclusive:
   a. If `(nowX, _staticY)` is in `mProcessAreaDic`, break [S1, unofficial].
   b. If `!mIsHaveTileCallback(nowX, _staticY)`, break [S1, unofficial].
   c. If `!mIsSameTileCallback(_startX, _refY, nowX, _staticY)`, break [S1, unofficial].
   d. Update `_maxX = nowX` and `isHaveResult = true` [S1, unofficial].
3. Return `isHaveResult` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_voxel_mesh_optimization(bound_min, bound_max, is_have_tile, is_same_tile):
    """
    Optimizes a 2D voxel grid by merging adjacent identical tiles into rectangular quads.
    
    Args:
        bound_min (tuple): (min_x, min_y) inclusive.
        bound_max (tuple): (max_x, max_y) inclusive.
        is_have_tile (function): (x, y) -> bool, returns True if a tile exists at (x, y).
        is_same_tile (function): (x1, y1, x2, y2) -> bool, returns True if tile at (x1, y1) is mergeable with tile at (x2, y2).
        
    Returns:
        list: List of dicts with keys 'x', 'y', 'sizeX', 'sizeY'.
    """
    result_list = []
    process_area_dic = {}  # Maps (x, y) -> True
    
    min_x, min_y = bound_min
    max_bound_x, max_bound_y = bound_max
    
    def find_same_tile_max_x(start_x, ref_y, static_y, max_search_x):
        """
        Finds the maximum X coordinate in a row (static_y) starting from start_x
        where all tiles are present and identical to the tile at (start_x, ref_y).
        Stops if a tile is missing, different, or already processed.
        Returns (is_have_result, max_x_found).
        """
        is_have_result = False
        max_x_found = start_x
        
        for now_x in range(start_x, max_search_x + 1):
            if (now_x, static_y) in process_area_dic:
                break
            if not is_have_tile(now_x, static_y):
                break
            if not is_same_tile(start_x, ref_y, now_x, static_y):
                break
            max_x_found = now_x
            is_have_result = True
            
        return is_have_result, max_x_found

    def do_process_tile(now_x, now_y):
        # If already processed or no tile exists, skip
        if (now_x, now_y) in process_area_dic or not is_have_tile(now_x, now_y):
            return

        # Step 1: Find max X extent for the starting row (now_y)
        is_have_result, now_max_x = find_same_tile_max_x(now_x, now_y, now_y, max_bound_x)
        
        # Step 2: Expand vertically upwards (increasing Y)
        now_max_y = now_y
        for check_y in range(now_y + 1, max_bound_y + 1):
            is_have_result, check_max_x = find_same_tile_max_x(now_x, now_y, check_y, now_max_x)
            
            # If no result or the new row's max X is smaller than the current max X, stop expansion
            if not is_have_result or check_max_x < now_max_x:
                break
            now_max_y = check_y
            
        # Mark all tiles in the found rectangle as processed
        for writer_x in range(now_x, now_max_x + 1):
            for writer_y in range(now_y, now_max_y + 1):
                process_area_dic[(writer_x, writer_y)] = True
                
        # Add the merged quad to results
        result_list.append({
            'x': now_x,
            'y': now_y,
            'sizeX': now_max_x - now_x + 1,
            'sizeY': now_max_y - now_y + 1
        })

    # Main iteration
    for y in range(min_y, max_bound_y + 1):
        for x in range(min_x, max_bound_x + 1):
            do_process_tile(x, y)
            
    return result_list
```

**Edge Cases and Pitfalls:**
- If `FindSameTileMaxX` returns `false` for a row above the current one (`checkY > _nowY`), the vertical expansion stops immediately because it indicates either no tile exists or the tile type differs at the start of that row [S1, unofficial].
- The algorithm assumes the grid is bounded by `mBoundMin` and `mBoundMax` [S1, unofficial]. Tiles outside this range are ignored.
- The algorithm operates strictly on integer grid coordinates; floating-point positions must be discretized before processing [G].
## Subtopics

### [Disjoint Set Visible Plane Optimization](4045-disjoint-set-visible-plane-optimization.md)
Disjoint Set Visible Plane Optimization is a greedy meshing algorithm that optimizes a 2D slice (VisiblePlane) of solid voxels by merging contiguous regions of identical voxel IDs into single rectangular mesh quads using a Disjoint Set Union (DSU) data structure. It differs from its parent, Greedy Voxel Mesh Optimization, by operating on pre-culled 2D planes rather than the full 3D volume, and from Disjoint Set Mesh Optimization by focusing specifically on the plane-by-plane merging phase after occlusion culling.

### Color Quantization
Color Quantization reduces the number of unique colors in the voxel grid to facilitate greedy meshing by snapping RGB values to discrete levels [S2, unofficial]. This preprocessing step increases the likelihood that adjacent tiles are considered "identical" by the `is_same_tile` callback, thereby enabling larger merges [S2, unofficial].
- **Step:**
  1. Calculate step size: `step = 256 // levels` [S2, unofficial].
  2. Quantize color: `quantized = (color // step) * step + step // 2` [S2, unofficial].
  3. Clip and cast: `return tuple(np.clip(quantized, 0, 255).astype(int))` [S2, unofficial].
- **Parameter:** `levels`: Number of quantization levels per channel; typical values 8, 16, 32 [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The algorithm iterates through each tile in the bounding box once. For each unprocessed tile, it performs a horizontal scan and a vertical scan. In the worst case (no merges), it visits each tile a constant number of times. With merges, the number of `DoProcessTile` calls is reduced. The complexity is roughly $O(N)$ where $N$ is the number of tiles in the bounding box, assuming constant-time callbacks and dictionary lookups [G].
- **Memory Complexity:** $O(N)$ for `mProcessAreaDic` to track processed tiles and $O(M)$ for `mResultList` where $M$ is the number of merged quads ($M \le N$) [G].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Simple to implement, effective for uniform regions (e.g., Minecraft terrain), reduces draw calls significantly [G].
- **Limitations:** Only works on 2D planes (requires pre-culling or slicing for 3D volumes), greedy approach may not find globally optimal mesh (though it finds maximal rectangles locally), relies on efficient callbacks [G].
- **When to use:** When generating meshes for 2D tile-based games or pre-culled 2D slices of 3D voxel data where draw call reduction is critical [G].
- **When not to use:** When global mesh optimization is required, or when the data is not naturally grid-aligned or tile-based [G].
## Background
Greedy Voxel Mesh Optimization, also known as CEVoxelGreedyOptimizer, originates from an unofficial implementation by Tunied in 2022 [S1, unofficial]. It is designed for optimizing voxel mesh generation in Minecraft-like games [S1, unofficial]. The algorithm is part of a broader family of voxelization optimization techniques [G]. Color Quantization, as a variant/preprocessing step, is associated with the 3DGS-to-Voxel-OpenUSD project by omolism [S2, unofficial].
## Sources
[S1] Tunied/CEVoxelGreedyOptimizer, https://github.com/Tunied/CEVoxelGreedyOptimizer, fetched 2026-10-26 (unofficial)
[S2] omolism/3DGS-to-Voxel-OpenUSD, https://github.com/omolism/3DGS-to-Voxel-OpenUSD, fetched 2026-10-27 (unofficial)
