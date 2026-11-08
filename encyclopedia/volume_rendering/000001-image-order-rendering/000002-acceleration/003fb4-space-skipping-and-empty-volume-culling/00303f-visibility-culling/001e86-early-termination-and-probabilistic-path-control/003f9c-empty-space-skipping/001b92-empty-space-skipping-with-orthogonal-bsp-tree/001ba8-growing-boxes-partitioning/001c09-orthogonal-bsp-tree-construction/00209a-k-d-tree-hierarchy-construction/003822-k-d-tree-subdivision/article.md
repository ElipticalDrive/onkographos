# K-D Tree Subdivision

Also called: k-D Tree Volume Subdivision

## Overview
K-D Tree Subdivision partitions a volumetric dataset into subvolumes using a k-D tree structure with alternating binary subdivision of coordinate planes, enabling unambiguous front-to-back ordering for compositing [S1, S2, unofficial]. Unlike fixed viewpoint partitioning which minimizes overlap by slicing orthogonal to the view, k-D tree partitioning is view-independent and suitable for animation sequences where the viewpoint changes [S1, unofficial].

## How it works
**Inputs:** A 3D volume grid of $N$ voxels (or grid points).
**Outputs:** A k-D tree structure defining subvolumes and a traversal order for front-to-back compositing.

**Data Structures:**
The k-D tree is a binary tree. Each node represents a subvolume.
- `Node`: Contains a split dimension ($d \in \{x, y, z\}$), a split position (index or coordinate), and pointers to `left` (front) and `right` (back) children.
- The split dimension alternates at each level: Level 0 splits $x$, Level 1 splits $y$, Level 2 splits $z$, Level 3 splits $x$, etc.

**Steps:**
1. **Partitioning:** Recursively subdivide the volume. At each node, determine the split dimension based on the tree depth. Split the current subvolume into two equal halves along the chosen coordinate plane [S1, S2, unofficial].
2. **Boundary Replication:** To prevent loss of ray samples at cut boundaries, replicate one layer of the boundary grid at each subvolume [S1, unofficial]. If a subvolume is split at index $k$, the left child includes layers up to $k$, and the right child includes layers from $k$ to $n$, with layer $k$ replicated in the right child [S1, unofficial]. This prevents black strips from appearing when the view vector is parallel to the cut plane [S2, unofficial].
3. **Traversal Order:** Determine the front-to-back compositing order by recursively traversing the k-D tree, visiting the 'front' child before the 'back' child [S1, S2, unofficial]. The definition of "front" and "back" depends on the view direction relative to the split plane, but the traversal logic remains a consistent recursive visitation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_kd_tree(volume, depth=0):
    """
    Recursively build a k-D tree for volume subdivision.
    volume: list of grid points or voxel data
    depth: current tree depth, used to determine split dimension
    """
    if len(volume) <= 1:
        return LeafNode(volume)
    
    # Determine split dimension: 0=x, 1=y, 2=z
    dim = depth % 3
    
    # Split volume equally along the chosen dimension
    mid = len(volume) // 2
    
    # Split the data
    left_data = volume[:mid]
    right_data = volume[mid:]
    
    # Boundary Replication: Replicate the boundary layer to the right child
    # to avoid missing ray samples at the cut.
    # Assuming volume is a 1D array of grid points along the split axis for simplicity
    # In 3D, this implies replicating the slice at the boundary.
    if len(right_data) > 0:
        # Replicate the last element of left_data into right_data
        boundary_point = left_data[-1]
        right_data = [boundary_point] + right_data
        
    # Recursively build children
    left_child = build_kd_tree(left_data, depth + 1)
    right_child = build_kd_tree(right_data, depth + 1)
    
    return InternalNode(dim, left_child, right_child)

def get_compositing_order(node):
    """
    Returns the list of subvolumes in front-to-back compositing order.
    """
    if node is None:
        return []
    if isinstance(node, LeafNode):
        return [node]
    
    # Visit front child first, then back child
    # Note: The source implies a generic "front" and "back" visitation.
    # In a view-independent k-D tree, the "front" child is typically the one
    # closer to the viewer. However, for parallel rendering with unknown view,
    # the traversal order is fixed by the tree structure (e.g., left then right).
    # The source states: "visiting the 'front' child before the 'back' child".
    # For implementation, this usually maps to left then right in the tree structure
    # if the tree is built such that left is 'front' relative to the split.
    
    front_order = get_compositing_order(node.left)
    back_order = get_compositing_order(node.right)
    
    return front_order + back_order
```

**Parameters:**
- Tree Depth: Determined by the number of subvolumes desired.
- Split Strategy: Equal partitioning along coordinate planes [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Loss:** If boundary layers are not replicated, black strips appear in the composited image when the view vector is parallel to the cut plane [S2, unofficial].
- **View Independence:** This method does not minimize overlap for a specific view, potentially leading to more compositing work than view-dependent slicing [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Building the tree is $O(N \log N)$ or $O(N)$ depending on implementation. Traversal is $O(N)$.
- **Memory Complexity:** $O(N)$ to store the volume and tree structure.
- **Strengths:** View-independent, suitable for animation and perspective projection, enables parallel rendering of sibling nodes [S1, S2, unofficial].
- **Limitations:** Does not minimize overlap for a fixed view, potentially higher compositing cost than view-dependent methods [S1, unofficial].

## Background
Originated in the context of data-distributed parallel ray-traced volume rendering [S1, S2, unofficial]. It was developed as an alternative to fixed viewpoint partitioning to handle changing viewpoints in animation sequences [S1, unofficial]. The method is similar to front-to-back traversals of BSP-trees and octrees [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19940009248: A data distributed parallel algorithm for ray-traced volume rendering, https://archive.org/details/NASA_NTRS_Archive_19940009248, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA270501: A Data Distributed Parallel Algorithm for Ray-Traced Volume Rendering, https://archive.org/details/DTIC_ADA270501, fetched 2026-10-26 (unofficial)
