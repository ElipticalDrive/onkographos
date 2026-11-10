# Ray Traversal of Octree Nodes

Also called: Child Octant Traversal, Interior Node Traversal, Mid-plane Intersection Traversal, Octree Node Traversal Order, Octree Traversal (OT), Octree Traversal with Mid-Plane Sorting, e-space coordinate traversal

## Overview
This method traverses an octree along a ray by computing the intersection order of the X, Y, and Z mid-planes that divide a parent node into its eight child octants, allowing the ray to march sequentially through children in the correct spatial order [S3][S4, unofficial][S2]. Unlike [Hierarchical Traversal](896-hierarchical-traversal.md), which may use bounding volume hierarchies or less structured descent, this approach explicitly sorts mid-plane intersections to determine the exact traversal sequence, exploiting binary arithmetic to minimize branching [S3][S4, unofficial][S2]. It is distinct from [Sparse Voxel Traversal](1587-octree-traversal.md) in its specific focus on the mid-plane sorting logic for interior nodes, and from [Ray-Octree Traversal (Gargantini and Atkinson variant)](1890-ray-octree-traversal-gargantini-and-atkinson-variant.md) by relying on standard floating-point comparisons rather than rational arithmetic for precision [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and an octree node defined by its center $\mathbf{c}$, half-extent $h$, and depth. The ray entry ($t_{enter}$) and exit ($t_{exit}$) parameters relative to the node's bounding box are pre-computed [S5, unofficial].
*   **Output:** A sequence of child octant indices (0–7) to visit, and potentially the values of voxels at leaf/cap nodes for isosurface reconstruction [S2][S4, unofficial].

**Data Structures:**
*   **Node:** Contains depth, node index, and pointers to children or scalar data [S3][S5, unofficial].
*   **`axis_isects[]`:** An array of size 3 storing the sorted $t$-values where the ray intersects the X, Y, and Z mid-planes of the current node [S3][S5, unofficial].
*   **`axisbit`:** A bitmask (integer) representing the current child octant index [S3][S5, unofficial].

**Steps:**
1.  **Ray-Node Intersection:** Perform a box test to determine if the ray intersects the current node. If not, terminate. Compute $t_{enter}$ and $t_{exit}$ for the node's bounding box [S5, unofficial].
2.  **Mid-Plane Computation:** Compute the octree-space coordinates of the three mid-planes (X, Y, Z) that divide the node into 8 equal child octants. The mid-plane positions are at $\mathbf{c} + \{0, h\}$ along each axis [S3][S5, unofficial].
3.  **Intersection Evaluation:** Evaluate the $t$-parameter where the ray intersects each of the three mid-planes. Let these be $t_x, t_y, t_z$ [S3][S5, unofficial].
4.  **Sorting:** Sort $t_x, t_y, t_z$ and store them in `axis_isects[]` in ascending order. This array dictates the order in which the ray crosses the boundaries between child octants [S3][S5, unofficial].
5.  **First Child Determination:** Compute the index of the first child octant intersected by the ray. This is done using constant shifting and binary-OR operations based on the ray's position relative to the node center, identical to point location [S3][S5, unofficial]. Let this initial index be `axisbit` [S3][S5, unofficial].
6.  **Sequential Marching:** Iterate through the sorted `axis_isects[]`. For each mid-plane intersection:
    *   Determine which axis (X, Y, or Z) corresponds to the current smallest $t$ value in `axis_isects[]` [S3][S5, unofficial].
    *   Update `axisbit` by inverting the bit corresponding to that axis. For example, if the ray crosses the X mid-plane, flip the X-bit in `axisbit` [S3][S5, unofficial].
    *   Visit the child octant indicated by the new `axisbit` [S3][S5, unofficial].
7.  **Leaf/Cap Handling:** If a child is a leaf or cap node, handle it using specialized routines (see Subtopics) [S2, S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_octree_node(ray_origin, ray_direction, node_center, node_half_extent, t_enter, t_exit):
    # Step 1: Compute mid-plane positions relative to node center
    # Mid-planes are at center + 0 and center + half_extent for each axis
    # However, for traversal order, we only need the t-values of intersection
    
    # Step 2 & 3: Compute t-values for X, Y, Z mid-planes
    # The mid-planes are at x=0, x=1, y=0, y=1, z=0, z=1 in local node space [-1, 1]
    # Or more simply, the ray intersects the plane x=0 (local) at t_x, etc.
    # Local ray: p(t) = (ray_origin - node_center) / node_half_extent + ray_direction * t / node_half_extent
    # Simplified: We compute t for planes at 0 and 1 in local coordinates.
    # Since the ray is linear, we can compute t for the plane at 0 (center) and 1 (edge).
    # But the algorithm sorts the mid-planes dividing the children.
    # The children are divided by planes at 0.5 in local space if node is [0,1], 
    # or 0 if node is [-1,1]. Let's assume local space [-1, 1].
    # Mid-planes are at 0.
    
    # Local ray parameters
    local_origin = (ray_origin - node_center) / node_half_extent
    local_direction = ray_direction / node_half_extent
    
    # Intersection with mid-plane at 0 for each axis
    # t = (0 - local_origin[i]) / local_direction[i]
    t_x = -local_origin.x / local_direction.x
    t_y = -local_origin.y / local_direction.y
    t_z = -local_origin.z / local_direction.z
    
    # Step 4: Sort t-values
    # We need to know which axis corresponds to each t-value
    isects = [
        {'t': t_x, 'axis': 0}, # 0 for X
        {'t': t_y, 'axis': 1}, # 1 for Y
        {'t': t_z, 'axis': 2}  # 2 for Z
    ]
    isects.sort(key=lambda k: k['t'])
    
    # Step 5: Compute first child octant
    # Determine which octant the ray enters first based on local_origin
    # If local_origin[i] < 0, we are in the lower half (bit 0), else upper (bit 1)
    # Child index bits: X is LSB, Y is middle, Z is MSB (or similar convention)
    # Convention from [S3][S5, unofficial]: binary-or as point location
    child_index = 0
    if local_origin.x >= 0: child_index |= 1
    if local_origin.y >= 0: child_index |= 2
    if local_origin.z >= 0: child_index |= 4
    
    current_child = child_index
    
    # Step 6: Sequential Marching
    for i in range(3):
        # Visit current child
        visit_child(current_child)
        
        # Determine which axis we are crossing
        axis = isects[i]['axis']
        
        # Update child index by flipping the bit for this axis
        # Bit 0 for X, Bit 1 for Y, Bit 2 for Z
        if axis == 0:
            current_child ^= 1
        elif axis == 1:
            current_child ^= 2
        elif axis == 2:
            current_child ^= 4
            
    # Visit the last child after crossing all mid-planes
    visit_child(current_child)

def visit_child(child_index):
    # Retrieve child node
    child_node = get_child_node(child_index)
    if child_node.is_leaf:
        handle_leaf(child_node)
    elif child_node.is_cap:
        handle_cap(child_node)
    else:
        # Recurse or continue traversal
        # Compute new t_enter/t_exit for child
        # This requires bounding box intersection for the child
        pass
```

**Parameters:**
*   **Sorting Cost:** Moderate initial cost for computing and sorting mid-plane intersections [S3][S5, unofficial].
*   **Traversal Cost:** Subsequent traversal is trivial, involving only bitmask inversion [S3][S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Standard floating-point comparisons may introduce errors in $t$-value comparisons, potentially leading to incorrect traversal order [S1].
*   **Cap Nodes:** Between 1 and 8 voxels in the neighborhood of a ray-intersected voxel may lie within the same cap node, requiring efficient local access [S2].
*   **Empty Nodes:** The algorithm can utilize an empty octree strategy to skip nodes that do not contain the isosurface, reducing iterations [S1].

## Subtopics

### [Ray-Octree Traversal (Gargantini and Atkinson variant)](1890-ray-octree-traversal-gargantini-and-atkinson-variant.md)
This variant explicitly sorts mid-plane intersections to march sequentially through child octants, often using neighbor-finding for non-leaf transitions. It is distinguished by its use of rational arithmetic to maintain precision in $t$-value comparisons, avoiding floating-point errors inherent in standard implementations.

### [IEEE-754 Bit Manipulation Octree Traversal](3907-ieee-754-bit-manipulation-octree-traversal.md)
This method interprets the IEEE-754 bit representation of floating-point coordinates to determine node depth and child indices, avoiding explicit pointer chasing or coordinate division. It replaces arithmetic division and modulo operations with bitwise shifts and masks on the mantissa.

### [Stack-based Octree Traversal with Direction Flipping](5007-stack-based-octree-traversal-with-direction-flipping.md)
A stack-based algorithm that handles arbitrary ray directions by flipping the coordinate system and ray parameters to ensure positive traversal direction. It uses an explicit stack to manage non-linear traversal paths and direction flipping to simplify child selection logic.

### Cap Node Neighbor Finding
For cap nodes, the algorithm skips the isovalue min/max range check and recursive child traversal. Instead, it determines the values of eight forward-neighbor voxels. It checks if a neighbor lies within the same cap node using the bitwise AND of the current child octant index (`child`) and the neighbor direction (`dir`): `if (child & dir)`. If false, the value is retrieved directly from `cap.scalars[dir_idx]`. If true, neighbor-finding logic is used [S2]. A pitfall is that 1 to 8 voxels in the neighborhood may lie within the same cap node, requiring efficient local access [S2].

## Operating characteristics
*   **Time Complexity:** Moderate initial cost for computing and sorting mid-plane intersections per node [S3][S5, unofficial]. Subsequent traversal is $O(1)$ per child, involving only bitmask inversion [S3][S5, unofficial].
*   **Memory Complexity:** Low, requiring only a small array (`axis_isects[]`) and bitmask per node traversal [S3][S5, unofficial].
*   **Performance:** On average, Octree Traversal (OT) is faster than Sphere Marching (SM) because it utilizes empty octree nodes with smaller average depth, despite potentially requiring more iterations [S1]. It produces a grid pattern in iteration counts due to dependence on node location [S1].
*   **Strengths:** Efficient skipping of empty space; simple child selection logic after initial sorting [S3][S5, unofficial].
*   **Limitations:** Floating-point precision issues in $t$-value comparisons [S1]; moderate initial sorting cost [S3][S5, unofficial].

## Background
The method originates from interactive isosurface ray tracing of large octree volumes, introduced by Wald et al. in 2006 [S2, S3][S4, S5, unofficial]. It was developed to efficiently traverse sparse voxel octrees for real-time rendering, leveraging the hierarchical structure to skip empty regions and the mid-plane sorting to determine traversal order without complex branching [S2, S3][S4, S5, unofficial].

## Sources
[S1] Rendering piecewise approximations of SDFs through analytic intersections, https://doi.org/10.1016/j.cag.2024.103981, fetched 2026-10-27
[S2] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S3] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S4] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S5] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
