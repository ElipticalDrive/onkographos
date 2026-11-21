# Implicit Min/Max KD-Trees
Also called: Wald et al
## Overview
Implicit Min/Max KD-Trees are a memory-efficient acceleration structure for coherent iso-surface ray tracing that stores min/max scalar values in a balanced binary kd-tree to enable early subtree culling [S1, unofficial]. The structure is built by recursively splitting the volume at cell boundaries closest to the center in the largest dimension, ensuring a one-to-one mapping between volume cells and kd-tree voxels [S1, unofficial].
## How it works
**Inputs:** A 3D volume dataset of size $N_x \times N_y \times N_z$ [S1, unofficial].
**Outputs:** An implicit kd-tree structure defined by splitting dimensions and min/max bounds, enabling ray traversal [S1, unofficial].
**Data Structures:** The tree is balanced and binary. Nodes are addressed implicitly. Split dimensions are determined per node based on voxel extent. Min/max bounds store the minimum and maximum scalar values of the voxels in each node's subtree [S1, unofficial]. Leaves store the value of a single voxel [S1, unofficial].
**Construction Steps:**
1. **Build Base KD-Tree:** Construct a kd-tree over all voxels of the entire dataset [S1, unofficial].
2. **Determine Splitting Axis:** For a node covering a range of voxels, identify the dimension with the largest extent (number of cells) [S1, unofficial].
3. **Split Plane:** Split at the cell boundary closest to the center of the node in that largest dimension [S1, unofficial].
4. **Compute Min/Max Recursively:** Each leaf node stores the min/max values of its associated cell, and each inner node stores the min/max values of its children [S1, unofficial].
**Formulas:**
- $min_{node} = \min(min_{left}, min_{right})$ [G]
- $max_{node} = \max(max_{left}, max_{right})$ [G]
where $min_{left}$ and $max_{left}$ are the bounds of the left child, and $min_{right}$ and $max_{right}$ are the bounds of the right child.
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_implicit_kd_tree(voxels, min_idx, max_idx, depth):
    """
    Recursively builds the implicit kd-tree.
    voxels: 3D array of scalar values.
    min_idx, max_idx: inclusive bounds of the current node's voxel range.
    depth: current tree depth.
    Returns: (min_val, max_val) for this node.
    """
    # Base case: leaf node (single voxel)
    if min_idx[0] == max_idx[0] and min_idx[1] == max_idx[1] and min_idx[2] == max_idx[2]:
        val = voxels[min_idx[0], min_idx[1], min_idx[2]]
        return val, val

    # Determine split dimension: largest extent
    extent_x = max_idx[0] - min_idx[0] + 1
    extent_y = max_idx[1] - min_idx[1] + 1
    extent_z = max_idx[2] - min_idx[2] + 1

    if extent_x >= extent_y and extent_x >= extent_z:
        split_axis = 0
    elif extent_y >= extent_x and extent_y >= extent_z:
        split_axis = 1
    else:
        split_axis = 2

    # Find split plane: cell boundary closest to center in split_axis
    center = (min_idx[split_axis] + max_idx[split_axis]) / 2.0
    split_val = int(round(center))

    # Split ranges
    left_max = list(max_idx)
    right_min = list(min_idx)

    left_max[split_axis] = split_val
    right_min[split_axis] = split_val + 1

    # Recurse
    min_left, max_left = build_implicit_kd_tree(voxels, min_idx, tuple(left_max), depth + 1)
    min_right, max_right = build_implicit_kd_tree(voxels, tuple(right_min), max_idx, depth + 1)

    # Compute min/max for current node from children
    node_min = min(min_left, min_right)
    node_max = max(max_left, max_right)

    return node_min, node_max
```
**Parameters:** Splitting strategy uses largest extent and center split [S1, unofficial]. Precision uses floating point for scalar values and integer indices for bounds [G]. Typical values are not given in the sources.
**Edge Cases:** Degenerate input occurs when a node has only one voxel, making it a leaf [S1, unofficial]. Boundary conditions require split planes to align with voxel boundaries to ensure correct mapping [S1, unofficial].
## Subtopics

### [Implicit KD-Tree for Iso-Surface Ray Tracing](8881-implicit-kd-tree-for-iso-surface-ray-tracing.md)
The Implicit KD-Tree is a memory-efficient acceleration structure for coherent iso-surface ray tracing that stores min/max values in a balanced binary kd-tree to enable early subtree culling, avoiding the storage of pointers and splitting plane positions. It differs from standard kd-trees by eliminating explicit node pointers and split-plane coordinates, relying instead on implicit addressing derived from a balanced tree structure and precomputed level-wise splitting dimensions. This approach significantly reduces memory overhead compared to naive implementations, which require 12 bytes per node (8 bytes for plane/pointers, 4 bytes for min/max), resulting in a 12x input data size overhead.

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Measured Performance:** Single PC (Dual-CPU): Bonsai 3.4 fps, Aneurysm 3.0 fps, ML 512^3 1.2 fps, Female 2.7 fps, Female (zoom) 2.3 fps [S1, unofficial]. 5-Node Cluster: Bonsai 16.2 fps, Aneurysm 14.6 fps, ML 512^3 6.1 fps, Female 13.6 fps, Female (zoom) 11.2 fps [S1, unofficial].
**Strengths:** Low memory footprint, fast construction, effective culling for coherent rays [S1, unofficial].
**Limitations:** Memory footprint for the LLNL dataset exceeded 2 GB RAM per client, preventing rendering [S1, unofficial].
## Background
Originated by Wald et al. [WFM* 05] [S1, unofficial]. The acceleration structure is similar to the one used by Wilhelm and van Gelder [WV92] [S1, unofficial]. It is used in interactive volume rendering frameworks, including scenes with diffuse shading and global illumination [S1, unofficial].
## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
