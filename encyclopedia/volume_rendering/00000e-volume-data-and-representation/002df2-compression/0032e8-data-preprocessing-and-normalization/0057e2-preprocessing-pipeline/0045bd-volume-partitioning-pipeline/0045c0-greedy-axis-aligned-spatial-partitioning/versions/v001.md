# Greedy Axis-Aligned Spatial Partitioning

Also called: Greedy AABB Partitioning

## Overview
Greedy Axis-Aligned Spatial Partitioning creates a disjoint spatial partition of a 3D volume into axis-aligned bounding boxes (AABBs) using a greedy heuristic that minimizes the total bounding box volume [S1, unofficial]. It is used in the Volume Partitioning Pipeline to split foreground regions into manageable components for export [S1, unofficial]. It differs from [Slab-projected kd-tree partitioning](11586-slab-projected-kd-tree-partitioning.md) by using a direct greedy box-splitting approach rather than slab-projection slices for out-of-core processing [S2, unofficial].

## How it works
**Inputs:**
- `volume`: 3D numpy array of density values.
- `components`: Target number of partitions (default 8) [S1, unofficial].

**Outputs:**
- `labels`: 3D array of integer identifiers (1-based) for each voxel belonging to a leaf partition. Voxels not in any partition (e.g., background) are 0.
- `leaves`: List of AABBs defining the partition.

**Data Structures:**
- `AABB`: Tuple or object `(min_xyz, max_xyz)` where `min_xyz` and `max_xyz` are integer arrays of shape `(3,)` inclusive bounds.
- `tight(min, max)`: Helper function returning `None` if `min > max` or if the box contains no active voxels (volume > 0), otherwise returns `(min, max)` [S1, unofficial].

**Steps:**
1. **Root Initialization:** Compute the tight bounding box of all active voxels in the volume. Let `root = tight(np.zeros(3, dtype=int), np.array(volume.shape))`. If `root` is `None`, return empty labels [S1, unofficial].
2. **Leaf List:** Initialize `leaves = [root]` [S1, unofficial].
3. **Iterative Splitting:** While `len(leaves) < components`:
    a. **Candidate Generation:** For each box `B` in `leaves` and each axis `a` in `{0, 1, 2}`:
        i. Calculate `length = B.max[a] - B.min[a] + 1`.
        ii. Generate candidate offsets at 25%, 50%, and 75% of `length`, rounded to integers: `offsets = [round(0.25 * length), round(0.5 * length), round(0.75 * length)]` [S1, unofficial].
        iii. For each `offset` in `offsets`:
            - Split `B` into `left` and `right` along axis `a` at `offset`.
            - `left.max[a] = B.min[a] + offset - 1`, `right.min[a] = B.min[a] + offset`.
            - Compute `tight_left = tight(left.min, left.max)` and `tight_right = tight(right.min, right.max)` [S1, unofficial].
            - If both `tight_left` and `tight_right` are not `None`, calculate the split score:
              $$
              \text{score} = \left( \text{size}(B) - \text{size}(\text{left}) - \text{size}(\text{right}), \quad \text{size}(B) - \max(\text{size}(\text{left}), \text{size}(\text{right})), \quad -\text{surface}(\text{left}) - \text{surface}(\text{right}) \right)
              $$
              where $\text{size}(B) = \prod_{i=0}^2 (B.\max[i] - B.\min[i] + 1)$ and $\text{surface}(B) = \sum_{i=0}^2 \prod_{j \neq i} (B.\max[j] - B.\min[j] + 1)$ [S1, unofficial].
    b. **Selection:** Select the split with the best (maximum) score. Replace the parent box in `leaves` with the two children `tight_left` and `tight_right` [S1, unofficial].
4. **Label Assignment:** Initialize `labels` array of zeros with shape `volume.shape`. For each leaf `L` in `leaves` with identifier `gid` (starting at 1):
    - Extract the sub-volume defined by `L`.
    - Set `labels` voxels to `gid` where `volume > 0` within `L`'s bounds [S1, unofficial].

**Parameters:**
- `components`: Target number of partitions. Default 8 [S1, unofficial].

**Edge Cases:**
- If the volume has no active voxels, `root` is `None`, and the function returns empty labels [S1, unofficial].
- Precision issues may arise if `tight()` fails to identify active voxels due to thresholding; ensure `volume > 0` check matches the active voxel definition [S1, unofficial].

## Subtopics

### Slab-projected kd-tree partitioning
A method to find a kd-tree partition using nonempty voxel information represented by a series of slab-projection slices [S2, unofficial]. It uses out-of-core bricking to create bricks of volumetric data along with a directed acyclic graph (DAG) of relative distance along a single view direction. A heuristic evaluates the coherency of data slices to parse them into bricks sized for rendering hardware [S2, unofficial]. Choose this when processing massive out-of-core volumes where memory constraints prevent direct spatial partitioning [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Minimizes bounding box volume, potentially reducing empty space in exported bricks [S1, unofficial].
- **Limitations:** Greedy approach may not find global optimum; requires multiple passes over the volume to compute tight bounds [S1, unofficial].
- **Use Cases:** Suitable for preprocessing volumes into fixed-size components for export [S1, unofficial].

## Background
Originated in the VolumeSegmenter project by Winson-L [S1, unofficial]. It is part of a preprocessing pipeline that groups volume data into bands and partitions them spatially for efficient export and visualization [S1, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
