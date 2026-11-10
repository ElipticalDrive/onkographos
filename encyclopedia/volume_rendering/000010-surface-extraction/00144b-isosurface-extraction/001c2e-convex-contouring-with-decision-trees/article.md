# Convex Contouring with Decision Trees

Also called: Convex Contouring, Convex contouring of volumetric data

## Overview
Convex Contouring is a table-driven isosurface extraction technique that generates topologically consistent surfaces with a local convexity property, ensuring that the negative space inside each cell is convex [S1]. Unlike Marching Cubes, which may produce disconnected non-convex negative space, this method preserves convexity, enabling fast collision detection on the triangulated isosurface [S1]. It differs from other cell-based algorithms by using a composite look-up table that associates each sign configuration with a compact decision tree for dynamic triangulation, minimizing errors through algorithmic table construction rather than manual entry [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset defined on a regular grid (cubic cells). Each cell contains scalar values at its 8 corners. An isovalue threshold is used to determine the sign of each corner (positive or negative relative to the isovalue).
*   **Output:** A triangle mesh representing the isosurface.

**Data Structures:**
1.  **Sign Configuration:** A 8-bit integer representing the signs of the 8 corners of a cubic cell.
2.  **Look-up Table (LUT):** A pre-computed table indexed by sign configuration. Each entry contains:
    *   Patch boundaries: Lists of edge intersections forming the boundary of each convex contour patch.
    *   Decision Tree: A structure for dynamic triangulation.
3.  **Decision Tree Node:**
    *   `test_id`: Identifier for the 4-point test (indices of vertices V1, V2, V3, V4).
    *   `result_true`: Index of the child node if the test condition is met.
    *   `result_false`: Index of the child node if the test condition is not met.
    *   `triangulation_id`: Leaf node identifier pointing to the final set of triangles.

**Algorithm Steps:**

1.  **Assumption:** The underlying implicit function is trilinear inside each cell [S1].
2.  **Convex Hull Definition:** The convex hull of the negative space in a cell is the convex hull of all negative cell corners and edge intersections [S1]. The convex contour is the part of this convex hull that lies interior to the cell [S1].
3.  **Boundary Determination:** The convex contour is outlined by linear convex contours on the faces of the cell. Since the linear convex contour in a 2D square is uniquely determined by the signs at the 4 corners, neighboring 3D cells share the same linear contour on common faces, ensuring topological consistency [S1].
4.  **Patch Identification:** A convex contour may consist of multiple connected components called patches. Each patch may contain holes. Each hole is surrounded by a ring of linear convex contours detectable by the signs at the corners [S1].
5.  **Triangulation Set Construction:** Construct a set $S$ of all possible triangulations for each patch. The vertices of these triangles are restricted to fixed cell edges, reducing the size of $S$ [S1].
6.  **Pre-triangulation (Filtering):** Eliminate triangulations containing ineligible triangles. A triangle is ineligible if it lies completely to the back of all non-degenerate border triangles. This reduces the search space for remaining triangulations [S1].
7.  **Decision Tree Pre-computation:**
    *   For the remaining triangulations, pre-compute a set of 4-point tests on the vertices of the patch (edge intersections) to distinguish between them [S1].
    *   Organize these tests into a decision tree structure.
    *   Use a search algorithm to find the optimal set of tests that produces a decision tree with the smallest depth. The minimal depth is lower bounded by $\log_2 N$, where $N$ is the total number of triangulations [S1].
    *   Store the resulting decision tree in the LUT entry for the corresponding sign configuration [S1].
8.  **Runtime Processing (Per Cell):**
    *   Determine the sign configuration of the cell.
    *   Retrieve the LUT entry for this configuration.
    *   Identify the patches and their boundary edge intersections.
    *   For each patch, traverse the pre-computed decision tree using 4-point tests to select the correct triangulation from the reduced set $S$ [S1].
    *   Generate triangles for the patch and the faces filling the negative regions on cell faces [S1].

**The 4-Point Test:**
Given four distinct edge intersections $V_1, V_2, V_3, V_4$:
*   If $V_4$ lies on the front-facing side of the triangle $(V_1, V_2, V_3)$, then:
    *   The inverted triangle $(V_1, V_3, V_2)$ does not belong to the convex contour.
    *   Triangles $(V_1, V_2, V_4)$, $(V_2, V_3, V_4)$, and $(V_3, V_1, V_4)$ do not satisfy the convex hull property [S1].
*   If $V_4$ does not lie on the front-facing side of $(V_1, V_2, V_3)$, then:
    *   Triangles $(V_1, V_2, V_3)$, $(V_1, V_4, V_2)$, $(V_2, V_4, V_3)$, and $(V_1, V_3, V_4)$ do not lie on the convex contour [S1].
*   If all vertices lie on the same plane, either choice can be made [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_convex_contour(cell_values, isovalue):
    """
    Extracts the convex contour for a single cubic cell.
    
    Args:
        cell_values: List of 8 scalar values at the cell corners.
        isovalue: The threshold value for the isosurface.
        
    Returns:
        List of triangles (each triangle is a tuple of 3 vertices).
    """
    # 1. Determine signs
    # Corner indices: 0-7. Sign is 1 if value > isovalue, else 0.
    # Note: The paper defines "negative space" as the space where value < isovalue.
    # We track which corners are "negative" (inside the object) or "positive" (outside).
    # Let's define sign_bit = 1 if cell_values[i] < isovalue (negative), else 0.
    # However, standard Marching Cubes often uses 1 for positive. 
    # The paper states: "convex hull of the negative space... convex hull of all negative cell corners".
    # Let's assume we identify the set of negative corners.
    
    negative_corners = []
    edge_intersections = {} # Map edge_index -> intersection_point
    
    # Define edges of a cube (12 edges)
    # Edge indices 0-11. Each edge connects two corners.
    edges = [
        (0, 1), (1, 3), (3, 2), (2, 0), # Bottom face
        (4, 5), (5, 7), (7, 6), (6, 4), # Top face
        (0, 4), (1, 5), (3, 7), (2, 6)  # Vertical edges
    ]
    
    # Identify negative corners and compute edge intersections
    sign_config = 0
    for i in range(8):
        if cell_values[i] < isovalue:
            negative_corners.append(i)
            sign_config |= (1 << i)
        else:
            sign_config |= (0 << i)
            
    # Compute edge intersections
    for edge_idx, (c1, c2) in enumerate(edges):
        v1 = cell_values[c1]
        v2 = cell_values[c2]
        
        # Check if the edge crosses the isovalue
        if (v1 < isovalue) != (v2 < isovalue):
            # Linear interpolation
            t = (isovalue - v1) / (v2 - v1)
            # Assuming unit cube, corners at integer coordinates
            p1 = get_corner_position(c1)
            p2 = get_corner_position(c2)
            intersection = p1 + t * (p2 - p1)
            edge_intersections[edge_idx] = intersection
            
    # 2. Retrieve LUT entry
    # The LUT is pre-computed. We assume it's available globally.
    lut_entry = lookup_lut(sign_config)
    
    # 3. Identify patches
    patches = lut_entry['patches']
    
    triangles = []
    
    # 4. Triangulate each patch
    for patch in patches:
        # Patch contains a list of edge indices forming the boundary
        boundary_edge_indices = patch['boundary_edges']
        
        # Get the vertices for this patch (edge intersections)
        patch_vertices = [edge_intersections[e] for e in boundary_edge_indices]
        
        # Get the decision tree for this sign config
        decision_tree = lut_entry['decision_tree']
        
        # Traverse decision tree to find triangulation
        triangulation_id = traverse_decision_tree(decision_tree, patch_vertices)
        
        # Get the triangles for this triangulation
        patch_triangles = get_triangles_for_triangulation(triangulation_id, patch_vertices)
        triangles.extend(patch_triangles)
        
    # 5. Fill negative regions on cell faces
    # The convex hull includes triangles filling the negative regions on cell faces.
    # These are determined by the linear convex contours on the faces.
    face_triangles = fill_face_negative_regions(cell_values, isovalue, negative_corners)
    triangles.extend(face_triangles)
    
    return triangles

def traverse_decision_tree(node, vertices):
    """
    Traverses the decision tree to determine the correct triangulation.
    
    Args:
        node: Current node in the decision tree.
        vertices: List of vertex positions for the patch.
        
    Returns:
        Triangulation ID.
    """
    while not node.is_leaf():
        # Perform 4-point test
        # node.test_id contains indices V1, V2, V3, V4
        v1 = vertices[node.test_id.V1]
        v2 = vertices[node.test_id.V2]
        v3 = vertices[node.test_id.V3]
        v4 = vertices[node.test_id.V4]
        
        # Check if V4 is on the front-facing side of (V1, V2, V3)
        # Front-facing means V4 is on the side of the normal (V2-V1) x (V3-V1)
        normal = cross_product(v2 - v1, v3 - v1)
        vector_v1_v4 = v4 - v1
        dot_product = dot_product(normal, vector_v1_v4)
        
        if dot_product > 0: # V4 is on the front-facing side
            node = node.result_true
        else:
            node = node.result_false
            
    return node.triangulation_id

def get_corner_position(corner_index):
    """
    Returns the 3D position of a corner index in a unit cube.
    """
    x = corner_index & 1
    y = (corner_index >> 1) & 1
    z = (corner_index >> 2) & 1
    return Vector3(x, y, z)

def fill_face_negative_regions(cell_values, isovalue, negative_corners):
    """
    Fills the negative regions on the cell faces with triangles.
    This is based on the linear convex contours on the 2D faces.
    """
    triangles = []
    # There are 6 faces. Each face is a square with 4 corners.
    faces = [
        [0, 1, 3, 2], # Bottom (z=0)
        [4, 5, 7, 6], # Top (z=1)
        [0, 1, 5, 4], # Front (y=0)
        [2, 3, 7, 6], # Back (y=1)
        [0, 2, 6, 4], # Left (x=0)
        [1, 3, 7, 5]  # Right (x=1)
    ]
    
    for face_corners in faces:
        # Determine signs of the 4 corners of this face
        face_signs = [cell_values[c] < isovalue for c in face_corners]
        
        # Determine the linear convex contour on this 2D square
        # The linear convex contour is uniquely determined by the signs.
        # It consists of line segments connecting edge intersections.
        # We need to identify which edges have intersections.
        face_edges = [
            (0, 1), (1, 3), (3, 2), (2, 0)
        ]
        
        edge_intersections_face = []
        for i, (c1, c2) in enumerate(face_edges):
            if face_signs[c1] != face_signs[c2]:
                # Compute intersection
                v1 = cell_values[face_corners[c1]]
                v2 = cell_values[face_corners[c2]]
                t = (isovalue - v1) / (v2 - v1)
                p1 = get_corner_position(face_corners[c1])
                p2 = get_corner_position(face_corners[c2])
                intersection = p1 + t * (p2 - p1)
                edge_intersections_face.append((i, intersection))
                
        # The linear convex contour on a 2D square is a set of segments.
        # If there are 2 intersections, they form a segment.
        # If there are 4 intersections, they form a polygon (usually a quad or two triangles).
        # The negative space on the face is filled by triangles.
        
        # Simplified logic: The negative space on the face is the convex hull of the negative corners and edge intersections.
        # Since it's 2D, we can just triangulate the polygon formed by the negative region.
        
        # Identify negative points on this face
        negative_points = []
        for i, c in enumerate(face_corners):
            if face_signs[c]:
                negative_points.append(get_corner_position(c))
                
        for idx, pt in edge_intersections_face:
            # Check if this edge is part of the negative boundary
            # An edge is part of the negative boundary if one corner is negative and the other is positive
            c1, c2 = face_edges[idx]
            if face_signs[c1] != face_signs[c2]:
                negative_points.append(pt)
                
        # Triangulate the polygon of negative points on the face
        # Since it's convex, we can fan triangulate from the first point
        if len(negative_points) >= 3:
            for i in range(1, len(negative_points) - 1):
                triangles.append((negative_points[0], negative_points[i], negative_points[i+1]))
                
    return triangles
```

**Parameters:**
*   **Maximum Patch Length:** The maximum length of a patch in a cubic cell that cannot be pre-triangulated is 6 [S1].
*   **Decision Tree Depth:** For a patch of length 4, 5, and 6, the optimized decision tree depths are 1, 3, and 5 respectively [S1].
*   **Average Tests:** On average, 1.88 tests are required to determine the triangulation of a patch [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** If all vertices lie on the same plane in the 4-point test, either choice of triangulation can be made [S1].
*   **Precision:** The algorithm assumes trilinear interpolation within cells. Precision issues may arise if the isovalue is very close to a corner value, leading to ambiguous signs. [G] A small epsilon should be used when comparing values to the isovalue to handle floating-point inaccuracies.
*   **Boundary Conditions:** The method relies on topological consistency between neighboring cells. If the grid is not regular or if cells are missing, the shared face contours may not match, leading to cracks. [G] Ensure that the grid traversal handles boundaries correctly, possibly by padding the volume or handling boundary cells specially.

## Subtopics

### [4-point test](1224-4-point-test.md)
The 4-point test is a geometric filtering method used in convex contouring to identify which triangles from a set of candidate triangulations belong to the convex contour of volumetric data. It solves the ambiguity of surface configuration by enforcing the convex hull property, ensuring that all edge intersections and negative corners lie on one side of any triangle in the contour.

### [Multi-resolution Convex Contouring](1287-multi-resolution-convex-contouring.md)
Multi-resolution Convex Contouring is a view-dependent surface extraction method that accelerates rendering by contouring volumetric data at varying grid resolutions based on the viewer's distance. It extends Convex Contouring with Decision Trees by introducing transition cells to maintain topological consistency across resolution boundaries, significantly reducing triangle count and processing time while preserving visual detail. Unlike uniform contouring, it trades off some implementation complexity for performance gains, though its speed is still largely dominated by the processing of regular cells.

### [Cluster Connection Tree Table](11800-cluster-connection-tree-table.md)
The Cluster Connection Tree Table is a data structure used in convex contouring to summarize the topological connections between clusters of isosurface points across adjacent slices. It serves as the bridge between 2D slice processing and 3D surface construction, explicitly tracking which clusters in slice $i$ connect to which clusters in slice $i+1$. Unlike methods that infer connectivity implicitly, this table stores explicit bidirectional links (Head and Tail IDs) and a correlation metric, enabling the construction of topologically consistent surfaces with local convexity properties. It distinguishes itself by handling non-convex objects through multiple connection handling, where branching and merging are decomposed into 1:1 connections to prevent surface smoothing or duplication.

## Operating characteristics
*   **Time Complexity:** The algorithm is table-driven, with constant-time look-up for sign configurations. The decision tree traversal is logarithmic with respect to the number of triangulations, with a maximum depth of 5 for cubic cells [S1].
*   **Memory Complexity:** The look-up table size depends on the number of sign configurations (256 for cubic cells) and the complexity of the decision trees. The decision trees are pre-computed and stored, requiring additional memory proportional to the number of patches and their triangulation sets [S1].
*   **Measured Performance:** The implementation performs over 2 million point classifications per second [S1].
*   **Strengths:**
    *   Fast collision detection due to convex negative space [S1].
    *   Topologically consistent surfaces [S1].
    *   Algorithmic table construction minimizes errors compared to manual tables [S1].
    *   Supports multi-resolution contouring without cracks [S1].
*   **Limitations:**
    *   Limited to convex cells (cubic, or arbitrary convex space-filling polyhedra) [S1].
    *   Decision tree pre-computation can be complex for high-dimensional cells [S1].
*   **When to Use:** Suitable for applications requiring fast collision detection and topologically consistent surfaces, such as real-time visualization and simulation.
*   **When Not to Use:** Not suitable for non-convex cells or unstructured grids without modification.

## Background
Convex Contouring was introduced in a paper titled "Convex contouring of volumetric data" [S1]. The method addresses the limitations of Marching Cubes, particularly the potential for disconnected non-convex negative space and the susceptibility of manually created look-up tables to errors [S1]. It builds on the principles of isosurface extraction, focusing on generating convex negative space within individual cells [S1]. The technique has been extended to multi-resolution contouring and arbitrary convex polyhedra [S1].

## Sources
[S1] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
