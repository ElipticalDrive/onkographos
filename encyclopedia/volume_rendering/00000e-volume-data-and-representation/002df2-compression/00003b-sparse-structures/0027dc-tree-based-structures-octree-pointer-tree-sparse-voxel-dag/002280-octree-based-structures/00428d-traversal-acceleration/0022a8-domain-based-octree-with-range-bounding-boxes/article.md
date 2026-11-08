# Domain-based Octree with Range Bounding Boxes

Also called: Domain-Based Acceleration (Octree), Domain-based acceleration data-structure (Octree), Octree Acceleration, Range Bounding Box (RBB) Octree

## Overview
The Domain-based Octree with Range Bounding Boxes is an acceleration structure for fiber surface extraction in tetrahedral meshes, utilizing an octree over the spatial domain to store interval ranges (RBBs) at each node [S1]. It solves the problem of efficiently identifying tetrahedra that intersect a given FSCP (Fiber Surface Control Polygon) edge by filtering candidates via 2-D intersection tests in range space [S1]. Unlike its sibling [Domain-based BVH with Range Bounding Boxes](2808-domain-based-bvh-with-range-bounding-boxes.md), the octree is strictly domain-based, which may result in more returned tetrahedra and larger memory requirements [S1].

## How it works
**Inputs:**
- A tetrahedral mesh with a scalar function $f$ defined on it [S1].
- An FSCP edge $e$ (a line segment in 3D space) [S1].
- Construction parameters: $n_T$ (minimum tetrahedra per node threshold) and $\alpha$ (RBB size fraction termination threshold) [S1].

**Outputs:**
- A set of candidate tetrahedra that potentially intersect the FSCP edge $e$ [S1].

**Data Structures:**
- **Octree Node:**
  - `children`: Array of up to 8 pointers to child nodes (null if leaf or empty) [S1].
  - `rbb_min`: 3D vector $(min_x, min_y, min_z)$ representing the minimum bounds of the function range in the node's volume [S1].
  - `rbb_max`: 3D vector $(max_x, max_y, max_z)$ representing the maximum bounds of the function range in the node's volume [S1].
  - `tets`: List of tetrahedron indices (populated only at leaf nodes or nodes below the $n_T$ threshold) [S1].

**Algorithm Steps:**
1. Off-line construction: Compute the octree of the domain in a top-to-bottom fashion by recursively dividing the domain into octants [S1].
2. At each node, take the RBBs of the child nodes and compute the component-wise $(min, max)$ to find the RBB of the entire node [S1].
3. Stop descending to individual tetrahedra if the number of tetrahedra per node is above a threshold $n_T$; compute the base level RBB from the vertices of the tetrahedra if below $n_T$ [S1].
4. Terminate recursion if a node's RBB is smaller than a fraction $\alpha$ of the RBB of the entire mesh [S1].
5. On-line query: Start at the root node. Recursively visit children only if the child's RBB intersects or overlaps the FSCP edge $e$ [S1].
6. Replace simple point-in-interval tests with polygon-polygon intersection/inclusion tests between the FSCP and the image of the cell, approximated by a conservative test of axis-aligned bounding boxes (AABB) of the polygons in the range, accepting false positives [S1].

**Formulas:**
- RBB aggregation: $RBB_{node}.min = \min_i(RBB_{child_i}.min)$ and $RBB_{node}.max = \max_i(RBB_{child_i}.max)$ for all children $i$ [S1].
- Termination check: $size(RBB_{node}) < \alpha \times size(RBB_{mesh})$ [S1].

**Parameters:**
- $n_T$: Threshold for minimum number of tetrahedra per node. Default/typical value: 1 if desired, but set to 5 for 5-subdivided regular grids in experiments [S1].
- $\alpha$: Fraction of the entire mesh's RBB used as a termination criterion for octree depth. Typical value: 0.001 offered the best trade-off in experiments [S1].

**Edge Cases and Pitfalls:**
- The octree is not necessarily balanced and sub-trees can be arbitrarily deep, leading to many line-bounding-box intersection tests [S1].
- For 5-subdivided regular grids, the minimum number of tets per octree leaf is 5 (one voxel), causing the RBB of a leaf to intersect a FSCP edge even if none of its tets are actually intersected [S1].
- The conservative AABB test in range space may return tetrahedra that do not actually intersect the FSCP edge, but these are discarded by the subsequent fiber surface extraction algorithm [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_rbb_from_tets(tetrahedra, eval_f):
    """Compute the bounding box of scalar values for a list of tetrahedra."""
    min_val = float('inf')
    max_val = float('-inf')
    for tet in tetrahedra:
        for vertex in tet.vertices:
            val = eval_f(vertex)
            if val < min_val: min_val = val
            if val > max_val: max_val = val
    return (min_val, max_val)

def split_octant(bounds):
    """Divide a spatial bounding box into 8 sub-octants."""
    (min_x, min_y, min_z), (max_x, max_y, max_z) = bounds
    mid_x, mid_y, mid_z = (min_x + max_x) / 2, (min_y + max_y) / 2, (min_z + max_z) / 2
    sub_bounds = []
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                sub_bounds.append(((min_x + dx*(mid_x-min_x), min_y + dy*(mid_y-min_y), min_z + dz*(mid_z-min_z)),
                                   (mid_x + dx*(max_x-mid_x), mid_y + dy*(max_y-mid_y), mid_z + dz*(max_z-mid_z))))
    return sub_bounds

def tet_intersects_box(tet, box):
    """Check if a tetrahedron intersects a spatial bounding box."""
    # Standard axis-aligned bounding box intersection test for tetrahedra
    tet_min = min(v for vertex in tet.vertices for v in vertex)
    tet_max = max(v for vertex in tet.vertices for v in vertex)
    box_min, box_max = box
    return not (tet_max < box_min or tet_min > box_max)

def build_octree(spatial_bounds, tetrahedra, nT, alpha, global_rbb_size, eval_f):
    if not tetrahedra:
        return None

    # Compute RBB from vertices if below nT, otherwise aggregate from children
    if len(tetrahedra) < nT:
        rbb_min, rbb_max = compute_rbb_from_tets(tetrahedra, eval_f)
        rbb_size = max(rbb_max[i] - rbb_min[i] for i in range(3))
        if rbb_size < alpha * global_rbb_size:
            return {'rbb_min': rbb_min, 'rbb_max': rbb_max, 'children': [], 'tets': tetrahedra}

    # Subdivide into 8 octants
    sub_bounds = split_octant(spatial_bounds)
    children = []
    for bounds in sub_bounds:
        sub_tets = [tet for tet in tetrahedra if tet_intersects_box(tet, bounds)]
        child = build_octree(bounds, sub_tets, nT, alpha, global_rbb_size, eval_f)
        if child is not None:
            children.append(child)

    if children:
        # Aggregate RBB from children
        rbb_min = min(c['rbb_min'][i] for c in children for i in range(3))
        rbb_max = max(c['rbb_max'][i] for c in children for i in range(3))
        return {'rbb_min': rbb_min, 'rbb_max': rbb_max, 'children': children, 'tets': []}
    else:
        # No children, compute RBB from current tets
        rbb_min, rbb_max = compute_rbb_from_tets(tetrahedra, eval_f)
        return {'rbb_min': rbb_min, 'rbb_max': rbb_max, 'children': [], 'tets': tetrahedra}

def query_octree(node, edge_p1, edge_p2, eval_f):
    if node is None:
        return []

    # Project edge endpoints to range space
    p1_range = eval_f(edge_p1)
    p2_range = eval_f(edge_p2)
    edge_min = [min(p1_range[i], p2_range[i]) for i in range(3)]
    edge_max = [max(p1_range[i], p2_range[i]) for i in range(3)]

    # Conservative AABB intersection test in range space
    intersects = all(node['rbb_min'][i] <= edge_max[i] and node['rbb_max'][i] >= edge_min[i] for i in range(3))
    if not intersects:
        return []

    candidates = []
    if not node['children']:
        candidates.extend(node['tets'])
    else:
        for child in node['children']:
            candidates.extend(query_octree(child, edge_p1, edge_p2, eval_f))
    return candidates
```

## Subtopics

### [Domain-based BVH with Range Bounding Boxes](2808-domain-based-bvh-with-range-bounding-boxes.md)
The Domain-based BVH with Range Bounding Boxes is an acceleration structure for fiber surface extraction in tetrahedral meshes, utilizing a Bounding Volume Hierarchy (BVH) over the domain to store interval ranges (RBBs) at each node. It solves the problem of efficiently identifying tetrahedra that intersect a given FSCP (Fiber Surface Control Polygon) edge by filtering candidates via 2-D intersection tests in range space. Unlike its sibling [Domain-based Octree with Range Bounding Boxes](2806-domain-based-octree-with-range-bounding-boxes.md), the BVH is not strictly domain-based, allowing it to better capture the geometry of the input polygon in range-space, resulting in fewer returned tetrahedra, smaller memory requirements, and faster queries.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On the EthaneDiol dataset, an octree with $\alpha=0.001$ used 649.736 MB memory, took 2.362s pre-processing, queried 35.260% of tetrahedra, and took 0.311s for extraction [S1]. Average speedup over non-accelerated algorithm: 18x [S1].
- **Strengths:** Effective culling of large empty regions; conservative test allows fast rejection [S1].
- **Limitations:** Unbalanced trees can lead to deep recursion; conservative test returns false positives; less efficient than BVH in terms of memory and query speed for complex polygons [S1].
- **When to use:** When a simple, domain-based spatial hierarchy is preferred over a range-based BVH [S1].
- **When not to use:** When memory is constrained or when the FSCP geometry is highly complex and requires precise culling (BVH is better) [S1].

## Background
The Domain-based Octree with Range Bounding Boxes was introduced in the context of fast and exact fiber surface extraction for tetrahedral meshes [S1]. It extends the traditional use of octrees for storing scalar function intervals to handle polygonal FSCP edges by using range-space bounding boxes [S1].

## Sources
[S1] Fast and Exact Fiber Surfaces for Tetrahedral Meshes, https://hal.science/hal-01206120v2, fetched 2026-10-26
