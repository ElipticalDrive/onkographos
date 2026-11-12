# Convex Contouring with Decision Trees

Also called: Convex Contouring, Convex contouring of volumetric data

## Overview
Convex Contouring is a table-driven isosurface extraction technique that generates topologically consistent surfaces with a local convexity property, ensuring that the negative space inside each cell is convex [S1]. Unlike Marching Cubes, which may produce disconnected non-convex negative space, this method preserves convexity, enabling fast collision detection on the triangulated isosurface [S1]. It differs from other cell-based algorithms by using pre-computed decision trees to dynamically select triangulations based on corner values, minimizing errors through algorithmic table construction rather than manual entry [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric dataset defined on a regular grid (typically cubic cells). Each cell contains scalar values at its vertices. The underlying implicit function is assumed to be trilinear within each cell [S1].
*   **Output:** A triangulated isosurface mesh.

**Data Structures**
*   **Look-up Table:** A composite table associating each sign configuration (the signs of the scalar values at the 8 corners of a cell relative to the isovalue) with a compact decision tree [S1].
*   **Decision Tree:** A pre-computed structure where each node represents a "4-point test" and edges represent the outcome (front-facing or back-facing). The leaves represent the selected triangulation [S1].
*   **Patch:** A connected component of the convex contour within a cell. Patches may contain holes [S1].

**Method Steps**

1.  **Sign Configuration Determination:** For each cell, determine the sign configuration by comparing the scalar values at the 8 vertices to the isovalue. This yields an index into the look-up table [S1].

2.  **Boundary Detection:** Identify the linear convex contours on the faces of the cell. In 2D, a linear convex contour on a square face is uniquely determined by the signs at the 4 corners. These 2D contours form the boundary of the 3D convex contour [S1].
    *   The convex contour inside the cell is the part of the convex hull of the negative space (negative corners and edge intersections) that lies interior to the cell [S1].
    *   Identify patches (connected components of the contour) and holes. Each hole is surrounded by a ring of linear convex contours [S1].

3.  **Pre-triangulation (Filtering):** Construct the set $S$ of all possible triangulations for each patch. The vertices of these triangles are restricted to fixed cell edge intersections.
    *   Eliminate triangulations containing "ineligible" triangles. A triangle is ineligible if it lies completely to the back of all non-degenerate border triangles formed by the edges [S1].
    *   This reduces the search space $N$ (the number of remaining triangulations) significantly. For example, for some sign configurations, $N$ is reduced to 1 or 4 [S1].

4.  **Decision Tree Lookup:** Retrieve the pre-computed decision tree from the look-up table corresponding to the sign configuration [S1].
    *   The decision tree is optimized to have minimal depth. The minimal depth is lower bounded by $\log_2 N$, where $N$ is the number of remaining triangulations after pre-triangulation [S1].
    *   Walk down the decision tree using 4-point tests to select the correct triangulation from $S$ [S1].

5.  **4-Point Test:** This is the core geometric test used in the decision tree nodes. Given four distinct edge intersections $V_1, V_2, V_3, V_4$:
    *   Assume triangles on the convex contour face inside the convex hull of the negative space.
    *   Test if $V_4$ lies on the front-facing side of the triangle $(V_1, V_2, V_3)$.
    *   **Case A (Front-facing):** If $V_4$ is on the front-facing side of $(V_1, V_2, V_3)$:
        *   The inverted triangle $(V_1, V_3, V_2)$ does not belong to the convex contour.
        *   Triangles $(V_1, V_2, V_4)$, $(V_2, V_3, V_4)$, and $(V_3, V_1, V_4)$ do not satisfy the convex hull property.
        *   These triangulations are eliminated [S1].
    *   **Case B (Back-facing or Coplanar):** If $V_4$ does not lie on the front-facing side of $(V_1, V_2, V_3)$:
        *   Triangles $(V_1, V_2, V_3)$, $(V_1, V_4, V_2)$, $(V_2, V_4, V_3)$, and $(V_1, V_3, V_4)$ do not lie on the convex contour.
        *   These triangulations are eliminated.
        *   If all vertices lie on the same plane, either choice can be made [S1].

6.  **Mesh Generation:** The leaf of the decision tree reached after the tests specifies the final set of triangles for the patch. Generate the triangles using the edge intersection coordinates [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convex_contouring(grid, isovalue):
    """
    Extracts isosurface using Convex Contouring with Decision Trees.
    
    Args:
        grid: 3D array of scalar values.
        isovalue: The threshold value for the isosurface.
        
    Returns:
        List of triangles (each triangle is a tuple of 3 vertex coordinates).
    """
    triangles = []
    # Pre-computed look-up table mapping sign_config -> decision_tree
    # decision_tree is a list of nodes, each node is (v1, v2, v3, v4, left_child, right_child)
    # or a leaf index.
    lookup_table = precompute_lookup_table() 
    
    for each cell in grid:
        # 1. Determine sign configuration
        signs = []
        for vertex in cell.vertices:
            if grid[vertex] > isovalue:
                signs.append(1)
            else:
                signs.append(0)
        sign_config = tuple(signs)
        
        # 2. Get decision tree for this configuration
        decision_tree = lookup_table[sign_config]
        
        # 3. Identify edge intersections and patches
        # Calculate exact positions of edge intersections using trilinear interpolation
        edge_intersections = compute_edge_intersections(cell, isovalue)
        
        # Identify patches and their boundary vertices based on signs
        # This step determines the set of candidate triangulations S
        patches = identify_patches(cell, signs, edge_intersections)
        
        for patch in patches:
            # 4. Pre-triangulation: Filter set S
            # Identify ineligible triangles based on border triangles
            candidate_triangulations = get_all_triangulations(patch)
            valid_triangulations = []
            
            for tri_set in candidate_triangulations:
                if is_valid_pre_triangulation(tri_set, patch.border_triangles):
                    valid_triangulations.append(tri_set)
            
            # If only one triangulation remains, use it directly
            if len(valid_triangulations) == 1:
                triangles.extend(valid_triangulations[0])
                continue
            
            # 5. Use decision tree to select triangulation
            # The decision tree nodes are 4-point tests
            selected_tri_set = traverse_decision_tree(
                decision_tree, 
                edge_intersections, 
                valid_triangulations
            )
            triangles.extend(selected_tri_set)
            
    return triangles

def is_valid_pre_triangulation(tri_set, border_triangles):
    """
    Checks if any triangle in tri_set lies completely to the back 
    of all non-degenerate border triangles.
    """
    for triangle in tri_set:
        if is_ineligible(triangle, border_triangles):
            return False
    return True

def is_ineligible(triangle, border_triangles):
    """
    A triangle is ineligible if it lies completely to the back 
    of all non-degenerate border triangles.
    """
    for border in border_triangles:
        if not border.is_degenerate():
            if triangle.is_behind(border):
                return True
    return False

def traverse_decision_tree(node, vertices, valid_triangulations):
    """
    Walks the decision tree to find the correct triangulation.
    """
    if node.is_leaf():
        return valid_triangulations[node.index]
    
    v1, v2, v3, v4 = node.vertices
    # Perform 4-point test
    # Check if v4 is on the front-facing side of (v1, v2, v3)
    if is_front_facing(v1, v2, v3, v4):
        # Case A: Eliminate triangles containing (v1, v3, v2), (v1, v2, v4), (v2, v3, v4), (v3, v1, v4)
        filtered = filter_triangulations(valid_triangulations, 
                                        excluded=[(v1,v3,v2), (v1,v2,v4), (v2,v3,v4), (v3,v1,v4)])
        return traverse_decision_tree(node.left_child, vertices, filtered)
    else:
        # Case B: Eliminate triangles containing (v1, v2, v3), (v1, v4, v2), (v2, v4, v3), (v1, v3, v4)
        filtered = filter_triangulations(valid_triangulations,
                                        excluded=[(v1,v2,v3), (v1,v4,v2), (v2,v4,v3), (v1,v3,v4)])
        return traverse_decision_tree(node.right_child, vertices, filtered)

def is_front_facing(v1, v2, v3, v4):
    """
    Returns True if v4 lies on the front-facing side of triangle (v1, v2, v3).
    Assumes consistent winding order for front-facing.
    """
    # Compute normal of triangle (v1, v2, v3)
    normal = cross_product(subtract(v2, v1), subtract(v3, v1))
    # Vector from v1 to v4
    vec = subtract(v4, v1)
    # Dot product
    dot = dot_product(normal, vec)
    # If dot > 0, v4 is on the front side (assuming outward normal)
    return dot > 0
```

**Parameters and Typical Values**
*   **Maximum Patch Length:** The maximum length of a patch in a cubic cell that cannot be pre-triangulated is 6 [S1].
*   **Decision Tree Depth:** For patches of length 4, 5, and 6, the optimized decision tree depths are 1, 3, and 5, respectively [S1].
*   **Average Tests:** On average, 1.88 tests are required to determine the triangulation of a patch [S1].
*   **Performance:** The implementation performs over 2 million point classifications per second [S1].

**Edge Cases and Pitfalls**
*   **Degenerate Input:** If all vertices in a 4-point test lie on the same plane, either choice of triangulation elimination can be made [S1].
*   **Precision:** The method assumes trilinear interpolation. Precision errors in edge intersection calculation may affect the "front-facing" test. [G] Ensure robust geometric predicates for the 4-point test to handle near-coplanar cases.
*   **Boundaries:** The method relies on the uniqueness of linear convex contours on 2D faces to ensure topological consistency between neighboring cells. This holds for regular grids. [S1]

## Subtopics

### [4-point test](1224-4-point-test.md)
A geometric filtering method used to identify which triangles from a set of candidate triangulations belong to the convex contour. It enforces the convex hull property, ensuring all edge intersections and negative corners lie on one side of any triangle in the contour.

### [Multi-resolution Convex Contouring](1287-multi-resolution-convex-contouring.md)
A view-dependent surface extraction method that accelerates rendering by contouring volumetric data at varying grid resolutions based on the viewer's distance. It uses transition cells to maintain topological consistency across resolution boundaries, reducing triangle count and processing time.

### [Cluster Connection Tree Table](11800-cluster-connection-tree-table.md)
A data structure that summarizes topological connections between clusters of isosurface points across adjacent slices. It explicitly tracks connections (Head and Tail IDs) to enable the construction of topologically consistent surfaces with local convexity, handling non-convex objects through branching and merging decomposition.

## Operating characteristics
*   **Time Complexity:** The method is table-driven, making the per-cell processing time constant on average. The decision tree depth is logarithmic with respect to the number of triangulations ($\log_2 N$). On average, only 1.88 tests are needed per patch [S1].
*   **Memory Complexity:** Requires storage for the look-up table containing decision trees for all sign configurations. The size of the table depends on the maximum patch complexity (max length 6) [S1].
*   **Measured Performance:** The implementation performs over 2 million point classifications per second [S1].
*   **Strengths:**
    *   Generates convex negative space inside each cell, enabling fast collision detection [S1].
    *   Topologically consistent surfaces due to unique 2D face contours [S1].
    *   Algorithmic table construction minimizes errors compared to manual tables (e.g., Marching Cubes) [S1].
    *   Fast, crack-free multi-resolution contouring is possible [S1].
*   **Limitations:**
    *   Performance is dominated by the processing of regular cells in multi-resolution modes [S1].
    *   Requires pre-computation of decision trees, which adds complexity to the implementation [S1].
*   **When to Use:** When fast collision detection is required, or when convexity of the negative space is important for downstream applications. Also suitable for real-time rendering with multi-resolution grids.
*   **When Not to Use:** When the underlying data does not support trilinear interpolation assumptions, or when non-convex features within cells are critical and cannot be approximated by the convex hull.

## Background
Convex Contouring was introduced as a fast, table-driven isosurface extraction technique for volumetric data [S1]. It was developed to address the limitations of Marching Cubes, particularly the potential for disconnected non-convex negative space and the susceptibility of manually created look-up tables to errors [S1]. The method was extended to support multi-resolution contouring on nested grids, allowing for view-dependent rendering without surface cracks [S1]. The technique can also be extended to extract isosurfaces on arbitrary convex, space-filling polyhedra [S1].

## Sources
[S1] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
