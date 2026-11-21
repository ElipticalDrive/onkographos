# Surface Area Heuristic (SAH)

Also called: Crofton’s Theorem based cost model, Surface Area Heuristic (SAH) / Crofton’s Theorem

## Overview
The Surface Area Heuristic (SAH) is a cost model used to determine the optimal partitioning of primitives in Bounding Volume Hierarchy (BVH) construction, both for top-down recursive splitting and iterative insertion [S1, S2, unofficial]. It minimizes the estimated ray tracing cost by leveraging the probability that a ray intersects a bounding volume, which is proportional to its surface area [S1, S2, unofficial]. It sets itself apart from sibling methods like [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) or [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) by operating on the acceleration structure's topology rather than the geometric intersection logic itself, providing a general framework for spatial partitioning efficiency [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of primitives $P$ (e.g., triangles) enclosed in a parent Bounding Volume $B$ with surface area $Area(B)$.
*   **Output:** A partition of $P$ into subsets $B_1$ and $B_2$ (and potentially more) that minimizes the SAH cost function, or the decision to attach a primitive directly to the current node in iterative insertion.

**Data Structures:**
*   **Bounding Volume (BV):** Typically an Axis-Aligned Bounding Box (AABB) or Oriented Bounding Box (OBB). The layout includes minimum and maximum coordinates defining the volume. The surface area is calculated from these extents.
*   **Primitive Set:** A list of primitives associated with each node.

**Core Formula:**
The cost $C(B)$ of a node containing a set of polygons $B$ is estimated by the weighted sum of the surface areas of its children multiplied by the number of primitives they contain [S1, unofficial]:

$$ C(B) = \min_{B_1 \subseteq P(B)} \left[ Area(B_1) \cdot N(B_1) + Area(B_2) \cdot N(B_2) \right] $$

Where:
*   $B$ is the set of polygons in the parent BV.
*   $B_1$ and $B_2$ are the subsets of primitives assigned to the left and right children, respectively.
*   $Area(B_i)$ is the surface area of the bounding volume enclosing subset $B_i$.
*   $N(B_i)$ is the number of primitives in subset $B_i$.

**Theoretical Basis:**
The heuristic relies on Crofton’s Theorem, which states that for a convex body, the total number of rays hitting an object is proportional to its surface area [S2, unofficial]. Consequently, the probability of a ray hitting an object completely inside a cell is proportional to the object's surface area relative to the cell's properties [S2, unofficial]. The cost model assumes that the traversal cost is proportional to the surface area of the bounding volumes tested [S1, S2, unofficial].

**Algorithm Steps:**

1.  **Bounding Volume Calculation:** Calculate the bounding volume for the given set of objects [S1, unofficial].
2.  **Partitioning (Top-Down):**
    *   Partition the set of objects into two (or more) subsets [S1, unofficial].
    *   Calculate the bounding volumes for each subset.
    *   Compute the cost $C(B)$ for the partition using the formula above.
    *   Seek the optimal split that minimizes this cost [S1, unofficial].
    *   *Note:* Exhaustively searching all possible partitions is not practical [S1, unofficial]. Heuristic approaches, such as representing objects by their centroids, determining the axis of largest extent, sorting points along this axis, and using a plane-sweep to find the minimum cost split, are typically employed [S1, unofficial].
3.  **Recursive Construction:** Recursively construct BVHs for each subset [S1, unofficial].
4.  **Iterative Insertion (Goldsmith/Salmon):**
    *   For each new object, propagate it into the subtree that causes the smallest cost increase for ray tracing [S1, unofficial].
    *   **Tie-Breaking:** If both subtrees cause equal costs (e.g., 0), use a secondary heuristic, such as the number of triangles in the subtree [S1, unofficial].
    *   **Non-Binary Fallback:** If all subtrees cause too high costs (e.g., area increase reaches 90% of the parent's area), attach the object as a direct child of the current node, allowing for non-binary BVHs [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_sah_cost(subset_primitives, bounding_volume):
    """
    Calculates the SAH cost for a node.
    Note: The formula in [S1, unofficial] defines the cost of the *split* (children).
    Here we assume the cost contribution of a node is proportional to 
    Area(BV) * NumPrimitives, which is minimized by the split choice.
    """
    area = bounding_volume.surface_area()
    num_prims = len(subset_primitives)
    return area * num_prims

def find_optimal_split_top_down(primitives, parent_bv):
    """
    Finds the split that minimizes SAH cost for top-down construction.
    Uses a heuristic plane-sweep approach as exhaustive search is impractical [S1, unofficial].
    """
    # 1. Represent objects by centroids [S1, unofficial]
    centroids = [p.centroid() for p in primitives]
    
    # 2. Determine axis of largest extent [S1, unofficial]
    extent = parent_bv.extent()
    axis = argmax(extent)
    
    # 3. Sort points along this axis [S1, unofficial]
    sorted_indices = sort_indices_by_axis(centroids, axis)
    
    # 4. Plane-sweep to find minimum cost [S1, unofficial]
    min_cost = infinity
    best_split_index = -1
    
    # Initialize left and right subsets
    left_prims = []
    right_prims = primitives # Initially all in right
    
    # Calculate initial right BV and cost
    right_bv = compute_bounding_volume(right_prims)
    current_right_cost = calculate_sah_cost(right_prims, right_bv)
    
    for i in range(len(primitives) - 1): # Split between i and i+1
        # Move primitive i from right to left
        prim = primitives[sorted_indices[i]]
        left_prims.append(prim)
        right_prims.remove(prim)
        
        # Update bounding volumes
        left_bv = compute_bounding_volume(left_prims)
        right_bv = compute_bounding_volume(right_prims)
        
        # Calculate split cost
        cost = calculate_sah_cost(left_prims, left_bv) + calculate_sah_cost(right_prims, right_bv)
        
        if cost < min_cost:
            min_cost = cost
            best_split_index = i
            
    return best_split_index, left_prims, right_prims

def insert_iterative(node, primitive):
    """
    Inserts a primitive into an existing BVH using SAH for iterative insertion [S1, unofficial].
    """
    # Calculate cost increase for inserting into each child
    cost_increases = []
    for child in node.children:
        # Estimate new BV if primitive is added
        new_bv = compute_bounding_volume(child.primitives + [primitive])
        old_cost = calculate_sah_cost(child.primitives, child.bv)
        new_cost = calculate_sah_cost(child.primitives + [primitive], new_bv)
        cost_increase = new_cost - old_cost
        cost_increases.append((child, cost_increase))
        
    # Find child with smallest cost increase [S1, unofficial]
    best_child, min_increase = min(cost_increases, key=lambda x: x[1])
    
    # Check for high cost increase (fallback to non-binary) [S1, unofficial]
    # Threshold: area increase to 90% of parent's area
    parent_area = node.bv.surface_area()
    new_bv_area = compute_bounding_volume(best_child.primitives + [primitive]).surface_area()
    
    if new_bv_area > 0.9 * parent_area:
        # Attach as direct child of current node [S1, unofficial]
        node.children.append(Node([primitive]))
        return
        
    # Recursively insert into best child
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
*   **Time Complexity:** The cost calculation for a single split is $O(N)$ where $N$ is the number of primitives, assuming bounding volume updates are efficient. Finding the optimal split via plane-sweep is $O(N \log N)$ due to sorting [S1, unofficial].
*   **Memory Complexity:** Depends on the BVH structure; typically $O(N)$ for $N$ primitives.
*   **Strengths:** Provides a robust, geometry-independent metric for balancing the BVH tree, leading to efficient ray traversal [S1, S2, unofficial]. It effectively balances the number of nodes visited against the cost of intersection tests [S1, unofficial].
*   **Limitations:** The heuristic assumes uniform ray distribution and may not be optimal for all scene types or ray distributions. Exhaustive optimization is impractical [S1, unofficial].
*   **When to use:** Standard choice for BVH construction in ray tracers [S1, S2, unofficial].
*   **When not to use:** When scene-specific heuristics (e.g., density-based) are known to be more accurate, or when build time is extremely constrained and simpler splits are acceptable.

## Background
The Surface Area Heuristic is based on Crofton’s Theorem, which relates the number of rays hitting an object to its surface area [S2, unofficial]. It was popularized in the context of BVH construction by Goldsmith and Salmon for iterative insertion and is widely used in top-down construction methods [S1, S2, unofficial]. It addresses the problem of efficiently organizing primitives for ray tracing by minimizing the expected number of intersection tests [S1, S2, unofficial].

## Sources
[S1] Advanced Computer Graphics, https://cgvr.cs.uni-bremen.de/teaching/cg2_13/folien/03%20-%20Acceleration%20data%20structures.pdf, fetched 2026-10-24 (unofficial)
[S2] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
