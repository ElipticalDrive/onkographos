# IndexGridBuilder::build
## Overview
`IndexGridBuilder::build` populates a flat output data structure with voxel values from multiple OpenVDB grids (SDF, float, vector) and generates global coordinates for each active voxel, using parallel leaf traversal [S1, unofficial].
## How it works
**Inputs:**
- A collection of OpenVDB grids: SDF grids, float grids, and vector grids [S1, unofficial].
- A domain grid tree structure that defines the topology and leaf layout [S1, unofficial].
- `m_leafOffsets`: A mapping from leaf index to base offset in the flat output array [S1, unofficial].
- `m_numLeaves`: Total number of leaves in the domain grid [S1, unofficial].

**Outputs:**
- `m_outData`: A flat data structure containing:
  - `pValues<T>(name)`: Returns a pointer to the value buffer for grid type `T` [S1, unofficial].
  - `pCoords()`: Returns a pointer to a `Coord` array storing global coordinates [S1, unofficial].
  - `allocateCoords(totalVoxels)`: Allocates memory for global coordinates [S1, unofficial].

**Data Structures:**
- `LeafManager`: Manages iteration over leaves of the domain grid tree [S1, unofficial].
- `TreeType::LeafNodeType::SIZE`: Constant integer representing the number of voxels per leaf node [S1, unofficial].
- `m_leafOffsets[i]`: Integer offset for leaf `i` in the flat output buffer [S1, unofficial].

**Steps:**
1. Create a `LeafManager` for the domain grid tree [S1, unofficial].
2. For each SDF grid: get the float tree, create a `LeafManager`, obtain the output pointer via `m_outData->pValues<float>(name)` [S1, unofficial]. Launch a parallel loop over leaf indices $i \in [0, m\_numLeaves)$ [S1, unofficial]. For each leaf $i$: get the leaf, compute `leafBaseOffset = m_leafOffsets[i]`, get `leafPos = leaf.origin()` [S1, unofficial]. If the grid leaf exists at `leafPos` via `probeConstLeaf`, copy `TreeType::LeafNodeType::SIZE` floats to `outPtr + leafBaseOffset` [S1, unofficial]. Else, fill with `1.0f` [S1, unofficial].
3. Repeat step 2 for float grids, filling missing values with `0.0f` [S1, unofficial].
4. Repeat step 2 for vector grids, copying `Vec3f` data and filling missing values with `0.0f` [S1, unofficial].
5. Launch a parallel loop over leaf indices again [S1, unofficial]. For each leaf $i$ and local index $j$: compute global coordinate $c = \text{leaf.offsetToGlobalCoord}(j)$, compute `outIdx = leafBaseOffset + j`, store $c$ in `m_outData->pCoords()[outIdx]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def IndexGridBuilder_build(domain_tree, sdf_grids, float_grids, vector_grids, m_leafOffsets, m_numLeaves, m_outData):
    # Step 1: Create LeafManager for the domain grid tree
    domain_leaf_manager = LeafManager(domain_tree)

    # Helper to process a single grid type
    def process_grid_type(grids, fill_value, is_vector):
        for grid in grids:
            tree = grid.tree
            leaf_manager = LeafManager(tree)
            out_ptr = m_outData.pValues(grid.name, float if not is_vector else Vec3f)
            # Parallelized over leaf indices
            for i in range(m_numLeaves):
                leaf = leaf_manager.get_leaf(i)
                leaf_base_offset = m_leafOffsets[i]
                leaf_pos = leaf.origin()
                source_leaf = tree.probe_const_leaf(leaf_pos)
                if source_leaf is not None:
                    if is_vector:
                        for j in range(3):
                            out_ptr[leaf_base_offset + j] = source_leaf.get_voxel(j)
                    else:
                        for j in range(LeafNodeType.SIZE):
                            out_ptr[leaf_base_offset + j] = source_leaf.get_voxel(j)
                else:
                    if is_vector:
                        for j in range(3):
                            out_ptr[leaf_base_offset + j] = fill_value
                    else:
                        for j in range(LeafNodeType.SIZE):
                            out_ptr[leaf_base_offset + j] = fill_value

    # Step 2: Process SDF grids (fill missing with 1.0f)
    process_grid_type(sdf_grids, 1.0, False)
    # Step 3: Process float grids (fill missing with 0.0f)
    process_grid_type(float_grids, 0.0, False)
    # Step 4: Process vector grids (fill missing with 0.0f)
    process_grid_type(vector_grids, 0.0, True)

    # Step 5: Compute global coordinates
    for i in range(m_numLeaves):
        leaf = domain_leaf_manager.get_leaf(i)
        leaf_base_offset = m_leafOffsets[i]
        for j in range(LeafNodeType.SIZE):
            global_coord = leaf.offset_to_global_coord(j)
            out_idx = leaf_base_offset + j
            m_outData.pCoords()[out_idx] = global_coord
```

**Parameters:**
- `m_leafOffsets`: Array mapping leaf index to flat buffer offset. Size: `m_numLeaves` [S1, unofficial].
- `m_numLeaves`: Total number of leaves [S1, unofficial].
- `TreeType::LeafNodeType::SIZE`: Constant leaf voxel count [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Topology Mismatch:** The domain grid leaf structure must match all added grids' leaf structures for correct offset mapping; mismatched topology causes incorrect data placement [S1, unofficial].
- **Missing Leaves:** If a leaf exists in the domain grid but not in a source grid, it is filled with a default value (`1.0f` for SDF, `0.0f` for float/vector) [S1, unofficial].
- **Precision:** Global coordinates are stored as `Coord` (typically integer) [S1, unofficial].

## Subtopics

### IndexGridBuilder::writeIndexGrid
Creates an OpenVDB grid from the flat output buffer by cloning the domain grid's topology and copying voxel data back into leaf nodes [S1, unofficial]. It differs from `build` by reconstructing a hierarchical OpenVDB grid structure from the flat buffer, rather than producing a flat array [S1, unofficial]. Choose this when the output needs to be saved as or further processed as an OpenVDB grid [S1, unofficial]. **Parameters:** `voxelSize` (float, linear transform scale factor for the output grid) [S1, unofficial]. **Pitfalls:** Topology cloning copies inactive voxels too; pruning/deactivation is commented out but available if needed [S1, unofficial]. **Steps:**
1. Create new grid, set name and linear transform with `voxelSize` [S1, unofficial].
2. Set grid class to `GRID_FOG_VOLUME` for `FloatGrid`, or `GRID_STAGGERED` with `VEC_CONTRAVARIANT_RELATIVE` for `VectorGrid` [S1, unofficial].
3. Assert `m_totalVoxels` equals `m_outData->size()` to ensure consistency [S1, unofficial].
4. Call `leafManager.foreach` with lambda: for each leaf and `leafIdx`, get `leafBaseOffset` from `m_leafOffsets[leafIdx]`, get `leafData` pointer, copy `LeafT::SIZE` values from `data + leafBaseOffset` to `leafData` [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. [S1, unofficial].
- **Memory Complexity:** Not given in the sources. [S1, unofficial].
- **Measured Performance:** Not given in the sources. [S1, unofficial].
- **Scaling:** Not given in the sources. [S1, unofficial].
- **Strengths:** Not given in the sources. [S1, unofficial].
- **Limitations:** Not given in the sources. [S1, unofficial].
- **When to use:** Not given in the sources. [S1, unofficial].
- **When not to use:** Not given in the sources. [S1, unofficial].

## Background
Originated in: Not given in the sources. [S1, unofficial].
Used in: Not given in the sources. [S1, unofficial].

## Sources
[S1] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
