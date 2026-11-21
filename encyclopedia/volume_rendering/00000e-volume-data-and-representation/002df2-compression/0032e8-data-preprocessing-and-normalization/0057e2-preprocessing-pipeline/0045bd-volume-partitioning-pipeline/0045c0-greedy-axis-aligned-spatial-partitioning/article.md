# Greedy Axis-Aligned Spatial Partitioning

Also called: Greedy Bounding Box Volume Minimization

## Overview
Greedy Axis-Aligned Spatial Partitioning creates a disjoint spatial partition of a 3D volume into axis-aligned bounding boxes (AABBs) by recursively splitting regions to minimize the total bounding box volume of the resulting leaves [S1, unofficial]. It differs from [K-Means Spatial-Intensity Partitioning](11588-k-means-spatial-intensity-partitioning.md) by relying purely on geometric sparsity rather than intensity clustering, and from [Slab-projected kd-tree partitioning](18019-slab-projected-kd-tree-partitioning.md) by using a greedy, local optimization strategy rather than global projection-based heuristics.

## How it works
**Inputs:**
- `volume`: 3D numpy array or memory-mapped array of voxel data.
- `components`: Target number of leaf partitions (default: 8) [S1, unofficial].
- `tight()`: Helper function that returns the tightest AABB containing all non-zero voxels within a given region, or `None` if the region contains no non-zero voxels.

**Outputs:**
- `leaves`: List of AABBs (tuples of `(min_xyz, max_xyz)`).
- `labels`: Mapping of leaf index to voxel mask.

**Data Structures:**
- `leaves`: List of AABB objects. Each AABB is defined by `min` and `max` integer coordinates (inclusive or exclusive, consistent with `tight()`).
- `scores`: Tuple `(volume_reduction, max_child_volume_reduction, surface_area_penalty)` used for sorting.

**Algorithm Steps:**
1. **Initialize Root:** Compute the tight bounding box for the entire volume. If the volume is empty, return empty labels.
   $$ \text{root} = \text{tight}(\mathbf{0}, \text{volume.shape}) $$
   Initialize `leaves = [root]`.

2. **Iterative Splitting:** While `len(leaves) < components`:
   a. Iterate over each box $B$ in `leaves`.
   b. For each axis $a \in \{0, 1, 2\}$:
      i. Calculate the length $L_a$ of box $B$ along axis $a$.
      ii. Define candidate split offsets at 25%, 50%, and 75% of $L_a$, rounded to integers.
      iii. For each offset $o$:
           - Split $B$ into left child $B_L$ and right child $B_R$ at $o$.
           - Compute $T_L = \text{tight}(B_L)$ and $T_R = \text{tight}(B_R)$.
           - If both $T_L$ and $T_R$ are valid (not `None`):
             - Calculate the score $S$ for this split:
               $$ S = \left( \text{size}(B) - \text{size}(T_L) - \text{size}(T_R), \quad \text{size}(B) - \max(\text{size}(T_L), \text{size}(T_R)), \quad -\text{surface}(T_L) - \text{surface}(T_R) \right) $$
               Where $\text{size}(B)$ is the volume (number of voxels) of the bounding box, and $\text{surface}(B)$ is the surface area.
             - Store $(S, B, B_L, B_R, \text{axis}, \text{offset})$ as a candidate.

   c. Select the candidate with the best score (highest volume reduction, then highest max-child reduction, then lowest surface area).
   d. Replace parent $B$ in `leaves` with $B_L$ and $B_R$.

3. **Label Assignment:**
   For each leaf $L_i$ in `leaves` (identifier $i$ starting at 1):
   - Create a mask where `volume > 0` within the bounds of $L_i$.
   - Assign label $i$ to these voxels.

**Parameters:**
- `components`: Integer, target number of partitions. Default 8 [S1, unofficial].

**Edge Cases:**
- Empty volume: Returns empty labels immediately [S1, unofficial].
- Degenerate splits: If a split results in one child being empty (tight returns `None`), that split is invalid and ignored [S1, unofficial].
- Precision: Offsets are rounded to integers to ensure voxel alignment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def tight(box_min, box_max, volume):
    """
    Returns the tightest AABB containing all non-zero voxels within box_min <= v < box_max.
    Returns None if no non-zero voxels exist in the region.
    """
    # Extract sub-volume
    # Note: Implementation details of slicing depend on volume structure.
    # Assuming standard numpy slicing for this example.
    sub_vol = volume[box_min[0]:box_max[0], box_min[1]:box_max[1], box_min[2]:box_max[2]]
    
    # Find non-zero indices
    nz = np.nonzero(sub_vol)
    if not nz[0].size:
        return None
    
    # Compute tight bounds relative to box_min
    min_z, max_z = nz[0].min(), nz[0].max()
    min_y, max_y = nz[1].min(), nz[1].max()
    min_x, max_x = nz[2].min(), nz[2].max()
    
    # Adjust for absolute coordinates
    tight_min = np.array([box_min[0] + min_z, box_min[1] + min_y, box_min[2] + min_x])
    tight_max = np.array([box_min[0] + max_z + 1, box_min[1] + max_y + 1, box_min[2] + max_x + 1])
    
    return (tight_min, tight_max)

def box_size(box):
    """Returns volume of the bounding box."""
    return np.prod(box[1] - box[0])

def box_surface(box):
    """Returns surface area of the bounding box."""
    dims = box[1] - box[0]
    return 2 * (dims[0]*dims[1] + dims[1]*dims[2] + dims[2]*dims[0])

def greedy_partition(volume, components=8):
    if volume.size == 0 or np.all(volume == 0):
        return []
    
    root_min = np.zeros(3, dtype=int)
    root_max = np.array(volume.shape, dtype=int)
    root = (root_min, root_max)
    
    # Check if root is valid
    if tight(root_min, root_max, volume) is None:
        return []
        
    leaves = [root]
    
    while len(leaves) < components:
        best_score = None
        best_split = None
        
        for i, box in enumerate(leaves):
            box_min, box_max = box
            dims = box_max - box_min
            
            for axis in range(3):
                length = dims[axis]
                if length < 2:
                    continue
                
                # Try offsets at 25%, 50%, 75%
                offsets = [
                    int(length * 0.25),
                    int(length * 0.5),
                    int(length * 0.75)
                ]
                
                for offset in offsets:
                    if offset <= 0 or offset >= length:
                        continue
                    
                    # Calculate split position
                    split_pos = box_min[axis] + offset
                    
                    # Define children
                    left_min = box_min.copy()
                    left_max = box_max.copy()
                    left_max[axis] = split_pos
                    
                    right_min = box_min.copy()
                    right_max = box_max.copy()
                    right_min[axis] = split_pos
                    
                    # Compute tight boxes
                    t_left = tight(left_min, left_max, volume)
                    t_right = tight(right_min, right_max, volume)
                    
                    if t_left is None or t_right is None:
                        continue
                    
                    # Calculate score
                    s_vol = box_size(box) - box_size(t_left) - box_size(t_right)
                    s_max = box_size(box) - max(box_size(t_left), box_size(t_right))
                    s_surf = -box_surface(t_left) - box_surface(t_right)
                    
                    score = (s_vol, s_max, s_surf)
                    
                    if best_score is None or score > best_score:
                        best_score = score
                        best_split = (i, t_left, t_right)
        
        if best_split is None:
            break # Cannot split further
            
        idx, left, right = best_split
        leaves[idx] = left
        leaves.append(right)
        
    return leaves
```

## Subtopics

### [Division Heuristics Refinements](18019-division-heuristics-refinements.md)
Division Heuristics Refinements are minor algorithmic optimizations applied to the greedy axis-aligned spatial partitioning process to improve load-balancing accuracy and computational efficiency during volume data subdivision. These refinements address approximation errors inherent in linear division schemes by introducing binary search for cut location, recursive load recalculation, and multi-array workload tracking. Unlike the primary greedy heuristic which minimizes bounding box volume, these refinements focus on minimizing load imbalance errors introduced during the partitioning phase, though they have a lesser impact on overall runtime compared to earlier rasterization-stage refinements [18019-division-heuristics-refinements.md].

### Slab-projected kd-tree partitioning
A method to find a kd-tree partition using nonempty voxel information represented by a series of slab-projection slices. It uses out-of-core bricking to create bricks of volumetric data along with a directed acyclic graph (DAG) of the relative distance of each brick along a single view direction. A heuristic evaluates the coherency of a series of data slices, which are parsed into a set of bricks sized for rendering hardware [S2, unofficial].

### Block Trimming and Texture Block Merging
A two-stage process where partitions are trimmed to tight bounding boxes of non-empty voxels, then merged into denser texture chunks to optimize memory usage. The first stage discards empty surrounding regions. The second stage merges irregularly sized blocks into denser "texture chunks" via horizontal and vertical merge operations [S3].

### Median-Cut
A method to build a spatial hierarchy bottom-up by choosing the direction and position of splits carefully. It uses the Surface Area Heuristic (SAH) to optimize the split position and direction, based on the principle that the number of rays hitting an object is proportional to its projected area [S4, unofficial].

### Multiscale Spatiotemporal Partitioning
A unified 4D space-time partitioning strategy that produces hierarchical bounding boxes, enabling the handling of large time-varying datasets provided memory can hold effective blocks at the finest scale. It uses Euclidean distance to measure similarity between blocks. A caveat is that enforcing the same number of scales for spatial and temporal domains may not work ideally for datasets with small spatial and large temporal dimensions; manually separating temporal intervals may hinder temporal coherence utilization [S5].

## Operating characteristics
- **Time Complexity:** $O(N \cdot C \cdot A \cdot K)$, where $N$ is the number of voxels, $C$ is the target components, $A$ is the number of axes (3), and $K$ is the number of candidate offsets per axis (typically 3). The `tight()` function scans the volume, making it expensive if not optimized with precomputed histograms or sparse structures.
- **Memory Complexity:** $O(N)$ for the volume data plus $O(C)$ for the leaf list.
- **Strengths:** Simple to implement; effectively removes empty space; produces compact bounding boxes.
- **Limitations:** Greedy nature may lead to suboptimal global load balancing; sensitive to the order of splits; performance degrades with high sparsity if `tight()` is not accelerated.
- **When to use:** When memory efficiency is critical and the volume has significant empty space; when a quick, heuristic-based partition is sufficient.
- **When not to use:** When precise load balancing across heterogeneous hardware is required (use [K-Means](11588-k-means-spatial-intensity-partitioning.md) or [Division Heuristics Refinements](18019-division-heuristics-refinements.md)); when temporal coherence is paramount (use [Multiscale Spatiotemporal Partitioning](18019-multiscale-spatiotemporal-partitioning.md)).

## Background
The method originates from the VolumeSegmenter project, which implements a greedy heuristic to minimize bounding box volume for disjoint spatial partitioning [S1, unofficial]. It is part of a broader pipeline for volume data preprocessing, often following automatic crop box generation and preceding export or compression stages. The approach is rooted in early volume visualization techniques that prioritized spatial coherence and memory reduction over intensity-based clustering [S2, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] A Hybrid CPU-GPU Approach for Memory-Efficient Volume Rendering of Medical Image Data, https://doi.org/10.24132/csrn.2026-d53, fetched 2026-10-26
[S4] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
[S5] ECNR: Efficient Compressive Neural Representation of Time-Varying Volumetric Datasets, https://doi.org/10.1109/pacificvis60374.2024.00017, fetched 2026-10-26
