# Greedy Axis-Aligned Spatial Partitioning

Also called: Greedy AABB Partitioning

## Overview
Greedy Axis-Aligned Spatial Partitioning creates a disjoint spatial partition of a 3D volume into axis-aligned bounding boxes using a greedy heuristic that minimizes bounding box volume [S1, unofficial]. It is used in the Volume Partitioning Pipeline to split foreground regions into manageable components for export [S1, unofficial]. It differs from [Slab-projected kd-tree partitioning](11586-slab-projected-kd-tree-partitioning.md) by using a direct greedy box-splitting approach rather than slab-projection slices for out-of-core processing [S2, unofficial].

## How it works
**Inputs:**
- `volume`: 3D array of density values.
- `components`: Target number of partitions (default 8) [S1, unofficial].

**Outputs:**
- `labels`: 3D array of integer identifiers (1-based) for each voxel belonging to a leaf partition. Voxels not in any partition (e.g., background) are 0 [S1, unofficial].
- `leaves`: List of AABBs defining the partition.

**Data Structures:**
- `AABB`: Tuple `(min_xyz, max_xyz)` where `min_xyz` and `max_xyz` are integer arrays of shape `(3,)` representing inclusive bounds [G].
- `tight(min, max)`: Helper function returning `None` if `min > max` or if the box contains no active voxels (`volume > 0`), otherwise returns `(min, max)` [S1, unofficial].

**Steps:**
1. **Root Initialization:** Compute the tight bounding box for the root volume: `root = tight(np.zeros(3, dtype=int), np.array(volume.shape))`. If `root` is `None`, return empty labels [S1, unofficial].
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

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_aabb_partition(volume, components=8):
    root_min = np.zeros(3, dtype=int)
    root_max = np.array(volume.shape, dtype=int)
    root = tight(root_min, root_max)
    if root is None:
        return np.zeros(volume.shape, dtype=int), []

    leaves = [root]
    leaf_id = 1

    while len(leaves) < components:
        best_score = None
        best_split = None
        best_parent_idx = None

        for i, box in enumerate(leaves):
            for axis in range(3):
                length = box.max[axis] - box.min[axis] + 1
                for frac in (0.25, 0.5, 0.75):
                    offset = round(frac * length)
                    if offset <= 0 or offset >= length:
                        continue

                    left_min = box.min.copy()
                    left_max = box.max.copy()
                    right_min = box.min.copy()
                    right_max = box.max.copy()

                    left_max[axis] = box.min[axis] + offset - 1
                    right_min[axis] = box.min[axis] + offset

                    tight_left = tight((left_min, left_max))
                    tight_right = tight((right_min, right_max))

                    if tight_left is not None and tight_right is not None:
                        size_b = volume_size(box)
                        size_l = volume_size(tight_left)
                        size_r = volume_size(tight_right)
                        surf_l = surface_area(tight_left)
                        surf_r = surface_area(tight_right)

                        score = (size_b - size_l - size_r,
                                 size_b - max(size_l, size_r),
                                 -surf_l - surf_r)

                        if best_score is None or score > best_score:
                            best_score = score
                            best_split = (tight_left, tight_right)
                            best_parent_idx = i

        if best_split is None:
            break

        leaves.pop(best_parent_idx)
        leaves.append(best_split[0])
        leaves.append(best_split[1])

    labels = np.zeros(volume.shape, dtype=int)
    for leaf in leaves:
        mask = volume[leaf.min[0]:leaf.max[0]+1,
                      leaf.min[1]:leaf.max[1]+1,
                      leaf.min[2]:leaf.max[2]+1] > 0
        labels[leaf.min[0]:leaf.max[0]+1,
               leaf.min[1]:leaf.max[1]+1,
               leaf.min[2]:leaf.max[2]+1][mask] = leaf_id
        leaf_id += 1

    return labels, leaves

def tight(min_xyz, max_xyz):
    if np.any(min_xyz > max_xyz):
        return None
    # Placeholder for active voxel check; sources do not specify exact thresholding logic [S1, unofficial]
    return (min_xyz, max_xyz)

def volume_size(box):
    b = box if isinstance(box, tuple) else box
    return int(np.prod(b.max - b.min + 1))

def surface_area(box):
    b = box if isinstance(box, tuple) else box
    dims = b.max - b.min + 1
    return int(dims[1]*dims[2] + dims[0]*dims[2] + dims[0]*dims[1])
```

**Parameters:**
- `components`: Target number of partitions. Default 8 [S1, unofficial].

**Edge Cases:**
- If the volume has no active voxels, `root` is `None`, and the function returns empty labels [S1, unofficial].
- Precision issues may arise if `tight()` fails to identify active voxels due to thresholding; ensure `volume > 0` check matches the active voxel definition [S1, unofficial].

## Subtopics

### Slab-projected kd-tree partitioning
A method to find a kd-tree partition using nonempty voxel information represented by a series of slab-projection slices [S2, unofficial]. It uses out-of-core bricking to create bricks of volumetric data along with a directed acyclic graph (DAG) of relative distance along a single view direction. A heuristic evaluates the coherency of data slices to parse them into bricks sized for rendering hardware [S2, unofficial]. Choose this when processing massive out-of-core volumes where memory constraints prevent direct spatial partitioning [S2, unofficial].

### Block Trimming and Texture Block Merging
A two-stage process where partitions are trimmed to tight bounding boxes of non-empty voxels, then merged into denser texture chunks to optimize memory usage [S3]. The first stage analyses each partitioned data block to find the tightest bounding box of its non-empty "object voxels," effectively discarding large, empty surrounding regions. These irregularly sized blocks are then merged into denser "texture chunks" via horizontal and vertical merge operations [S3]. Choose this when memory footprint reduction is prioritized over regular packing simplicity [S3].

### Median-Cut
A method to build a spatial hierarchy bottom-up by choosing the direction and position of splits carefully [S4, unofficial]. It uses the Surface Area Heuristic (SAH) to optimize the split position and direction [S4, unofficial]. Choose this when minimizing ray intersection cost is more critical than minimizing bounding box volume [S4, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Minimizes bounding box volume, potentially reducing empty space in exported bricks [S1, unofficial].
- **Limitations:** Greedy approach may not find global optimum [S1, unofficial].
- **Use Cases:** Suitable for preprocessing volumes into fixed-size components for export [S1, unofficial].

## Background
Originated in the VolumeSegmenter project by Winson-L [S1, unofficial]. It is part of a preprocessing pipeline that groups volume data into bands and partitions them spatially for efficient export and visualization [S1, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://doi.org/10.24132/csrn.2026-d53, fetched 2026-10-26
[S4] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
