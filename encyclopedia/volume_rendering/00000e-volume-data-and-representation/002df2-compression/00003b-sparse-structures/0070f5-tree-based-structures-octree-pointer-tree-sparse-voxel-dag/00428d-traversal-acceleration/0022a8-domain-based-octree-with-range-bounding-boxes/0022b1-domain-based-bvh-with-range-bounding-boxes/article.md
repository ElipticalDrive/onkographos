# Domain-based BVH with Range Bounding Boxes

Also called: BVH acceleration, Bounding Volume Hierarchy, Object Subdivision Techniques, Range-based BVH, Range-based acceleration data-structure (BVH)

## Overview
This method accelerates fiber surface extraction on tetrahedral meshes by organizing tetrahedra into a Bounding Volume Hierarchy (BVH) over the 2D range space, where each node stores a Range Bounding Box (RBB) and a list of tetrahedra whose RBBs are contained within it [S1]. It solves the problem of efficiently identifying which tetrahedra intersect a given edge of the Fiber Surface Control Polygon (FSCP) by reducing the query to a 2D intersection test between the FSCP edge and the node's RBB [S1]. Unlike the domain-based octree, which partitions the spatial domain, the BVH partitions the range space directly, resulting in fewer returned tetrahedra, smaller memory requirements, and faster queries for the same dataset [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A tetrahedral mesh $T$, the 2D range space domain $Dom_f$ (defined by the scalar field's min/max values), and construction parameters $n_S$ (number of splits) and $n_T$ (leaf threshold) [S1].
- **Output:** A BVH tree structure where each node contains an RBB and a list of tetrahedra indices [S1].

**Data Structures:**
- **Node:**
  - `rbb`: A 2D bounding box defined by $[x_{min}, x_{max}, y_{min}, y_{max}]$ in the range space.
  - `tetrahedra`: A list of indices pointing to tetrahedra in the input mesh.
- **RBB Definition:** The RBB of a tetrahedron is the axis-aligned bounding box of its projection into the 2D range space [S1]. The RBB of a node is the union of the RBBs of all tetrahedra assigned to it [S1].

**Construction Algorithm (Off-line):**
1. **Initialize Root:** Start with the root node covering the entire domain $Dom_f$ [S1]. Assign all tetrahedra whose RBBs intersect the root's RBB to the root [S1].
2. **Update RBB:** Compute the node's final RBB as the union of the RBBs of all tetrahedra currently assigned to it [S1].
3. **Stop Recursion:** If a node is given fewer tetrahedra than $n_T$, stop recursion [S1].
4. **Split:** Recursively split the node's RBB in the middle along the horizontal and vertical axes [S1]. This is performed $n_S$ times for each node, yielding an $n_S$-ary tree [S1].
5. **Assign Children:** Assign to each child node the list of tetrahedra whose RBB is completely included in the child's RBB [S1].
6. **Note:** This data structure differs from a quadtree because each node updates its RBB after its list of tetrahedra has been transferred from its parent node and before creating children nodes, yielding less regular but more refined range subdivision patterns [S1].

**Query Algorithm (On-line):**
1. **Start at Root:** Begin the query at the root node [S1].
2. **Recursive Visit:** Given a FSCP edge $e$, recursively visit each node's children if their RBB intersects or overlaps $e$ [S1].
3. **Filter:** Only tetrahedra returned by the BVH are used for fiber surface extraction [S1].

**Parameters:**
- $n_S$: Number of splits performed for each node. Typical value: 8 offered the best trade-off [S1].
- $n_T$: Leaf threshold. Typical value: 8 [S1].

**Edge Cases and Pitfalls:**
- The BVH does not encode the precise polygonal projection of tetrahedra (only RBBs), so it can return tetrahedra not intersected by the fiber surface [S1].
- Precision issues may arise when checking RBB intersection with FSCP edges in range space [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_bvh(tetrahedra, domain_rbb, n_s, n_t):
    root = Node(rbb=domain_rbb, tetrahedra=[])
    for t in tetrahedra:
        if rbb_intersects(t.rbb, domain_rbb):
            root.tetrahedra.append(t.id)
    build_recursive(root, n_s, n_t)
    return root

def build_recursive(node, n_s, n_t):
    if len(node.tetrahedra) < n_t:
        return

    # Update RBB after transferring from parent [S1]
    node.rbb = union_rbb([tetrahedra[t].rbb for t in node.tetrahedra])

    # Split RBB in the middle along horizontal and vertical axes, performed n_s times [S1]
    # Yields an n_s-ary tree
    children = []
    x_min, x_max, y_min, y_max = node.rbb
    dx = (x_max - x_min) / n_s
    dy = (y_max - y_min) / n_s
    for i in range(n_s):
        for j in range(n_s):
            child_rbb = [x_min + i*dx, x_min + (i+1)*dx, y_min + j*dy, y_min + (j+1)*dy]
            child = Node(rbb=child_rbb, tetrahedra=[])
            # Assign tetrahedra whose RBB is completely included in child's RBB [S1]
            for t_id in node.tetrahedra:
                if rbb_contains(child_rbb, tetrahedra[t_id].rbb):
                    child.tetrahedra.append(t_id)
            if len(child.tetrahedra) > 0:
                children.append(child)
                build_recursive(child, n_s, n_t)
    node.children = children

def query_bvh(node, edge):
    if not rbb_intersects(node.rbb, edge):
        return []
    results = []
    if len(node.children) == 0:
        return node.tetrahedra
    for child in node.children:
        results.extend(query_bvh(child, edge))
    return results

# Helper functions (standard geometric operations)
def rbb_intersects(rbb, edge):
    # Returns True if the 2D bounding box intersects the line segment
    return True  # [G] Standard AABB-line segment intersection logic

def rbb_contains(parent_rbb, child_rbb):
    # Returns True if child_rbb is completely inside parent_rbb
    return (parent_rbb[0] <= child_rbb[0] and parent_rbb[1] >= child_rbb[1] and
            parent_rbb[2] <= child_rbb[2] and parent_rbb[3] >= child_rbb[3])

def union_rbb(rbb_list):
    # Returns the bounding box covering all RBBs in the list
    if not rbb_list: return [0, 0, 0, 0]
    return [min(r[0] for r in rbb_list), max(r[1] for r in rbb_list),
            min(r[2] for r in rbb_list), max(r[3] for r in rbb_list)]
```

## Subtopics

### [LBVH Construction for Sparse Volumetric Data](5097-lbvh-construction-for-sparse-volumetric-data.md)
Constructs a Linear Bounding Volume Hierarchy (LBVH) specifically for sparse volumetric data to enable efficient empty-space skipping during ray marching. It adapts the LBVH construction algorithm originally designed for triangle geometry. By using the boundaries of non-empty bricks as bounding geometry, it allows for construction on the GPU at interactive frame rates for large volumes (e.g., $1024^3$) and achieves two to three times higher frame rates compared to ray marching without empty-space skipping.

### Standard BVH
A standard binary Bounding Volume Hierarchy combined with DDA traversal over $8^3$ brick leaf nodes [S3, unofficial]. Bounding Volume Hierarchies are industry standard, extensively researched, and gaining hardware acceleration support. They are not limited to voxels and support traditional triangle-based geometry, transformations, and reasonably efficient updates [S3, unofficial]. On Sponza 2k, StdBVH achieves 175.0 Mrays/s for primary rays and 102 Mrays/s for primary + 1 diffuse bounce [S3, unofficial].

## Operating characteristics
- **Memory:** On the EthaneDiol dataset, a BVH with $n_T=8$ used 116.728 MB [S1].
- **Pre-processing:** 1.478s on the EthaneDiol dataset [S1].
- **Query Efficiency:** Queries 12.654% of tetrahedra on the EthaneDiol dataset [S1].
- **Extraction Time:** 0.177s for extraction on the EthaneDiol dataset [S1].
- **Speedup:** Average speedup of 36x over the non-accelerated algorithm [S1].
- **Comparison:** The BVH is not domain-based, so it better captures the geometry of the input polygon in range-space, resulting in fewer returned tetrahedra, smaller memory requirements, and faster queries compared to the octree [S1].
- **Strengths:** Efficient filtering via 2D intersection tests in range space, adaptive subdivision patterns.
- **Limitations:** Does not encode precise polygonal projections, leading to false positives (non-intersecting tetrahedra returned) [S1].

## Background
- **Origin:** Introduced in "Fast and Exact Fiber Surfaces for Tetrahedral Meshes" [S1].
- **History:** Developed as an acceleration structure for fiber surface extraction on tetrahedral meshes, leveraging BVH techniques to optimize range-space queries [S1].
- **Usage:** Applied in volume rendering and scientific visualization for efficient extraction of fiber surfaces from tetrahedral meshes [S1].

## Sources
[S1] Fast and Exact Fiber Surfaces for Tetrahedral Meshes, https://hal.science/hal-01206120v2, fetched 2026-10-26
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
