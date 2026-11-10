# HDDA Iterator Reorganization
Also called: HDDAActiveValueIterator, HDDAIterators, HDDALeafVoxelIterator

## Overview
A reorganization of Hierarchical Data-Driven Approach (HDDA) iterators that separates the traversal of all active values from the traversal of only leaf voxels, enabling tile-skipping for leaf-specific operations [S1, unofficial]. It sets itself apart from its parent [VolumeHDDA](2353-volumehdda.md) by providing a specialized leaf-only traversal path that avoids processing active coarse tiles, addressing the latent mis-indexing bugs that occurred when callers assumed the previous single iterator yielded only leaf voxels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A hierarchical grid structure containing active values at various node levels (coarse tiles and leaf voxels).
- Output: A sequence of coordinates and values. `HDDAActiveValueIterator` yields all active values (tiles and leaves); `HDDALeafVoxelIterator` yields only active leaf voxels.

**Data Structures:**
Both iterators are aliases over a single implementation class:
`HDDAValueIteratorImpl<AccT, ScalarT, bool LeafOnly>`
- `AccT`: Accessor type.
- `ScalarT`: Scalar value type.
- `LeafOnly`: A compile-time boolean gate. When `true`, the iterator skips active tiles. When `false`, it yields all active values [S1, unofficial].

**Numbered Steps:**
1. **Initialization:** Create an instance of `HDDAValueIteratorImpl` with `LeafOnly` set to `false` for `HDDAActiveValueIterator` or `true` for `HDDALeafVoxelIterator` [S1, unofficial].
2. **Traversal Loop:** Iterate using the `++` operator.
3. **Level Re-alignment (`nextVoxel`):**
   - Use a bounded loop `for (pass < 3)` to handle level re-alignment [S1, unofficial].
   - This replaces three unrolled `getDim`/level-update passes.
   - In the common level-aligned case, this collapses to a single descent [S1, unofficial].
4. **Segment Advancement (`nextSegment`):**
   - Use predicated `TimeSpan` setters instead of an `if/else` ladder [S1, unofficial].
   - Only the "active region ended" condition remains a real branch, reducing warp divergence [S1, unofficial].
5. **Yielding:**
   - If `LeafOnly` is `false`: Yield the current active value (whether tile or leaf) [S1, unofficial].
   - If `LeafOnly` is `true`: Check if the current node is a leaf (`getDim == 1`). If it is a coarse tile (`getDim > 1`), skip it in a single coarse HDDA step [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class HDDAValueIteratorImpl:
    def __init__(self, grid, leaf_only: bool):
        self.grid = grid
        self.leaf_only = leaf_only
        self.current_node = grid.begin_active_value()
        self.pass_count = 0

    def __iter__(self):
        return self

    def __next__(self):
        while self.current_node:
            # Level re-alignment loop
            self.pass_count = 0
            while self.pass_count < 3:
                if self.is_level_aligned():
                    break
                self.realign_level()
                self.pass_count += 1
            
            # Check if we should skip this node
            current_dim = self.current_node.getDim()
            
            if self.leaf_only and current_dim > 1:
                # Skip active coarse tiles
                self.advance_to_next_active_value()
                continue
            
            # Yield the value
            yield self.current_node.getCoord(), self.current_node.getValue()
            
            # Advance to next segment/voxel
            self.advance_to_next_active_value()

    def is_level_aligned(self):
        # Implementation detail: checks if current node level matches expected traversal level
        return True # Placeholder for internal state check

    def realign_level(self):
        # Implementation detail: moves up or down the tree to align levels
        pass

    def advance_to_next_active_value(self):
        # Implementation detail: moves to the next active value in the grid
        self.current_node = self.current_node.next()
```

**Parameters:**
- `LeafOnly`: Compile-time boolean. `true` for leaf-only traversal, `false` for all active values [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Mis-indexing on tile-bearing grids:** Previously, callers indexing per-leaf-voxel buffers (like `gridScalars`) walked tiles they never needed, causing latent mis-indexing. The leaf iterator fixes this by skipping tiles entirely [S1, unofficial].
- **Warp Divergence:** The `nextSegment` optimization reduces divergence by using predicated setters, but the "active region ended" break remains a branch [S1, unofficial].

## Subtopics

### OpenVDB Tree vs Value Iteration
OpenVDB provides two distinct iterators: one for traversing tree topology nodes (internal and leaf nodes) and one for traversing all value-carrying elements (active/inactive voxels and tiles) [S2, unofficial]. The node iterator (`cbeginNode`) traverses the tree structure, while the value iterator (`cbeginValueAll`) traverses all values, including implicitly allocated tile nodes [S2, unofficial]. Using `cbeginValueAll` for visualization can lead to dense point clouds that misrepresent the actual sparse data density because it includes inactive values and tiles [S2, unofficial]. The `HDDAActiveValueIterator` and `HDDALeafVoxelIterator` are specialized for active values only, aligning more closely with the value iterator's intent but optimized for ray-marching [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of active values (or leaf voxels if `LeafOnly` is true). The bounded re-alignment loop ensures constant overhead per step [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory per iterator instance [S1, unofficial].
- **Performance:** The leaf-only iterator provides a significant performance lever for ray-marching kernels like `ray_implicit_intersection` by skipping active coarse tiles, which are not needed for leaf-specific operations [S1, unofficial].
- **Strengths:** Eliminates latent mis-indexing bugs; reduces warp divergence in `nextSegment`; simplifies caller code by separating tile and leaf traversal [S1, unofficial].
- **Limitations:** Requires compile-time specialization (`LeafOnly`); not suitable for operations that need to process coarse tiles [S1, unofficial].

## Background
The reorganization was inspired by Efty's 'branchless getValue' work in NanoVDB and aimed to apply similar principles to the ray-marching kernels in OpenVDB [S1, unofficial]. It was introduced in the `ray_implicit_intersection` improvements PR [S1, unofficial]. The change addresses issues where the previous `HDDAVoxelIterator` yielded active values at any node level, forcing callers to branch on `getDim()` to distinguish between tiles and leaves, which led to inefficiencies and bugs [S1, unofficial].

## Sources
[S1] `ray_implicit_intersection` improvements, https://github.com/openvdb/fvdb-core/pull/663, fetched 2026-10-25 (unofficial)
[S2] Question/doubt about VDB allocation, https://github.com/fzi-forschungszentrum-informatik/vdb_mapping/issues/2, fetched 2026-10-27 (unofficial)
