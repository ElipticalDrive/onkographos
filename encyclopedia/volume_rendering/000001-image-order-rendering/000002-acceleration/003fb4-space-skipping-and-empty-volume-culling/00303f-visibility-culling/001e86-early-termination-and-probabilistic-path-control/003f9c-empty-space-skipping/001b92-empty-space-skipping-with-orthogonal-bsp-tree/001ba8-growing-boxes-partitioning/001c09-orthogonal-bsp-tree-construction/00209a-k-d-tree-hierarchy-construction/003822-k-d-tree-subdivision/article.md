# K-D Tree Subdivision

Also called: k-D Tree Volume Subdivision

## Overview
K-D Tree Subdivision partitions a volumetric dataset into subvolumes using a k-D tree structure with alternating binary subdivision of coordinate planes, enabling unambiguous front-to-back ordering for compositing [S1][S2, S3, unofficial]. Unlike fixed viewpoint partitioning which minimizes overlap by slicing orthogonal to the view, k-D tree partitioning is view-independent and suitable for animation sequences where the viewpoint changes [S2, S3, unofficial].

## How it works
**Inputs:** A 3D volume grid of $N$ grid points (voxels) [S2, S3, unofficial].
**Outputs:** A k-D tree structure defining subvolumes and a hierarchical traversal order for front-to-back compositing [S1][S2, S3, unofficial].

**Data Structures:**
The k-D tree is a binary tree where each node represents a subvolume [S2, S3, unofficial]. Each node stores the following fields:
- `dim`: The split dimension ($d \in \{x, y, z\}$) [S2, S3, unofficial].
- `split_pos`: The coordinate or index along `dim` where the split occurs [G].
- `left`: Reference to the front child node [G].
- `right`: Reference to the back child node [G].
The split dimension alternates at each level: level 0 splits $x$, level 1 splits $y$, level 2 splits $z$, and so on [S2, S3, unofficial].

**Steps:**
1. Partition the volume equally along all coordinate planes using a k-D tree structure with alternating binary subdivision at each level [S2, S3, unofficial].
2. Replicate one layer of the boundary grid at each subvolume to prevent loss of ray samples at cut boundaries [S2, S3, unofficial]. If a subvolume is split at index $k$, the left child includes layers up to $k$, and the right child includes layers from $k$ to $n$, with layer $k$ replicated in the right child [S2, S3, unofficial].
3. Determine the front-to-back compositing order by recursively traversing the k-D tree, visiting the 'front' child before the 'back' child [S1][S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Node:
    def __init__(self, dim, split_pos, left, right):
        self.dim = dim
        self.split_pos = split_pos
        self.left = left
        self.right = right

def build_kd_tree(grid_points, bounds, depth=0):
    """
    Recursively build a k-D tree for volume subdivision.
    grid_points: list of 3D coordinate tuples representing the volume grid
    bounds: tuple (min_x, max_x, min_y, max_y, min_z, max_z)
    depth: current tree depth, used to determine split dimension
    """
    # Standard guard for leaf conversion [G]
    if depth >= 10 or len(grid_points) <= 1:
        return Node(dim=None, split_pos=None, left=None, right=None)

    # Determine split dimension: 0=x, 1=y, 2=z
    dim = depth % 3
    axis_idx = dim
    min_val, max_val = bounds[axis_idx * 2], bounds[axis_idx * 2 + 1]
    mid_val = (min_val + max_val) / 2.0

    # Partition grid points equally along the chosen dimension
    left_points = [p for p in grid_points if p[axis_idx] <= mid_val]
    right_points = [p for p in grid_points if p[axis_idx] > mid_val]

    # Boundary Replication: Replicate the boundary layer to the right child
    # to avoid missing ray samples at the cut.
    boundary_points = [p for p in grid_points if abs(p[axis_idx] - mid_val) < 1e-9]
    right_points.extend(boundary_points)

    # Update bounds for recursive calls
    left_bounds = list(bounds)
    right_bounds = list(bounds)
    left_bounds[axis_idx * 2 + 1] = mid_val
    right_bounds[axis_idx * 2] = mid_val

    left_child = build_kd_tree(left_points, tuple(left_bounds), depth + 1)
    right_child = build_kd_tree(right_points, tuple(right_bounds), depth + 1)

    return Node(dim, mid_val, left_child, right_child)

def get_compositing_order(node):
    """
    Returns the list of subvolumes in front-to-back compositing order.
    """
    if node is None:
        return []
    if node.left is None and node.right is None:
        return [node]
    
    # Visit front child first, then back child
    front_order = get_compositing_order(node.left)
    back_order = get_compositing_order(node.right)
    
    return front_order + back_order
```

**Parameters:**
- Split Strategy: Equal partitioning along coordinate planes [S2, S3, unofficial].
- Tree Depth: Determined by the desired number of subvolumes [G].

**Edge Cases and Pitfalls:**
- **Boundary Loss:** If boundary layers are not replicated, ray samples in the cut boundary region are lost, which causes black strips to appear in the composited image when the view vector is parallel to the cut plane [S2, S3, unofficial].
- **View Independence:** This method does not minimize overlap for a specific view, potentially increasing compositing cost compared to view-dependent slicing [S2, S3, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** View-independent, suitable for animation and perspective projection, enables parallel rendering of sibling nodes [S2, S3, unofficial].
- **Limitations:** Does not minimize overlap for a fixed view, potentially higher compositing cost than view-dependent methods [S2, S3, unofficial].

## Background
Developed for data-distributed parallel ray-traced volume rendering [S1][S2, S3, unofficial]. It serves as an alternative to fixed viewpoint partitioning to handle changing viewpoints in animation sequences [S2, S3, unofficial]. The traversal logic is similar to well-known front-to-back traversals of BSP-trees and octrees [S2, S3, unofficial].

## Sources
[S1] A data distributed, parallel algorithm for ray-traced volume rendering, https://doi.org/10.1109/prs.1993.586080, fetched 2026-10-24
[S2] NASA Technical Reports Server (NTRS) 19940009248: A data distributed parallel algorithm for ray-traced volume rendering, https://archive.org/details/NASA_NTRS_Archive_19940009248, fetched 2026-10-26 (unofficial)
[S3] DTIC ADA270501: A Data Distributed Parallel Algorithm for Ray-Traced Volume Rendering, https://archive.org/details/DTIC_ADA270501, fetched 2026-10-26 (unofficial)
