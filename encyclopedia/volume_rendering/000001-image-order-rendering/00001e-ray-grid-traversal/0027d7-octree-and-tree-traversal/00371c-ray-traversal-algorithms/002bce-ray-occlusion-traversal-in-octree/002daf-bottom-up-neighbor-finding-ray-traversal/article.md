# Bottom-up Neighbor-Finding Ray Traversal

Also called: Bottom-up ray tracing in octrees, Neighbor-finding ray traversal, Octree-based Ray Tracing with Neighbor Finding

## Overview
Bottom-up neighbor-finding ray traversal is a method for intersecting rays with an octree-represented scene by determining the sequence of cells the ray passes through via neighbor lookups rather than hierarchical descent. It solves the problem of efficiently stepping from one occupied or relevant cell to the next along a ray path, avoiding the overhead of traversing empty space from the root. Unlike top-down traversal methods that descend from the root for every step, this approach starts at the first intersected cell and moves laterally or vertically through the tree structure to find the subsequent cell, offering a trade-off between implementation complexity and traversal speed depending on the density of the scene [S1].

## How it works

### Inputs and Outputs
*   **Input:** A ray defined by an origin and direction vector; an octree data structure representing the scene (Region Octree or PM Octree).
*   **Output:** A sequence of cells (nodes) intersected by the ray, and for each cell, the intersection points with the objects contained within that cell.

### Data Structures
The octree is implemented using a pointer-based representation. Each node is stored as a record of type `node` containing 10 fields [S1]:
1.  **Father:** Pointer to the parent node.
2.  **Sons[0..7]:** Pointers to the eight child nodes, corresponding to the eight octants.
3.  **Type:** Information indicating the node's status (e.g., black/FULL, white/VOID, or gray for Region Octrees; or decomposition status for PM Octrees).

If a node is a leaf, the eight son pointers point to an empty record [S1].

**Coordinate System and Labeling:**
The coordinate system partitions a cube into halves along each axis:
*   X-axis: L (Lower), R (Right/Upper)
*   Y-axis: D (Down/Lower), U (Up/Upper)
*   Z-axis: B (Back/Lower), F (Front/Upper)

Octants are labeled by concatenation (e.g., LDB) or numerically (0-7) [S1].

**Neighbor Definition:**
A node $Q$ is a neighbor of node $R$ in direction $I$ if $Q$ is the smallest block adjacent to $R$ (touching even at a point) in direction $I$ with a size greater than or equal to the block corresponding to $R$ [S1].
*   **Face-neighbors:** 6 possibilities.
*   **Edge-neighbors:** 12 possibilities.
*   **Vertex-neighbors:** 8 possibilities.
*   Total: 26 possible neighbor directions [S1].

### Algorithm Steps

1.  **Locate Initial Cell:** Find the closest bounding volume or cell $C$ to the viewpoint that is intersected by the ray. This typically involves a top-down search from the root to the first non-empty leaf or the first cell intersected by the ray [S1].
2.  **Calculate Exit Point:** Let $P$ be the point at which the ray leaves cell $C$.
3.  **Check Intersection:** Determine if cell $C$ contains an object that intersects the ray.
    *   If yes, compute the intersection, determine the appropriate object, and continue tracing secondary rays (e.g., reflection) if necessary [S1].
    *   If no, proceed to step 4.
4.  **Project Ray:** Calculate the next point $Q$ along the ray. The source describes this as $Q = P + A$, where $A$ is the ray direction vector scaled appropriately to move past the current cell boundary [S1].
5.  **Find Next Cell:** Locate the smallest cell $C'$ containing point $Q$ using neighbor-finding methods. This is the core "bottom-up" step, where the algorithm moves from the current node to its neighbor(s) rather than restarting from the root [S1].
6.  **Repeat:** Set $C = C'$ and return to Step 3.

### Ray Definition
The ray is defined parametrically. Note that the source uses inconsistent variable names ($l$ and $t$) for the parameter in the provided equations [S1]. Let $t$ be the parametric distance along the ray:

$$
x(t) = m_x \cdot t + b_x
$$
$$
y(t) = m_y \cdot t + b_y
$$
$$
z(t) = m_z \cdot t + b_z
$$

Where:
*   $(b_x, b_y, b_z)$ is the ray origin.
*   $(m_x, m_y, m_z)$ is the ray direction vector [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray_bottom_up(ray_origin, ray_direction, octree_root):
    """
    Traces a ray through an octree using bottom-up neighbor finding.
    
    Args:
        ray_origin: Vector (bx, by, bz)
        ray_direction: Vector (mx, my, mz)
        octree_root: Pointer to the root node of the octree
        
    Returns:
        List of intersection events (cell, object, distance)
    """
    
    # Step 1: Locate the first intersected cell C
    # This requires a top-down traversal to find the first cell 
    # intersected by the ray.
    current_cell = find_first_intersected_cell(ray_origin, ray_direction, octree_root)
    
    if current_cell is None:
        return []

    # Calculate the exit point P from the current cell
    # P is the intersection of the ray with the boundary of current_cell
    # furthest from the entry point.
    entry_point, exit_point = compute_cell_intersections(ray_origin, ray_direction, current_cell)
    
    intersections = []
    
    while current_cell is not None:
        # Step 3: Check if current_cell contains intersecting objects
        objects_in_cell = get_objects(current_cell)
        hit = None
        hit_distance = infinity
        
        for obj in objects_in_cell:
            dist = intersect_ray_object(ray_origin, ray_direction, obj)
            if dist is not None and dist < hit_distance:
                hit_distance = dist
                hit = obj
        
        if hit is not None:
            # Record intersection
            intersections.append((current_cell, hit, hit_distance))
            # Trace secondary rays if necessary (e.g., reflection)
            # This is outside the scope of the primary traversal loop
            break 
            
        # Step 4: Project ray to next point Q
        # Move slightly past the exit point to avoid numerical issues
        # A is the direction vector scaled. 
        # Note: The source says Q = P + A. We use a small epsilon 
        # along the direction to ensure we are inside the next cell.
        epsilon = 1e-6
        next_point = exit_point + ray_direction * epsilon
        
        # Step 5: Find the smallest cell C' containing Q using neighbor finding
        # This is the bottom-up step. We start from current_cell and 
        # traverse up and down to find the neighbor containing next_point.
        next_cell = find_neighbor_cell(current_cell, next_point)
        
        if next_cell is None:
            break
            
        current_cell = next_cell
        
        # Calculate new exit point P for the new cell
        # We need the entry point for the new cell as well to define the segment
        # However, for the loop, we just need the exit point of the NEW cell
        # to project to the NEXT cell.
        # Re-calculate intersections for the new cell segment
        # Note: The ray origin for this segment is effectively the entry into current_cell
        # But the parametric ray is global.
        
        # Optimization: The entry point into current_cell is the exit point 
        # of the previous cell (or the initial entry).
        # We need to ensure we don't re-process the entry point.
        
        # Compute exit from current_cell
        _, exit_point = compute_cell_intersections(ray_origin, ray_direction, current_cell)
        
        # Safety check: if exit_point is not beyond the entry point, 
        # we might be stuck or have a degenerate case.
        # The source emphasizes that tangent rays at a single point do NOT pass through.
        
    return intersections

def find_neighbor_cell(start_node, target_point):
    """
    Finds the smallest cell containing target_point using neighbor-finding.
    This is the core "bottom-up" logic.
    """
    # The source mentions "neighbor-finding methods" [S1].
    # A standard implementation involves:
    # 1. Check if target_point is in start_node.
    # 2. If not, find the neighbor of start_node that contains target_point.
    #    This requires traversing up to the father to find shared boundaries,
    #    then down into the appropriate son of the father or a neighbor of the father.
    
    # Since the source does not provide the detailed neighbor-finding algorithm 
    # (citing [20, 24]), we implement a conceptual placeholder that reflects 
    # the requirement to move via neighbors.
    
    # In a full implementation, this function would:
    # - Determine which face/edge/vertex of start_node is closest to target_point.
    # - Identify the neighbor in that direction.
    # - Recursively check if the neighbor contains the point.
    # - If the neighbor is too small, move to the neighbor's neighbor.
    
    # For the purpose of this article, we assume a helper function exists 
    # that implements the neighbor traversal logic described in [20, 24].
    
    return neighbor_traversal_logic(start_node, target_point)
```

### Edge Cases and Pitfalls
*   **Tangency Convention:** A ray tangent to a cell at just one point does **NOT** pass through (intersect) the cell. A ray tangent along an edge or face **DOES** pass through. This convention is critical to avoid errors, such as skipping cells or double-counting intersections [S1].
*   **Degenerate Cases:** Rays passing directly from one cell to another without passing through an intermediate cell (e.g., cell 2 to cell 3, skipping cell 10 in the source's Fig. 3 example) must be handled correctly by the neighbor-finding logic [S1].
*   **Precision:** Numerical precision issues at cell boundaries can lead to incorrect neighbor selection. The source implies the use of a small scaling factor $A$ to move past the boundary [S1].

## Operating characteristics

*   **Time Complexity:** The traversal time depends on the number of cells intersected by the ray and the cost of neighbor finding. Neighbor finding is generally faster than top-down traversal for dense scenes or long rays, as it avoids restarting from the root for each step. However, the complexity of the neighbor-finding algorithm itself can be high if not optimized [S1].
*   **Memory Complexity:** The memory usage is dominated by the octree structure. PM Octrees use significantly less space than Region Octrees for polyhedral objects [S1].
*   **Strengths:** Efficient for scenes with large empty spaces or where rays traverse many cells, as it skips empty nodes via neighbor links.
*   **Limitations:** Implementation complexity is higher than simple top-down traversal. The neighbor-finding logic can be intricate, especially for vertex and edge neighbors [S1].
*   **When to Use:** Suitable for ray tracing in octrees where performance is critical and the scene structure allows for efficient neighbor lookups.
*   **When Not to Use:** For very sparse scenes or when the overhead of neighbor finding outweighs the benefit of skipping nodes.

## Background

*   **Origin:** The method was described in a paper by researchers at the University of Maryland, College Park, focusing on ray tracing implementations using octrees [S1].
*   **Development:** The approach contrasts with top-down methods, which start at the root and traverse down. The bottom-up method follows the ray from the first intersected cell to subsequent cells via neighbors [S1].
*   **Usage:** It is used in computer graphics for generating images of 3D scenes, particularly when using pointer-based octree representations [S1].

## Sources

[S1] Implementing ray tracing with octrees and neighbor finding, https://doi.org/10.1016/0097-8493(89)90006-x, fetched 2026-10-26
