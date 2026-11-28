# Surface Area Heuristic (SAH)

Also called: Crofton’s Theorem based cost model, Surface Area Heuristic (SAH) / Crofton’s Theorem

## Overview
The Surface Area Heuristic (SAH) is a cost model used to determine the optimal partitioning of primitives in Bounding Volume Hierarchy (BVH) construction, both for top-down recursive splitting and iterative insertion [S1, unofficial]. It minimizes the estimated ray tracing cost by leveraging the probability that a ray intersects a bounding volume, which is proportional to its surface area [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of primitives $P$ enclosed in a parent Bounding Volume $B$ [S1, unofficial].
*   **Output:** A partition of $P$ into subsets $B_1$ and $B_2$ (and potentially more) that minimizes the SAH cost function, or the decision to attach a primitive directly to the current node in iterative insertion [S1, unofficial].

**Data Structures:**
*   **Bounding Volume (BV):** Typically an Axis-Aligned Bounding Box (AABB) [S1, unofficial]. The layout includes minimum and maximum coordinates defining the volume [G]. The surface area is calculated from these extents [G].
*   **Primitive Set:** A list of primitives associated with each node [G].

**Core Formula:**
The cost $C(B)$ of a node containing a set of polygons $B$ is estimated by the weighted sum of the surface areas of its children multiplied by the number of primitives they contain [S1, unofficial]:

$$ C(B) = \min_{B \subseteq P(B)} \left[ Area(B_1) \cdot N(B_1) + Area(B_2) \cdot N(B_2) \right] $$

Where:
*   $B$ is the set of polygons in the parent BV [S1, unofficial].
*   $B_1$ and $B_2$ are the subsets of primitives assigned to the left and right children, respectively [S1, unofficial].
*   $Area(B_i)$ is the surface area of the bounding volume enclosing subset $B_i$ [G].
*   $N(B_i)$ is the number of primitives in subset $B_i$ [S1, unofficial].

**Theoretical Basis:**
The heuristic relies on Crofton’s Theorem, which states that for a convex body, the total number of rays hitting an object is proportional to its surface area [S2, unofficial]. Consequently, the probability of a ray hitting an object completely inside a cell is proportional to the object's surface area relative to the cell's properties [S2, unofficial]. The cost model assumes that the traversal cost is proportional to the surface area of the bounding volumes tested [S1, S2, unofficial].

**Algorithm Steps:**

1.  **Bounding Volume Calculation:** Calculate the bounding volume for the given set of objects [S1, unofficial].
2.  **Partitioning (Top-Down):**
    *   Partition the set of objects into two (or more) subsets [S1, unofficial].
    *   Calculate the bounding volumes for each subset [G].
    *   Compute the cost $C(B)$ for the partition using the formula above [S1, unofficial].
    *   Seek the optimal split that minimizes this cost [S1, unofficial]. Exhaustively searching all possible partitions is not practical [S1, unofficial]. Heuristic approaches represent objects by their centroids, determine the axis of largest extent, sort points along this axis, and use a plane-sweep to find the minimum cost split [S1, unofficial].
3.  **Recursive Construction:** Recursively construct BVHs for each subset [S1, unofficial].
4.  **Iterative Insertion (Goldsmith/Salmon):**
    *   For each new object, propagate it into the subtree that causes the smallest cost increase for ray tracing [S1, unofficial].
    *   **Tie-Breaking:** If both subtrees cause equal costs (e.g., 0), use a secondary heuristic, such as the number of triangles in the subtree [S1, unofficial].
    *   **Non-Binary Fallback:** If all subtrees cause too high costs (e.g., area increase reaches 90% of the parent's area), attach the object as a direct child of the current node, allowing for non-binary BVHs [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Node:
    def __init__(self, primitives=None, bv=None):
        self.primitives = primitives if primitives is not None else []
        self.bv = bv
        self.children = []

def compute_bounding_volume(primitives):
    if not primitives:
        return {'min': (0, 0, 0), 'max': (0, 0, 0)}
    min_x, min_y, min_z = float('inf'), float('inf'), float('inf')
    max_x, max_y, max_z = float('-inf'), float('-inf'), float('-inf')
    for p in primitives:
        min_x = min(min_x, p.min[0]); min_y = min(min_y, p.min[1]); min_z = min(min_z, p.min[2])
        max_x = max(max_x, p.max[0]); max_y = max(max_y, p.max[1]); max_z = max(max_z, p.max[2])
    return {'min': (min_x, min_y, min_z), 'max': (max_x, max_y, max_z)}

def surface_area(bv):
    d = (bv['max'][0] - bv['min'][0], bv['max'][1] - bv['min'][1], bv['max'][2] - bv['min'][2])
    return 2.0 * (d[0]*d[1] + d[1]*d[2] + d[2]*d[0])

def calculate_sah_cost(primitives, bv):
    return surface_area(bv) * len(primitives)

def sort_indices_by_axis(centroids, axis):
    return sorted(range(len(centroids)), key=lambda i: centroids[i][axis])

def find_optimal_split_top_down(primitives, parent_bv):
    if len(primitives) <= 1:
        return -1, primitives, []

    centroids = [p.centroid for p in primitives]
    extent = (parent_bv['max'][0] - parent_bv['min'][0],
              parent_bv['max'][1] - parent_bv['min'][1],
              parent_bv['max'][2] - parent_bv['min'][2])
    axis = max(range(3), key=lambda a: extent[a])

    sorted_indices = sort_indices_by_axis(centroids, axis)

    min_cost = float('inf')
    best_split_index = -1

    left_prims = []
    right_prims = list(primitives)

    left_bv = compute_bounding_volume(left_prims)
    right_bv = compute_bounding_volume(right_prims)

    for i in range(len(primitives) - 1):
        idx = sorted_indices[i]
        prim = primitives[idx]
        left_prims.append(prim)
        right_prims.pop(idx)

        left_bv = compute_bounding_volume(left_prims)
        right_bv = compute_bounding_volume(right_prims)

        cost = calculate_sah_cost(left_prims, left_bv) + calculate_sah_cost(right_prims, right_bv)

        if cost < min_cost:
            min_cost = cost
            best_split_index = i

    return best_split_index, left_prims, right_prims

def insert_iterative(node, primitive):
    if not node.children:
        node.children.append(Node([primitive]))
        return

    min_increase = float('inf')
    best_child = None

    for child in node.children:
        new_prims = child.primitives + [primitive]
        new_bv = compute_bounding_volume(new_prims)
        old_cost = calculate_sah_cost(child.primitives, child.bv)
        new_cost = calculate_sah_cost(new_prims, new_bv)
        cost_increase = new_cost - old_cost
        if cost_increase < min_increase:
            min_increase = cost_increase
            best_child = child

    parent_area = surface_area(node.bv)
    new_bv_area = surface_area(compute_bounding_volume(best_child.primitives + [primitive]))

    if new_bv_area > 0.9 * parent_area:
        node.children.append(Node([primitive]))
        return

    insert_iterative(best_child, primitive)
```

**Edge Cases and Pitfalls:**
*   **Exhaustive Search:** Finding the true global minimum split is computationally expensive and not practical [S1, unofficial]. Heuristics like plane-sweep on sorted centroids are necessary [S1, unofficial].
*   **Non-Binary Trees:** The heuristic allows for non-binary BVHs if the cost of splitting is too high (e.g., >90% area increase) [S1, unofficial].
*   **Tie-Breaking:** Equal costs require a secondary heuristic, such as primitive count [S1, unofficial].

## Subtopics

### Bounding Volume Intersection Cost Model
A cost model for ray intersection with a subset of the scene enclosed in a Bounding Volume (BV), used to evaluate the trade-off between BV simplicity and compactness [S1, unofficial]. The total cost $T$ is defined as $T = n \cdot B + m \cdot I$, where $n$ is the number of rays tested against the BV, $B$ is the cost for one ray-BV intersection test, $m$ is the number of rays that actually intersect the BV, and $I$ is the cost for testing the objects in the BV [S1, unofficial]. The goal is to minimize $T$ [S1, unofficial]. Simple BVs (e.g., sphere or box) have small intersection costs $B$ but usually result in a large number of ray hits $m$, while compact BVs (e.g., exact convex hull) have small $m$ but high intersection costs $B$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The cost calculation for a single split is $O(N)$ where $N$ is the number of primitives [G]. Finding the optimal split via plane-sweep is $O(N \log N)$ due to sorting [G].
*   **Memory Complexity:** Depends on the BVH structure; typically $O(N)$ for $N$ primitives [G].
*   **Strengths:** Provides a robust, geometry-independent metric for balancing the BVH tree, leading to efficient ray traversal [S1, S2, unofficial].
*   **Limitations:** The heuristic assumes uniform ray distribution and may not be optimal for all scene types or ray distributions [G]. Exhaustive optimization is impractical [S1, unofficial].
*   **When to use:** Standard choice for BVH construction in ray tracers [S1, S2, unofficial].
*   **When not to use:** When scene-specific heuristics are known to be more accurate, or when build time is extremely constrained and simpler splits are acceptable [G].

## Background
The Surface Area Heuristic is based on Crofton’s Theorem, which relates the number of rays hitting an object to its surface area [S2, unofficial]. It was popularized in the context of BVH construction by Goldsmith and Salmon for iterative insertion and is widely used in top-down construction methods [S1, unofficial]. It addresses the problem of efficiently organizing primitives for ray tracing by minimizing the expected number of intersection tests [S1, S2, unofficial].

## Sources
[S1] Advanced Computer Graphics, https://cgvr.cs.uni-bremen.de/teaching/cg2_13/folien/03%20-%20Acceleration%20data%20structures.pdf, fetched 2026-10-24 (unofficial)
[S2] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
