# Bottom-up Neighbor-Finding Ray Traversal

Also called: Bottom-up ray tracing in octrees, Neighbor-finding ray traversal, Octree-based Ray Tracing with Neighbor Finding

## Overview
Bottom-up neighbor-finding ray traversal is a method for intersecting rays with an octree-represented scene by determining the sequence of cells the ray passes through via neighbor lookups rather than hierarchical descent [S1]. It solves the problem of efficiently stepping from one relevant cell to the next along a ray path, avoiding the overhead of restarting traversal from the root for each step [S1]. Unlike top-down methods that descend from the root for every step, this approach starts at the first intersected cell and moves through the tree structure to find the subsequent cell, trading implementation complexity for traversal speed depending on scene density [S1].

## How it works

### Inputs and Outputs
*   **Input:** A ray defined by an origin and direction vector; an octree data structure representing the scene (Region Octree or PM Octree) [S1].
*   **Output:** A sequence of cells (nodes) intersected by the ray, and for each cell, the intersection points with the objects contained within that cell [S1].

### Data Structures
The octree is implemented using a pointer-based representation [S1]. Each node is stored as a record of type `node` containing 10 fields [S1]. Fields 1-9 contain pointers to the node's father and its eight sons, which correspond to the eight octants [S1]. If the node is a leaf, the eight son pointers point to an empty record [S1]. Field 10 contains type information (e.g., black/FULL, white/VOID, or gray for Region Octrees; or decomposition status for PM Octrees) [S1].

The coordinate system partitions a cube into halves along each axis [S1]. The X-axis uses L (Lower) and R (Upper), the Y-axis uses D (Lower) and U (Upper), and the Z-axis uses B (Lower) and F (Upper) [S1]. Octants are labeled by concatenation (e.g., LDB) or numerically (0-7) [S1].

A node $Q$ is a neighbor of node $R$ in direction $I$ if $Q$ is the smallest block adjacent to $R$ (touching even at a point) in direction $I$ with a size greater than or equal to the block corresponding to $R$ [S1]. In three dimensions, neighbors are classified by adjacency type: face-neighbors (6 possibilities), edge-neighbors (12 possibilities), and vertex-neighbors (8 possibilities), totaling 26 possible neighbor directions [S1].

### Algorithm Steps
1.  **Locate Initial Cell:** Find the closest bounding volume or cell $C$ to the viewpoint that is intersected by the ray [S1].
2.  **Calculate Exit Point:** Let $P$ be the point at which the ray leaves cell $C$ [S1].
3.  **Check Intersection:** Determine if cell $C$ contains an object that intersects the ray [S1]. If it intersects more than one object, determine the appropriate object and continue tracing secondary rays if necessary [S1]. If no intersection occurs, proceed to step 4 [S1].
4.  **Project Ray:** Calculate the next point $Q$ along the ray using $Q = P + A$ [S1].
5.  **Find Next Cell:** Locate the smallest cell $C'$ containing point $Q$ using neighbor-finding methods [S1].
6.  **Repeat:** Set $C = C'$ and return to step 3 [S1].

### Ray Definition
The ray is defined parametrically [S1]. The source uses inconsistent variable names ($l$ and $t$) for the parameter in its equations, so we standardize to $t$ [S1]. Let $t$ be the parametric distance along the ray:

$$
x(t) = m_x \cdot t + b_x
$$
$$
y(t) = m_y \cdot t + b_y
$$
$$
z(t) = m_z \cdot t + b_z
$$

Where $(b_x, b_y, b_z)$ is the ray origin and $(m_x, m_y, m_z)$ is the ray direction vector [S1].

### Parameters
Typical values are not given in the sources.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray_bottom_up(ray_origin, ray_direction, octree_root):
    """Traces a ray through an octree using bottom-up neighbor finding [S1]."""
    # Step 1: Locate initial cell C intersected by ray [S1]
    current_cell = find_first_intersected_cell(ray_origin, ray_direction, octree_root)
    if current_cell is None:
        return []

    # Step 2: Calculate exit point P from current cell [S1]
    entry_point, exit_point = compute_cell_intersections(ray_origin, ray_direction, current_cell)

    intersections = []
    while current_cell is not None:
        # Step 3: Check intersection with objects in cell [S1]
        objects = get_objects(current_cell)
        hit = None
        hit_dist = float('inf')
        for obj in objects:
            dist = intersect_ray_object(ray_origin, ray_direction, obj)
            if dist is not None and dist < hit_dist:
                hit_dist = dist
                hit = obj

        if hit is not None:
            intersections.append((current_cell, hit, hit_dist))
            # Trace secondary rays if necessary [S1]
            break

        # Step 4: Project ray to next point Q = P + A [S1]
        # A is a small offset along the ray direction to advance past the boundary [G]
        epsilon = 1e-6
        next_point = exit_point + ray_direction * epsilon

        # Step 5: Find smallest cell C' containing Q using neighbor finding [S1]
        next_cell = find_neighbor_cell(current_cell, next_point)
        if next_cell is None:
            break

        current_cell = next_cell
        entry_point, exit_point = compute_cell_intersections(ray_origin, ray_direction, current_cell)

    return intersections

def find_neighbor_cell(start_node, target_point):
    """Finds the smallest cell containing target_point using neighbor-finding [S1]."""
    # Standard octree neighbor traversal logic [G]
    node = start_node
    while node is not None:
        direction = get_neighbor_direction(node, target_point)
        if direction is not None:
            sibling = get_sibling(node, direction)
            if sibling is not None:
                return descend_to_leaf(sibling, target_point)
        node = node.father
    return None

# Standard helper operations [G]
def find_first_intersected_cell(origin, direction, root):
    # Traverses from root to find first cell intersected by ray [G]
    pass

def compute_cell_intersections(origin, direction, cell):
    # Returns entry and exit parametric distances t for cell bounds [G]
    pass

def get_objects(node):
    # Returns list of geometric primitives stored in leaf node [G]
    pass

def intersect_ray_object(origin, direction, obj):
    # Returns parametric distance t to first intersection, or None [G]
    pass

def get_neighbor_direction(node, point):
    # Determines which of the 26 neighbor directions contains point relative to node bounds [G]
    pass

def get_sibling(node, direction):
    # Returns pointer to sibling node in given direction, or None [G]
    pass

def descend_to_leaf(node, point):
    # Traverses down from node to leaf containing point [G]
    pass
```

### Edge Cases and Pitfalls
*   **Tangency Convention:** A ray tangent to a cell at just one point does **NOT** pass through (intersect) the cell [S1]. A ray tangent along an edge or face **DOES** pass through [S1]. This convention is critical to avoid errors [S1].
*   **Degenerate Cases:** Rays passing directly from one cell to another without passing through an intermediate cell (e.g., cell 2 to cell 3, skipping cell 10 in the source's Fig. 3 example) must be handled correctly by the neighbor-finding logic [S1].
*   **Precision:** Numerical precision issues at cell boundaries can lead to incorrect neighbor selection [G]. The offset $A$ in step 4 must be chosen carefully to avoid self-intersection or missing cells [G].

## Operating characteristics
*   **Time Complexity:** The traversal time depends on the number of cells intersected by the ray and the cost of neighbor finding [S1]. As long as the cost of moving between adjacent cells is relatively low, the method saves time over the cost of intersecting the ray with every object in the scene [S1].
*   **Memory Complexity:** The memory usage is dominated by the octree structure [S1]. PM Octrees use significantly less space than Region Octrees for polyhedral objects [S1].
*   **Strengths:** Efficient for scenes where rays traverse many cells, as it skips empty nodes via neighbor links [S1].
*   **Limitations:** Implementation complexity is higher than simple top-down traversal [S1]. The neighbor-finding logic can be intricate, especially for vertex and edge neighbors [S1].
*   **When to Use:** Suitable for ray tracing in octrees where performance is critical and the scene structure allows for efficient neighbor lookups [S1].
*   **When Not to Use:** Not given in the sources.

## Background
*   **Origin:** The method was described by researchers at the University of Maryland, College Park, focusing on ray tracing implementations using octrees [S1].
*   **Development:** The approach contrasts with top-down methods, which start at the root and traverse down [S1]. The bottom-up method follows the ray from the first intersected cell to subsequent cells via neighbors [S1].
*   **Usage:** It is used in computer graphics for generating images of 3D scenes, particularly when using pointer-based octree representations [S1].

## Sources
[S1] Implementing ray tracing with octrees and neighbor finding, https://doi.org/10.1016/0097-8493(89)90006-x, fetched 2026-10-26
