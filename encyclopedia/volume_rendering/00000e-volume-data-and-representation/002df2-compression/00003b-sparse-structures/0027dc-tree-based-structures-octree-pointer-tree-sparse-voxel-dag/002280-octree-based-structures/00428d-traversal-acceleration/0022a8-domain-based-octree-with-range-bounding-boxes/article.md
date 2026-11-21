# Domain-based Octree with Range Bounding Boxes

Also called: Domain-Based Acceleration (Octree), Domain-based acceleration data-structure (Octree), Octree Acceleration, Range Bounding Box (RBB) Octree

## Overview
This method accelerates fiber surface extraction on tetrahedral meshes by organizing tetrahedra into an octree over the spatial domain, where each node stores a Range Bounding Box (RBB) representing the function's range over its contained cells [S1]. It solves the problem of efficiently identifying which tetrahedra intersect a given edge of the Fiber Surface Control Polygon (FSCP) by replacing exact polygon-polygon intersection tests with a conservative 2D axis-aligned bounding box (AABB) intersection test between the FSCP edge and the node's RBB [S1]. Unlike the range-based BVH variant, which partitions the range space directly, this method partitions the spatial domain, resulting in larger memory requirements and potentially more returned tetrahedra, but remains effective for interactive visualization [S1, Subtopics].

## How it works
**Inputs:**
- A tetrahedral mesh with a scalar function $f$ defined on it.
- A Fiber Surface Control Polygon (FSCP), defined as a sequence of 2D points (edges $e$).
- Parameters: $n_T$ (minimum tetrahedra per node threshold), $\alpha$ (RBB size termination fraction).

**Outputs:**
- A set of candidate tetrahedra that may intersect the FSCP edges, ready for exact fiber surface extraction.

**Data Structures:**
- **Octree Node:**
  - `children`: Array of 8 child nodes (or null if leaf).
  - `rbb_min`: 2D vector $(min_u, min_v)$ representing the minimum range values of the function over the node's domain.
  - `rbb_max`: 2D vector $(max_u, max_v)$ representing the maximum range values.
  - `tets`: List of tetrahedra indices (only present if the node is a leaf or below threshold $n_T$).

**Construction (Off-line):**
1.  Compute the octree of the domain $Dom_f$ in a top-to-bottom fashion by recursively dividing the domain into octants [S1].
2.  For each node, compute its RBB by taking the component-wise minimum and maximum of the RBBs of its children:
    $$ RBB_{node}.min = \min_{c \in children} (RBB_c.min) $$
    $$ RBB_{node}.max = \max_{c \in children} (RBB_c.max) $$
    [S1]
3.  Determine if the node should be a leaf or continue splitting:
    - Let $N_{tets}$ be the number of tetrahedra in the node's domain.
    - If $N_{tets} > n_T$, split the node into 8 children and recurse.
    - If $N_{tets} \le n_T$, compute the base level RBB from the vertices of the tetrahedra contained in this node and store the tetrahedra list. Do not split further. [S1]
4.  Termination Criterion: During recursion, if a node's RBB size (e.g., area or max dimension) is smaller than a fraction $\alpha$ of the RBB of the entire mesh, terminate recursion regardless of $n_T$. This prevents arbitrarily deep sub-trees in regions where the function range concentrates [S1].

**Query (On-line):**
1.  For each edge $e$ of the FSCP:
2.  Start at the root node.
3.  Recursively visit children only if the child's RBB intersects or overlaps the line segment $e$ in the 2D range space.
    - The intersection test is a conservative AABB intersection test between the 2D line segment $e$ and the 2D rectangle defined by $RBB_{child}$ [S1].
4.  Collect all tetrahedra from visited leaf nodes (or nodes below $n_T$) into a candidate set.
5.  Pass the candidate set to the standard fiber surface extraction algorithm, which performs exact polygon-polygon tests to discard false positives and generate the surface [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_octree(domain_cells, f, nT, alpha_global_rbb_size):
    """
    Builds the domain-based octree with range bounding boxes.
    domain_cells: List of tetrahedra in the current node's spatial domain.
    f: Scalar function evaluator.
    nT: Threshold for minimum tetrahedra per node.
    alpha_global_rbb_size: Fraction of global RBB size to stop recursion.
    """
    # Compute RBB for current set of tetrahedra
    rbb_min = [inf, inf]
    rbb_max = [-inf, -inf]
    for tet in domain_cells:
        for vertex in tet.vertices:
            val = f(vertex)
            rbb_min[0] = min(rbb_min[0], val.u)
            rbb_min[1] = min(rbb_min[1], val.v)
            rbb_max[0] = max(rbb_max[0], val.u)
            rbb_max[1] = max(rbb_max[1], val.v)
    
    current_rbb_size = (rbb_max[0] - rbb_min[0]) * (rbb_max[1] - rbb_min[1])
    # Note: alpha_global_rbb_size is compared against the global mesh RBB size,
    # which should be passed or precomputed. Here we assume a check against a global constant.
    if current_rbb_size < alpha_global_rbb_size * global_rbb_size:
        return Node(children=None, rbb_min=rbb_min, rbb_max=rbb_max, tets=domain_cells)

    if len(domain_cells) > nT:
        # Split domain into 8 octants
        octants = split_domain(domain_cells)
        children = []
        for octant_cells in octants:
            child = build_octree(octant_cells, f, nT, alpha_global_rbb_size)
            children.append(child)
        # Compute parent RBB from children
        p_min = [inf, inf]
        p_max = [-inf, -inf]
        for child in children:
            p_min[0] = min(p_min[0], child.rbb_min[0])
            p_min[1] = min(p_min[1], child.rbb_min[1])
            p_max[0] = max(p_max[0], child.rbb_max[0])
            p_max[1] = max(p_max[1], child.rbb_max[1])
        return Node(children=children, rbb_min=p_min, rbb_max=p_max, tets=[])
    else:
        return Node(children=None, rbb_min=rbb_min, rbb_max=rbb_max, tets=domain_cells)

def query_octree(node, edge_segment, global_rbb_size):
    """
    Queries the octree for tetrahedra intersecting the FSCP edge.
    node: Current octree node.
    edge_segment: 2D line segment (p1, p2) in range space.
    """
    # Conservative AABB intersection test between edge_segment and node.rbb
    if not aabb_segment_intersect(node.rbb_min, node.rbb_max, edge_segment):
        return []
    
    if node.children is None:
        # Leaf node or below threshold
        return node.tets
    
    candidates = []
    for child in node.children:
        candidates.extend(query_octree(child, edge_segment, global_rbb_size))
    return candidates

def aabb_segment_intersect(aabb_min, aabb_max, segment):
    """
    Checks if a 2D line segment intersects or overlaps an AABB.
    Conservative test: returns True if they intersect or if the segment passes through the box.
    """
    # Standard 2D line-AABB intersection test
    # Returns True if intersection exists
    pass
```

**Parameters:**
- $n_T$: Threshold for minimum number of tetrahedra per node. Default/typical value: 1 if desired, but set to 5 for 5-subdivided regular grids in experiments [S1].
- $\alpha$: Fraction of the entire mesh's RBB used as a termination criterion for octree depth. Typical value: 0.001 offered the best trade-off in experiments [S1].

**Edge Cases and Pitfalls:**
- The octree is not necessarily balanced (except for regular grids), and sub-trees can be arbitrarily deep, leading to many line-bounding-box intersection tests [S1].
- For 5-subdivided regular grids, the minimum number of tets per octree leaf is 5 (one voxel), causing the RBB of a leaf to intersect a FSCP edge even if none of its tets are actually intersected. This results in false positives, which are handled by the subsequent exact extraction step [S1].
- The conservative AABB test may return tetrahedra that do not intersect the fiber surface, but the ordinary operation of the fiber surface extraction will discard them, so no additional geometry is created [S1].

## Subtopics

### [Domain-based BVH with Range Bounding Boxes](2808-domain-based-bvh-with-range-bounding-boxes.md)
This method accelerates fiber surface extraction on tetrahedral meshes by organizing tetrahedra into a Bounding Volume Hierarchy (BVH) over the 2D range space, where each node stores a Range Bounding Box (RBB) and a list of tetrahedra whose RBBs are contained within it. It solves the problem of efficiently identifying which tetrahedra intersect a given edge of the Fiber Surface Control Polygon (FSCP) by reducing the query to a 2D intersection test between the FSCP edge and the node's RBB. Unlike the domain-based octree, which partitions the spatial domain, the BVH partitions the range space directly, resulting in fewer returned tetrahedra, smaller memory requirements, and faster queries for the same dataset.

## Operating characteristics
- **Time Complexity:** Construction is $O(N)$ where $N$ is the number of tetrahedra, assuming balanced splitting. Query time depends on the number of visited nodes, which is reduced by the RBB filtering.
- **Memory Complexity:** The octree structure stores RBBs at each node. Memory usage is significant; for the EthaneDiol dataset, the octree used 649.736 MB [S1].
- **Measured Performance:** On the EthaneDiol dataset, the octree with $\alpha=0.001$ took 2.362s for pre-processing, queried 35.260% of tetrahedra, and took 0.311s for extraction [S1]. The average speedup over the non-accelerated algorithm is 18x [S1].
- **Strengths:** Provides interactive rates for fiber surface extraction; conservative test is fast; handles complex FSCP shapes better than point-isovalue methods.
- **Limitations:** Unbalanced trees can lead to deep recursion; conservative test returns false positives (handled by downstream exact test); higher memory usage compared to range-based BVH [S1].
- **When to use:** When spatial locality is important or when the domain structure is naturally hierarchical; when memory is not the primary constraint compared to the BVH variant.
- **When not to use:** When the range space is highly non-uniform and the BVH variant offers significantly better pruning; when memory is extremely constrained.

## Background
The method was introduced in the context of fast and exact fiber surface extraction for tetrahedral meshes [S1]. It extends the concept of using octrees for isovalue interval testing (where a scalar function's range is stored at each node) to the more complex case of fiber surfaces, which are defined by polygons in the range space rather than single points [S1]. The approach leverages the rich literature on collision detection, specifically replacing expensive polygon-polygon intersection tests with conservative AABB tests in the range space [S1].

## Sources
[S1] Fast and Exact Fiber Surfaces for Tetrahedral Meshes, https://hal.science/hal-01206120v2, fetched 2026-10-26
