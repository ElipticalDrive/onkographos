# K-D Tree Subdivision

Also called: Binary Space Partitioning (BSP) Tree Construction, k-D Tree Volume Subdivision

## Overview
K-D Tree Subdivision partitions a 3D volume into subvolumes using a k-d tree structure with alternating binary subdivision of coordinate planes, enabling unambiguous front-to-back ordering for compositing [S2, S3, unofficial]. Unlike fixed viewpoint partitioning which minimizes overlap by slicing orthogonal to the view, k-D tree partitioning is view-independent and suitable for animation sequences where the viewpoint changes [S2, unofficial]. It determines the front-to-back image compositing order hierarchically by recursive traversal, visiting the 'front' child before the 'back' child [S3, unofficial].

## How it works
**Inputs:** A 3D volume grid of points (voxels) or scattered data samples [S2, S3, unofficial][S5].
**Outputs:** A k-d tree structure defining subvolumes and a traversal order for front-to-back compositing [S2, unofficial].

**Data Structures:**
The volume is represented as a grid of points or an array of samples containing position $(x, y, z)$ and function value [S2, unofficial][S5]. The tree can be stored recursively with node pointers or in a flat array [S4, unofficial][S5]. For array storage, the root is at index 1, and children of node $j$ are at $2j$ and $2j+1$ [S5].

**Steps:**
1. **Recursive Subdivision:** Start with the root node representing the bounding box of all samples [S5]. For a node at depth $i$, determine the split axis $d = i \pmod 3$ [S4, unofficial][S5]. Sort the vector $v$ of samples in the $x_d$ direction [S5]. Split $v$ into two halves, $v_1$ and $v_2$ [S5]. If $v$ is odd-sized, store a link to the midpoint and the pivot value in the current node [S5]. Recurse on $v_1$ and $v_2$ until the subvector length is 1 [S5]. Each leaf node contains exactly one sample [S5].
2. **Boundary Replication:** When a volume is evenly subdivided, ray samples lying in the cut boundary region are lost because each voxel is located at a corner of the grid [S2, S3, unofficial]. To prevent this, replicate one layer of the boundary grid at each subvolume [S2, unofficial]. For example, if splitting into Subvolume 1 and Subvolume 2 at layer $k$, Subvolume 1 includes layers $1$ to $k$, and Subvolume 2 includes layers $k$ to $n$, where layer $k$ is replicated in Subvolume 2 [S2, unofficial].
3. **Traversal for Compositing:** Determine the front-to-back compositing order by recursively traversing the k-d tree [S2, S3, unofficial]. Visit the 'front' child before the 'back' child [S2, S3, unofficial]. The definition of 'front' and 'back' depends on the view direction relative to the split plane [G]. For a split along axis $d$, if the view direction has a positive component along $d$, the child with higher coordinate values is 'front' [G].

**Formulas:**
- Tree height: $\lfloor \log_2(n + 1) \rfloor$ [S5].
- Worst-case number of nodes (when $n = 2^j$): $2n - 1$ [S5].
- Binary indexing operators: $a^0_d = a_d$, $a^j_d = (a^{j-1}_d) * 2$ (left shift), $a_j = (a_{j-1}) \text{ div } 2$ (right shift) [S5].
- Father of node $b_d$ is $b_{d1}$ (right shift) [S5]. Children are $b_{d1}$ and $(b_{d1}) \oplus 1_d$ (XOR with 1) [S5].

**Parameters:**
- Alternating axis: $x, y, z$ repeating [S4, unofficial][S5].
- Leaf size: 1 sample [S5].

**Edge Cases and Pitfalls:**
- **Boundary Loss:** If boundary replication is not performed, black strips will appear in the composited image when the view vector is parallel to the cut plane [S2, S3, unofficial].
- **View Dependency:** The 'front' and 'back' children must be determined dynamically based on the current view direction during traversal [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_kd_tree(samples, depth=0):
    if len(samples) == 1:
        return LeafNode(pos=samples[0].pos, val=samples[0].val)
    axis = depth % 3
    sorted_samples = sorted(samples, key=lambda s: s.pos[axis])
    mid = len(sorted_samples) // 2
    left_samples = sorted_samples[:mid]
    right_samples = sorted_samples[mid:]
    pivot = sorted_samples[mid].pos[axis]
    return InternalNode(axis=axis, pivot=pivot,
                        left=build_kd_tree(left_samples, depth + 1),
                        right=build_kd_tree(right_samples, depth + 1))

def traverse_front_to_back(node, view_dir):
    if node is None: return
    if node.axis is None:
        composite(node.val)
        return
    front_child = node.right if view_dir[node.axis] >= 0 else node.left
    back_child = node.left if view_dir[node.axis] >= 0 else node.right
    traverse_front_to_back(front_child, view_dir)
    traverse_front_to_back(back_child, view_dir)
```

## Subtopics

### Array-based kd-tree Construction
A data structure storing $n$ scattered data points in a 3D kd-tree to enable fast neighbor search, avoiding expensive Voronoi diagram computation for natural neighbors [S5]. It stores the tree in a single array where the root is at index 1, and children of node $j$ are at $2j$ and $2j+1$ [S5]. Binary indexing operators are used: $a^0_d = a_d$, $a^j_d = (a^{j-1}_d) * 2$ (left shift), $a_j = (a_{j-1}) \text{ div } 2$ (right shift) [S5]. The father of node $b_d$ is $b_{d1}$ (right shift). Children are $b_{d1}$ and $(b_{d1}) \oplus 1_d$ (XOR with 1) [S5]. This method is chosen when fast neighbor search is required for point-based rendering [S5].

## Operating characteristics
- **Time Complexity:** Construction is $O(n \log n)$ due to sorting at each level [S5]. Traversal is $O(N)$ where $N$ is the number of nodes visited [G].
- **Memory Complexity:** $O(n)$ for storing samples and tree structure [S5].
- **Measured Performance:** Not given in the sources.
- **Strengths:** View-independent, suitable for animation [S2, unofficial]. Hierarchical structure allows parallel processing of sibling nodes [S2, unofficial].
- **Limitations:** Requires boundary replication to avoid artifacts [S2, unofficial]. Sorting at each level can be costly for large datasets [S5].
- **When to use:** When viewpoint is unknown or changes frequently (e.g., animation) [S2, S3, unofficial]. When parallel compositing is desired [S2, unofficial].
- **When not to use:** When a fixed orthographic viewpoint is guaranteed and minimal overlap is the primary optimization goal [S2, unofficial].

## Background
The method was proposed to handle volume partitioning for distributed rendering where the viewpoint might change or be perspective [S2, S3, unofficial]. It builds on the concept of k-d trees for spatial partitioning [S2, S3, unofficial].

## Sources
[S1] A data distributed, parallel algorithm for ray-traced volume rendering, https://doi.org/10.1109/prs.1993.586080, fetched 2026-10-24
[S2] NASA Technical Reports Server (NTRS) 19940009248: A data distributed parallel algorithm for ray-traced volume rendering, https://archive.org/details/NASA_NTRS_Archive_19940009248, fetched 2026-10-26 (unofficial)
[S3] DTIC ADA270501: A Data Distributed Parallel Algorithm for Ray-Traced Volume Rendering, https://archive.org/details/DTIC_ADA270501, fetched 2026-10-26 (unofficial)
[S4] PDF Generating Signed Distance Fields from Triangle Meshes using Vulkan ..., https://williamwhitehouse.dev/assets/projects/2023/sdf-gen/gen-sdf-from-meshes-using-vulkan-compute-report.pdf, fetched 2026-10-30 (unofficial)
[S5] Direct Isosurface Extraction from Scattered Volume Data, https://doi.org/10.2312/vissym/eurovis06/099-106, fetched 2026-10-26
