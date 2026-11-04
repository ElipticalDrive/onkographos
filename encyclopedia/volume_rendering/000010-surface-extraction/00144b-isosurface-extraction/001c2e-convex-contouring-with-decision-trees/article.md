# Convex Contouring with Decision Trees

Also called: Convex Contouring, Convex contouring of volumetric data

## Overview
Convex Contouring with Decision Trees is a table-driven isosurface extraction technique that generates topologically consistent surfaces where the negative space inside each cell is convex [S1]. Unlike [Marching Cubes](30-isosurface-extraction.md), which may produce disconnected or non-convex negative space, this method ensures that the polygonal contour inside each cell encloses a convex hull of negative corners and edge intersections [S1]. This convexity property enables fast collision detection on the triangulated isosurface [S1]. The method distinguishes itself by using pre-computed decision trees to dynamically select the correct triangulation from a set of candidates based on the exact positions of edge intersections, rather than relying solely on vertex signs [S1].

## How it works
The algorithm assumes the underlying implicit function is trilinear within each cell [S1]. The core mechanism involves identifying the convex hull of the negative space (defined by negative cell corners and edge intersections) and extracting the portion of this hull that lies interior to the cell [S1].

### Inputs and Outputs
*   **Input:** A regular grid of volumetric data with scalar values at vertices; an isovalue $I$.
*   **Output:** A triangle mesh representing the isosurface.

### Data Structures
1.  **Composite Look-up Table:** Maps each sign configuration (the signs of the 8 cell corners relative to the isovalue) to a compact decision tree [S1].
2.  **Decision Tree:** A binary tree structure where each node represents a "4-point test" [S1].
    *   **Node:** Contains indices for four distinct edge intersections $(V_1, V_2, V_3, V_4)$.
    *   **Children:** Left child (test false), Right child (test true).
    *   **Leaf:** Contains the final triangulation (a set of triangles) for the patch [S1].
3.  **Patch Data:** A connected component of the convex contour, which may contain holes [S1].

### Steps

1.  **Cell Traversal:** Iterate through all cells in the grid.
2.  **Sign Configuration:** Determine the sign of each of the 8 cell corners relative to the isovalue $I$. Let $s_i \in \{-1, 1\}$ be the sign of corner $i$.
3.  **Edge Intersection Calculation:** For each edge of the cell where the sign changes, calculate the intersection point $P$ using linear interpolation:
    $$ t = \frac{I - f_a}{f_b - f_a} $$
    $$ P = V_a + t(V_b - V_a) $$
    where $f_a, f_b$ are scalar values at endpoints $V_a, V_b$ [S1].
4.  **Patch Identification:** Identify the linear convex contours on the faces of the cell. These contours connect edge intersections and are uniquely determined by the corner signs [S1]. Group these contours into connected components called "patches" [S1].
5.  **Pre-triangulation (Filtering):** Construct the set $S$ of all possible triangulations for the patch. The vertices of these triangles are restricted to the fixed cell edge intersections [S1]. Eliminate triangulations containing "ineligible" triangles. A triangle is ineligible if it lies completely to the back of all non-degenerate border triangles formed by the patch boundary edges [S1]. This reduces the size of $S$ [S1].
6.  **Decision Tree Lookup:** Retrieve the pre-computed decision tree from the look-up table corresponding to the current sign configuration [S1].
7.  **Triangulation Selection:** Walk down the decision tree using 4-point tests to identify the correct triangulation from the reduced set $S$ [S1].
8.  **Mesh Generation:** Output the triangles from the selected triangulation. Fill the negative regions on the cell faces with triangles to complete the convex hull of the negative space [S1].

### The 4-Point Test
The decision tree nodes perform a geometric predicate to eliminate incorrect triangulations [S1]. Given four distinct edge intersections $V_1, V_2, V_3, V_4$:

1.  Form the triangle $T = (V_1, V_2, V_3)$.
2.  Determine if $V_4$ lies on the "front-facing" side of $T$. This is typically determined by the sign of the dot product between the triangle normal $\mathbf{n} = (V_2 - V_1) \times (V_3 - V_1)$ and the vector $(V_4 - V_1)$.
    *   If $V_4$ is on the front-facing side (i.e., $\mathbf{n} \cdot (V_4 - V_1) > 0$):
        *   The inverted triangle $(V_1, V_3, V_2)$ does not belong to the convex contour.
        *   Triangles $(V_1, V_2, V_4)$, $(V_2, V_3, V_4)$, and $(V_3, V_1, V_4)$ do not satisfy the convex hull property [S1].
    *   If $V_4$ is NOT on the front-facing side (i.e., $\mathbf{n} \cdot (V_4 - V_1) \le 0$):
        *   Triangles $(V_1, V_2, V_3)$, $(V_1, V_4, V_2)$, $(V_2, V_4, V_3)$, and $(V_1, V_3, V_4)$ do not lie on the convex contour [S1].
    *   If all vertices are coplanar, either choice can be made [S1].

Each test eliminates triangulations in $S$ that contain any of the ruled-out triangles [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def convex_contouring(grid, isovalue):
    """
    Main entry point for Convex Contouring.
    grid: 3D array of scalar values.
    isovalue: The threshold for surface extraction.
    Returns: List of triangles (each triangle is a tuple of 3 vertices).
    """
    triangles = []
    # Pre-computed lookup table mapping sign_config -> decision_tree
    # decision_tree is a Node object or None
    lookup_table = build_lookup_table() 
    
    # Iterate over all cells (assuming 0-indexed, dimensions D_x, D_y, D_z)
    for x in range(grid.shape[0] - 1):
        for y in range(grid.shape[1] - 1):
            for z in range(grid.shape[2] - 1):
                cell_triangles = process_cell(grid, x, y, z, isovalue, lookup_table)
                triangles.extend(cell_triangles)
                
    return triangles

def process_cell(grid, x, y, z, isovalue, lookup_table):
    # 1. Get corner values and signs
    corner_values = []
    signs = []
    # Define corner indices for a cube (0-7)
    # 0: (0,0,0), 1: (1,0,0), 2: (0,1,0), 3: (1,1,0)
    # 4: (0,0,1), 5: (1,0,1), 6: (0,1,1), 7: (1,1,1)
    corners = [
        (x, y, z), (x+1, y, z), (x, y+1, z), (x+1, y+1, z),
        (x, y, z+1), (x+1, y, z+1), (x, y+1, z+1), (x+1, y+1, z+1)
    ]
    
    for cx, cy, cz in corners:
        val = grid[cx, cy, cz]
        corner_values.append(val)
        signs.append(1 if val >= isovalue else -1)
        
    sign_config = tuple(signs)
    
    # 2. Calculate edge intersections
    # Edges connect specific corner indices
    edges = [
        (0,1), (2,3), (4,5), (6,7), # x-edges
        (0,2), (1,3), (4,6), (5,7), # y-edges
        (0,4), (1,5), (2,6), (3,7)  # z-edges
    ]
    
    edge_intersections = {} # Map edge_index -> Point
    
    for edge_idx, (c1, c2) in enumerate(edges):
        if signs[c1] != signs[c2]:
            v1 = corner_values[c1]
            v2 = corner_values[c2]
            p1 = corners[c1]
            p2 = corners[c2]
            
            # Linear interpolation
            t = (isovalue - v1) / (v2 - v1)
            # Interpolate coordinates
            px = p1[0] + t * (p2[0] - p1[0])
            py = p1[1] + t * (p2[1] - p1[1])
            pz = p1[2] + t * (p2[2] - p1[2])
            edge_intersections[edge_idx] = (px, py, pz)
            
    # 3. Identify patches and boundary edges
    # This step identifies the linear convex contours on faces.
    # For implementation, we identify which edges are part of the contour.
    # A face contour exists if the 4 corners of the face have mixed signs.
    # The contour connects the two edge intersections on that face.
    
    # Simplified patch identification:
    # In a cubic cell, patches are formed by the intersection of the isosurface with the cell.
    # We need to group the edge intersections into connected components (patches).
    # This requires knowing the topology of the sign configuration.
    # The lookup table provides the decision tree for the specific sign config.
    # The structure of the patches (number of vertices, connectivity) is implicit in the sign config.
    
    # Retrieve decision tree
    decision_tree = lookup_table.get(sign_config)
    
    if decision_tree is None:
        return [] # No surface in this cell
        
    # 4. Pre-triangulation filtering
    # Generate all possible triangulations for the patch(es) defined by sign_config.
    # This is complex to implement from scratch without the specific topology map.
    # However, the decision tree is designed to work on the specific set of 
    # "remaining triangulations" after pre-triangulation.
    # For the purpose of this article, we assume the decision tree nodes 
    # reference specific edge intersections by their global index in the cell's edge list.
    
    # 5. Decision Tree Walk
    triangulation = walk_decision_tree(decision_tree, edge_intersections)
    
    return triangulation

def walk_decision_tree(node, edge_intersections):
    """
    Walks the decision tree to find the correct triangulation.
    node: The current node in the decision tree.
    edge_intersections: Dict mapping edge index to 3D point.
    Returns: List of triangles (each triangle is tuple of 3 points).
    """
    if node is None:
        return []
        
    if node.is_leaf:
        return node.triangulation
        
    # Internal node: perform 4-point test
    v1_idx, v2_idx, v3_idx, v4_idx = node.test_vertices
    
    # Get points. Note: indices refer to the edges of the cell.
    # We need to map the test indices to the actual edge indices.
    # The decision tree is built based on the specific topology of the patch.
    # For a general implementation, the decision tree nodes would store 
    # references to the specific edge intersections involved in the test.
    
    # Assuming the decision tree stores the actual Point objects or 
    # indices relative to the patch vertices.
    # Let's assume the node stores indices into a list of patch vertices.
    # But the source says "4-point test on the vertices of the patch (edge intersections)".
    
    # To make this self-contained, we assume the decision tree node 
    # provides the indices of the 4 edge intersections involved in the test.
    # These indices must correspond to the keys in edge_intersections.
    
    # Since the exact mapping from sign config to edge indices is topology-dependent,
    # we assume a helper function `get_patch_edge_indices` exists which returns
    # the list of edge indices forming the boundary of the patch for this sign config.
    # And the decision tree nodes reference indices into this list.
    
    # For the sake of the pseudocode, we assume the node stores the 
    # actual 3D points or a way to retrieve them.
    
    # Let's assume the node stores indices into the `edge_intersections` values list.
    # This is a simplification. In reality, the decision tree is built for a 
    # specific patch topology.
    
    # Re-reading source: "Pre-compute a set of 4-point tests on the vertices of the patch".
    # So the test vertices are points.
    
    p1 = node.v1
    p2 = node.v2
    p3 = node.v3
    p4 = node.v4
    
    # 4-point test
    # Vector V1->V2
    v12 = (p2[0]-p1[0], p2[1]-p1[1], p2[2]-p1[2])
    # Vector V1->V3
    v13 = (p3[0]-p1[0], p3[1]-p1[1], p3[2]-p1[2])
    # Normal
    normal = cross(v12, v13)
    # Vector V1->V4
    v14 = (p4[0]-p1[0], p4[1]-p1[1], p4[2]-p1[2])
    
    dot_product = dot(normal, v14)
    
    if dot_product > 0:
        # V4 is on front-facing side
        return walk_decision_tree(node.left, edge_intersections)
    else:
        # V4 is not on front-facing side
        return walk_decision_tree(node.right, edge_intersections)

def cross(a, b):
    return (
        a[1]*b[2] - a[2]*b[1],
        a[2]*b[0] - a[0]*b[2],
        a[0]*b[1] - a[1]*b[0]
    )

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def build_lookup_table():
    """
    Placeholder for the pre-computed lookup table construction.
    In practice, this is done offline.
    """
    return {}
```

### Parameters and Edge Cases
*   **Patch Length:** The maximum length of a patch that cannot be pre-triangulated in a cubic cell is 6 [S1].
*   **Decision Tree Depth:** The minimal depth of the decision tree is lower bounded by $\log_2 N$, where $N$ is the total number of triangulations [S1]. For patches of length 4, 5, and 6, the optimized decision tree depths are 1, 3, and 5 respectively [S1].
*   **Average Tests:** On average, 1.88 tests are required to determine the triangulation of a patch [S1].
*   **Coplanarity:** If all vertices in a 4-point test lie on the same plane, either choice of triangulation can be made [S1].
*   **Precision:** The method relies on the trilinear assumption. Precision issues may arise if edge intersections are very close to vertices or if the isovalue is exactly equal to a vertex value (sign ambiguity). [G] Standard practice is to treat $val == isovalue$ as positive (or negative) consistently to avoid degeneracy.

## Subtopics

### [4-point test](1224-4-point-test.md)
The 4-point test is a geometric filtering method used in convex contouring to identify which triangles from a set of candidate triangulations belong to the convex contour of volumetric data. It solves the ambiguity of surface configuration by enforcing the convex hull property, ensuring that all edge intersections and negative corners lie on one side of any triangle in the contour. It distinguishes itself from sibling methods like [Multi-resolution Convex Contouring](1287-multi-resolution-convex-contouring.md) by operating as a local geometric predicate on edge intersections rather than a hierarchical data structure strategy.

### [Multi-resolution Convex Contouring](1287-multi-resolution-convex-contouring.md)
Multi-resolution Convex Contouring is a view-dependent surface extraction method that accelerates rendering by contouring volumetric data at varying grid resolutions based on the viewer's distance. It extends Convex Contouring with Decision Trees by introducing transition cells to maintain topological consistency across resolution boundaries, significantly reducing triangle count and processing time while preserving visual detail. Unlike uniform contouring, it trades off some implementation complexity for performance gains, though its speed is still largely dominated by the processing of regular cells.

## Operating characteristics
*   **Time Complexity:** The algorithm is table-driven. The lookup is $O(1)$. The decision tree walk is $O(\log N)$ where $N$ is the number of triangulations for a patch. Since the maximum patch length is 6, the depth is at most 5, making it effectively constant time per cell [S1].
*   **Memory Complexity:** Requires storage for the composite look-up table. The size of the table depends on the number of sign configurations (256 for cubic cells) and the size of the decision trees [S1].
*   **Performance:** The implementation performs over 2 million point classifications per second [S1].
*   **Strengths:**
    *   Generates convex negative space, enabling fast collision detection [S1].
    *   Topologically consistent surfaces [S1].
    *   Algorithmically constructed look-up tables minimize errors compared to manual tables [S1].
    *   Fast due to pre-computed decision trees [S1].
*   **Limitations:**
    *   Requires pre-computation of decision trees for each sign configuration [S1].
    *   The method is specific to the topology of the cells (e.g., cubic). Extension to arbitrary polyhedra requires new table construction [S1].
    *   Unlike [Marching Cubes](30-isosurface-extraction.md), the triangulation depends on the magnitude of corner values (via the decision tree), not just signs [S1].

## Background
Convex Contouring with Decision Trees was introduced in the paper "Convex contouring of volumetric data" [S1]. The method was developed to address the limitations of Marching Cubes, specifically the potential for disconnected or non-convex negative space within cells, which hinders applications like collision detection [S1]. The look-up tables are constructed algorithmically based on the topology and geometry of the polyhedrons, minimizing the errors associated with manually created tables [S1]. The method has been extended to support multi-resolution contouring on nested grids [S1].

## Sources
[S1] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
