# Domain-based BVH with Range Bounding Boxes

Also called: BVH acceleration, Bounding Volume Hierarchy, Range-based BVH, Range-based acceleration data-structure (BVH)

## Overview
The Domain-based BVH with Range Bounding Boxes is an acceleration structure for fiber surface extraction in tetrahedral meshes, utilizing a Bounding Volume Hierarchy (BVH) over the domain to store interval ranges (RBBs) at each node [S1]. It solves the problem of efficiently identifying tetrahedra that intersect a given FSCP (Fiber Surface Control Polygon) edge by filtering candidates via 2-D intersection tests in range space [S1]. Unlike its sibling [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md), the BVH is not strictly domain-based, allowing it to better capture the geometry of the input polygon in range-space, resulting in fewer returned tetrahedra, smaller memory requirements, and faster queries [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh defining the domain, the continuous scatter plot (CSP) function $f$, and a set of FSCP edges $e$ to query [S1].
*   **Output:** A subset of tetrahedra whose Range Bounding Boxes (RBBs) intersect the query edge $e$, used for fiber surface extraction [S1].

**Data Structures:**
The structure is a tree where each node $N$ contains:
1.  **RBB:** A 2-D bounding box in range space, defined by minimum and maximum values for the two range dimensions ($[min_x, max_x] \times [min_y, max_y]$) [S1].
2.  **Tetrahedra List:** A list of tetrahedra whose RBBs are completely included in the node's RBB [S1].
3.  **Children:** A list of child nodes (up to $n_S$ children per node) [S1].

**Construction Algorithm (Off-line):**
The BVH is constructed in a top-to-bottom fashion [S1].
1.  **Initialize Root:** Start with the root node covering the entire range space [S1].
2.  **Assign Tetrahedra:** Assign to the current node the list of tetrahedra whose RBB is completely included in the node's RBB [S1].
3.  **Update RBB:** Update the node's RBB after transferring tetrahedra from the parent node and before creating children nodes [S1]. This yields less regular but more refined range subdivision patterns compared to a quad-tree [S1].
4.  **Check Threshold:** If the number of tetrahedra assigned to the node is fewer than a threshold $n_T$, stop recursion for this branch [S1].
5.  **Split:** Perform the split $n_S$ times for the current node, recursively splitting the RBB in the middle along horizontal and vertical axes [S1]. This yields an $n_S$-ary tree [S1].
6.  **Recurse:** Create child nodes for each split region and repeat steps 2-5 for each child [S1].

**Query Algorithm (On-line):**
Given an FSCP edge $e$:
1.  **Start at Root:** Begin the query at the root node [S1].
2.  **Test Intersection:** Check if the node's RBB intersects or overlaps the edge $e$ in 2-D range space [S1].
3.  **Visit Children:** If the RBB intersects $e$, recursively visit each child node [S1].
4.  **Collect Results:** If the node is a leaf (or recursion stops), return the tetrahedra in the node's list [S1].
5.  **Filter:** Only tetrahedra returned by the BVH are used for fiber surface extraction [S1]. Note that the BVH does not encode the precise polygonal projection of tetrahedra (only RBBs), so it can return tetrahedra not intersected by the fiber surface [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def split_rbb(rbb_min, rbb_max, n_s):
    """Splits a 2D RBB into n_s child RBBs using a grid partition."""
    min_x, min_y = rbb_min
    max_x, max_y = rbb_max
    width = max_x - min_x
    height = max_y - min_y
    cols = int(round((n_s * width / height) ** 0.5))
    rows = int(round(n_s / cols))
    if cols * rows < n_s:
        cols += 1
    child_boxes = []
    for r in range(rows):
        for c in range(cols):
            if len(child_boxes) >= n_s:
                break
            cx_min = min_x + c * (width / cols)
            cy_min = min_y + r * (height / rows)
            cx_max = min_x + (c + 1) * (width / cols)
            cy_max = min_y + (r + 1) * (height / rows)
            child_boxes.append(((cx_min, cy_min), (cx_max, cy_max)))
    return child_boxes[:n_s]

def is_rbb_included(inner_min, inner_max, outer_min, outer_max):
    """Checks if inner RBB is completely inside outer RBB."""
    return (inner_min[0] >= outer_min[0] and inner_max[0] <= outer_max[0] and
            inner_min[1] >= outer_min[1] and inner_max[1] <= outer_max[1])

def rbb_intersects_edge(rbb_min, rbb_max, edge):
    """Checks if 2D RBB intersects or overlaps a 2D line segment (edge)."""
    x1, y1 = edge[0]
    x2, y2 = edge[1]
    dx, dy = x2 - x1, y2 - y1
    t_min, t_max = 0.0, 1.0
    for p, q in [(-dx, rbb_min[0]), (dx, rbb_max[0]), (-dy, rbb_min[1]), (dy, rbb_max[1])]:
        if p == 0:
            if q < 0 or q > 0: return False
        else:
            t = q / p
            if p < 0:
                if t > t_max: return False
                if t > t_min: t_min = t
            else:
                if t < t_min: return False
                if t < t_max: t_max = t
    return t_min <= t_max

class BVHNode:
    def __init__(self, rbb_min, rbb_max):
        self.rbb_min = rbb_min
        self.rbb_max = rbb_max
        self.tetrahedra = []
        self.children = []

def build_bvh(tetrahedra, rbb_min, rbb_max, n_s, n_t):
    node = BVHNode(rbb_min, rbb_max)
    for tet in tetrahedra:
        if is_rbb_included(tet.rbb_min, tet.rbb_max, rbb_min, rbb_max):
            node.tetrahedra.append(tet)
    if len(node.tetrahedra) < n_t:
        return node
    child_bounds = split_rbb(rbb_min, rbb_max, n_s)
    for child_min, child_max in child_bounds:
        child_tets = [t for t in node.tetrahedra if is_rbb_included(t.rbb_min, t.rbb_max, child_min, child_max)]
        node.children.append(build_bvh(child_tets, child_min, child_max, n_s, n_t))
    return node

def query_bvh(node, edge):
    if not rbb_intersects_edge(node.rbb_min, node.rbb_max, edge):
        return []
    results = []
    for child in node.children:
        results.extend(query_bvh(child, edge))
    if not node.children:
        return node.tetrahedra
    return results
```

**Parameters:**
*   $n_S$: Number of splits performed for each node. Typical value: 8 offered the best trade-off [S1].
*   $n_T$: Threshold for stopping recursion. Typical value: 8 [S1].

**Edge Cases and Pitfalls:**
*   The BVH does not encode the precise polygonal projection of tetrahedra (only RBBs), so it can return tetrahedra not intersected by the fiber surface [S1].
*   Precision issues in 2-D intersection tests may affect filtering accuracy [G].

## Subtopics

### LBVH Construction for Sparse Volumetric Data
Adapts a linear bounding volume hierarchy (LBVH) construction algorithm for triangle geometry to sparse volumetric data by using the boundaries of non-empty bricks as bounding geometry [S2, unofficial]. Zellmann et al. [ZSL18] divide the volume into a uniform grid with a fixed brick size for their LBVH construction algorithm [S2, unofficial]. This allows constructing an acceleration structure on the GPU at interactive frame rates for $1024^3$ volumes [S2, unofficial]. The LBVH structure allows for two to three times higher frame rates than using no empty-space skipping during ray marching [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Pre-processing time depends on $n_S$ and $n_T$. On the EthaneDiol dataset, pre-processing took 1.478s [S1].
*   **Memory Complexity:** On the EthaneDiol dataset, a BVH with $n_T=8$ used 116.728 MB memory [S1].
*   **Measured Performance:** On the EthaneDiol dataset, the BVH queried 12.654% of tetrahedra and took 0.177s for extraction [S1]. Average speedup over non-accelerated algorithm: 36x [S1].
*   **Strengths:** Fewer returned tetrahedra, smaller memory requirements, and faster queries compared to the octree [S1].
*   **Limitations:** Does not encode precise polygonal projection, leading to false positives (tetrahedra not intersected by the fiber surface) [S1].
*   **When to use:** When faster queries and lower memory usage are prioritized over the strict domain-based filtering of the octree [S1].

## Background
The Domain-based BVH with Range Bounding Boxes was introduced in "Fast and Exact Fiber Surfaces for Tetrahedral Meshes" [S1]. It is an alternative to the Domain-based Octree with Range Bounding Boxes, leveraging the efficiency of BVH structures for 2-D range space queries [S1].

## Sources
[S1] Fast and Exact Fiber Surfaces for Tetrahedral Meshes, https://hal.science/hal-01206120v2, fetched 2026-10-26
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
