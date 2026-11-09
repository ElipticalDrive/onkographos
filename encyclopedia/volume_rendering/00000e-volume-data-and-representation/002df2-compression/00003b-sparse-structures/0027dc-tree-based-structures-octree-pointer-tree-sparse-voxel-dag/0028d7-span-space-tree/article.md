# Span Space Tree
## Overview
The Span Space Tree is a hierarchical data structure designed to accelerate the rendering and extraction of large-scale volumetric data by reducing the number of visited cells during traversal [S1, unofficial]. It refines the interval tree concept to manage scene complexity for volumes larger than standard interactive resolutions [S1, unofficial].
## How it works
The Span Space Tree embeds axis-aligned span intervals within a recursive subdivision hierarchy to represent the spatial extent of active data [G].

**Inputs and Outputs:**
- Input: A 3D volumetric dataset (scalar or vector field) and a target subdivision depth [G].
- Output: A tree where each node stores the bounding intervals of non-empty voxels within its spatial extent [G].

**Data Structures:**
Each node contains:
- `min_span`, `max_span`: Arrays of 3 integers representing the minimum and maximum voxel coordinates along each axis [G].
- `children`: List of child nodes (empty for leaves) [G].
- `is_leaf`: Boolean flag indicating leaf status [G].
- `data`: Reference to the voxel value or compressed block if the node is a leaf [G].

**Steps to Build:**
1. **Subdivision:** Recursively divide the 3D domain into sub-volumes using an octree split [G].
2. **Span Calculation:** For each node, scan its sub-volume and compute the bounding box of all active (non-zero) voxels [G].
3. **Interval Encoding:** Store the computed bounds as `min_span` and `max_span` [G].
4. **Pruning:** If no active voxels exist in the sub-volume, return `None` to prune the branch [G].
5. **Refinement:** Recurse until the specified depth is reached or the bounding box collapses to a single voxel [G].

**Formulas:**
For a node covering spatial region $R$, the span along axis $i$ is defined as:
$$ S_i = [\min_{v \in R} v_i, \max_{v \in R} v_i] $$
where $v_i$ is the integer grid coordinate of active voxel $v$ along axis $i$ [G].

**Parameters:**
- Subdivision depth: Controls tree granularity (typical values: 4–8 levels) [G].
- Active threshold: Minimum value to classify a voxel as non-empty (default: $>0$) [G].

**Edge Cases and Pitfalls:**
- Rays parallel to coordinate axes require explicit handling to avoid division by zero during slab intersection [G].
- Floating-point precision in $t$-range calculations can cause missed intersections at node boundaries; using a small epsilon tolerance is recommended [G].
- Dense data causes excessive node creation and memory overhead [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SpanNode:
    def __init__(self, min_span, max_span):
        self.min_span = list(min_span)
        self.max_span = list(max_span)
        self.children = []
        self.is_leaf = False
        self.data = None

def build_span_space_tree(volume, bounds, depth):
    min_x, min_y, min_z, max_x, max_y, max_z = bounds
    active = [(x, y, z) for x in range(min_x, max_x + 1)
              for y in range(min_y, max_y + 1)
              for z in range(min_z, max_z + 1) if volume[x][y][z] != 0]
    if not active:
        return None
    span_min = [min(v[i] for v in active) for i in range(3)]
    span_max = [max(v[i] for v in active) for i in range(3)]
    node = SpanNode(span_min, span_max)
    if depth == 0 or (span_min == span_max):
        node.is_leaf = True
        node.data = volume[span_min[0]][span_min[1]][span_min[2]]
        return node
    mid_x, mid_y, mid_z = (min_x + max_x) // 2, (min_y + max_y) // 2, (min_z + max_z) // 2
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                c_min = (min_x + dx * (mid_x + 1 - min_x), min_y + dy * (mid_y + 1 - min_y), min_z + dz * (mid_z + 1 - min_z))
                c_max = (mid_x + dx * (max_x - mid_x), max_y + dy * (max_y - mid_y), max_z + dz * (max_z - mid_z))
                child = build_span_space_tree(volume, (*c_min, *c_max), depth - 1)
                if child:
                    node.children.append(child)
    return node

def raycast(ray_o, ray_d, node, t_min, t_max):
    t_enter, t_exit = 0.0, float('inf')
    for i in range(3):
        if ray_d[i] == 0:
            if ray_o[i] < node.min_span[i] or ray_o[i] > node.max_span[i]:
                return None
        else:
            t1 = (node.min_span[i] - ray_o[i]) / ray_d[i]
            t2 = (node.max_span[i] - ray_o[i]) / ray_d[i]
            if t1 > t2: t1, t2 = t2, t1
            t_enter = max(t_enter, t1)
            t_exit = min(t_exit, t2)
    if t_enter > t_max or t_exit < t_min:
        return None
    if node.is_leaf:
        pos = [ray_o[i] + ray_d[i] * t_enter for i in range(3)]
        cell = [int(p) for p in pos]
        return tuple(cell)
    for child in node.children:
        res = raycast(ray_o, ray_d, child, t_enter, t_exit)
        if res:
            return res
    return None
```
## Subtopics
## Operating characteristics
- **Time Complexity:** Building time scales with volume size and sparsity [G]. Traversal time is reduced compared to brute-force access due to empty span culling [G].
- **Memory Complexity:** Proportional to the number of non-empty spans [G].
- **Performance:** Enables interactive rendering of data up to $512^3$ resolution [S1, unofficial].
- **Strengths:** Efficient culling of empty regions; suitable for large-scale volumes [G].
- **Limitations:** Building is computationally expensive for very large datasets; performance degrades if data is dense [G].
## Background
The Span Space Tree was introduced by Livnat et al. as a refinement of the interval tree concept to manage scene complexity for larger volume data [S1, unofficial]. It builds on earlier work by Wilhelms and Van Gelder, who proposed a min-max hierarchy embedded in a branch-on-need octree for trivially ignoring empty regions [S1, unofficial]. The method is cited in the context of enabling interactive rendering of large volumetric datasets [S1, unofficial].
## Sources
[S1] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
