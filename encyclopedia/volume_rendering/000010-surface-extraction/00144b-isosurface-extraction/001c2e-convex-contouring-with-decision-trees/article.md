# Convex Contouring with Decision Trees

Also called: Convex Contouring, Convex contouring of volumetric data

## Overview
Convex Contouring is a table-driven isosurface extraction technique that generates topologically consistent surfaces where the negative space inside each cell is locally convex [S1]. Unlike [Marching Cubes](30-isosurface-extraction.md), which may produce disconnected or non-convex negative space within cells, Convex Contouring ensures that the convex hull of the negative space is preserved, enabling fast collision detection on the triangulated isosurface [S1]. It distinguishes itself from other cell-based methods by using pre-computed decision trees to dynamically select the correct triangulation from a set of candidates based on edge intersection positions, rather than relying on static lookup tables that ignore magnitude information [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset defined on a regular grid (typically cubic cells). Each cell contains scalar values at its 8 corners [G]. An isovalue threshold is used to determine the sign of each corner (positive or negative relative to the isovalue) [G].
*   **Output:** A triangle mesh representing the isosurface. The mesh consists of triangles whose vertices are located on the edges of the grid cells [S1].

**Data Structures:**
1.  **Sign Configuration:** An 8-bit integer representing the signs of the 8 corners of a cell [G].
2.  **Patch Data:** For each cell, the algorithm identifies "patches" on the convex contour. A convex contour may be composed of multiple connected components called patches [S1]. Each patch may contain multiple holes [S1].
3.  **Decision Tree:** A pre-computed binary tree structure stored in a lookup table [S1]. Each node in the tree represents a "4-point test" [S1]. The leaves represent the final triangulation indices [S1].
4.  **Lookup Table:** A composite table mapping each unique sign configuration to a specific decision tree and the geometry of the patches (edge intersection indices) [S1].

**Method Steps:**

1.  **Assumption of Implicit Function:** Assume the underlying implicit function is trilinear inside each cell [S1].
2.  **Determine Negative Space:** Identify all cell corners with negative values and all edge intersections where the implicit function crosses zero (isovalue). The convex hull of the negative space is the convex hull of these points [S1].
3.  **Identify Convex Contour:** The convex contour is the part of this convex hull that lies interior to the cell [S1]. It is outlined by linear convex contours on the faces of the cell [S1]. Since the linear convex contour on a 2D square face is uniquely determined by the signs of its 4 corners, neighboring 3D cells share the same contour on common faces, ensuring topological consistency [S1].
4.  **Extract Patches:** Decompose the convex contour into connected components called patches [S1]. Each patch may contain holes [S1]. The boundary of each patch is a ring of linear convex contours on the cell faces [S1].
5.  **Generate Candidate Triangulations ($S$):** Construct the set $S$ of all possible triangulations for each patch [S1]. The vertices of these triangles are restricted to the fixed edge intersections on the cell boundaries [S1]. The size of $S$ is reduced by restricting vertices to fixed cell edges, which eliminates triangulations that cannot form the convex hull of the negative space [S1].
6.  **Pre-triangulation (Filtering):** Eliminate triangulations in $S$ that contain "ineligible" triangles [S1]. A triangle is ineligible if it lies completely to the back of all non-degenerate border triangles [S1]. This reduces the search space for the decision tree [S1].
7.  **Decision Tree Lookup:**
    *   Retrieve the decision tree associated with the cell's sign configuration from the lookup table [S1].
    *   Perform a series of **4-point tests** by traversing the decision tree [S1].
    *   **4-Point Test:** Given four distinct edge intersections $V_1, V_2, V_3, V_4$:
        *   Check if $V_4$ lies on the front-facing side of the triangle $(V_1, V_2, V_3)$.
        *   If $V_4$ is on the front-facing side:
            *   The inverted triangle $(V_1, V_3, V_2)$ does not belong to the convex contour.
            *   Triangles $(V_1, V_2, V_4)$, $(V_2, V_3, V_4)$, and $(V_3, V_1, V_4)$ do not satisfy the convex hull property.
            *   Traverse the "True" branch of the decision tree.
        *   If $V_4$ is NOT on the front-facing side:
            *   Triangles $(V_1, V_2, V_3)$, $(V_1, V_4, V_2)$, $(V_2, V_4, V_3)$, and $(V_1, V_3, V_4)$ do not lie on the convex contour.
            *   Traverse the "False" branch of the decision tree.
        *   If all vertices lie on the same plane, either choice can be made [S1].
    *   Continue until a leaf node is reached, which specifies the unique triangulation for the patch [S1].
8.  **Mesh Generation:** Generate the triangles for all patches in the cell using the selected triangulations [G]. Merge vertices with adjacent cells to ensure watertightness [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def convex_contouring(volume_data, isovalue, grid_dimensions):
    """
    Main entry point for Convex Contouring.
    volume_data: 3D array of scalar values.
    isovalue: float, the threshold for surface extraction.
    grid_dimensions: tuple (nx, ny, nz), dimensions of the grid.
    Returns: List of triangles (each triangle is a tuple of 3 vertices).
    """
    triangles = []
    # Lookup table is pre-computed offline based on sign configurations [S1]
    lookup_table = build_lookup_table()
    
    nx, ny, nz = grid_dimensions
    for x in range(nx - 1):
        for y in range(ny - 1):
            for z in range(nz - 1):
                cell_triangles = process_cell(volume_data, x, y, z, isovalue, lookup_table)
                triangles.extend(cell_triangles)
    return triangles

def process_cell(volume_data, x, y, z, isovalue, lookup_table):
    """
    Process a single cubic cell.
    """
    # 1. Get corner values (standard cube ordering)
    corners = [
        volume_data[x, y, z],
        volume_data[x+1, y, z],
        volume_data[x+1, y+1, z],
        volume_data[x, y+1, z],
        volume_data[x, y, z+1],
        volume_data[x+1, y, z+1],
        volume_data[x+1, y+1, z+1],
        volume_data[x, y+1, z+1]
    ]
    
    # 2. Determine signs (1 for positive, 0 for negative)
    signs = [1 if c > isovalue else 0 for c in corners]
    sign_config = sum(s << i for i, s in enumerate(signs))
    
    # 3. Retrieve decision tree and patch info from lookup table
    tree_info = lookup_table.get(sign_config)
    if tree_info is None:
        return []
    
    decision_tree = tree_info['tree']
    patches = tree_info['patches']
    
    cell_triangles = []
    
    # 4. Calculate edge intersections via linear interpolation
    edge_indices = [
        (0, 1), (1, 2), (2, 3), (3, 0),
        (4, 5), (5, 6), (6, 7), (7, 4),
        (0, 4), (1, 5), (2, 6), (3, 7)
    ]
    
    edge_intersections = []
    for c1, c2 in edge_indices:
        v1 = corners[c1]
        v2 = corners[c2]
        if (v1 - isovalue) * (v2 - isovalue) < 0:
            t = (isovalue - v1) / (v2 - v1)
            p1 = get_vertex_position(x, y, z, c1)
            p2 = get_vertex_position(x, y, z, c2)
            intersection = lerp(p1, p2, t)
            edge_intersections.append(intersection)
        else:
            edge_intersections.append(None)
            
    # 5. Process each patch
    for patch in patches:
        patch_vertices = [edge_intersections[idx] for idx in patch['boundary']]
        patch_vertices = [v for v in patch_vertices if v is not None]
        
        if not patch_vertices:
            continue
            
        # 6. Perform 4-point tests to select triangulation
        triangulation_index = traverse_decision_tree(decision_tree, patch_vertices)
        
        # 7. Generate triangles for this triangulation
        tri_indices = patch['triangulations'][triangulation_index]
        for tri in tri_indices:
            v1 = patch_vertices[tri[0]]
            v2 = patch_vertices[tri[1]]
            v3 = patch_vertices[tri[2]]
            cell_triangles.append((v1, v2, v3))
            
    return cell_triangles

def traverse_decision_tree(tree, vertices):
    """
    Traverse the decision tree using 4-point tests.
    tree: Nested dictionary representing the decision tree.
          Structure: {'test': (v1_idx, v2_idx, v3_idx, v4_idx), 'true': ..., 'false': ...}
          Or leaf: {'result': index}
    vertices: List of 3D points (edge intersections).
    Returns: Index of the selected triangulation.
    """
    node = tree
    while 'test' in node:
        v1_idx, v2_idx, v3_idx, v4_idx = node['test']
        v1 = vertices[v1_idx]
        v2 = vertices[v2_idx]
        v3 = vertices[v3_idx]
        v4 = vertices[v4_idx]
        
        # Check if V4 lies on the front-facing side of triangle (V1, V2, V3)
        n = cross_product(subtract(v2, v1), subtract(v3, v1))
        vec = subtract(v4, v1)
        dot = dot_product(n, vec)
        
        if dot > 0:
            node = node['true']
        else:
            node = node['false']
            
    return node['result']

def build_lookup_table():
    """
    Pre-compute the lookup table offline.
    Associates each sign configuration with a compact decision tree [S1].
    """
    table = {}
    for sign_config in range(256):
        # Patches are detected automatically from corner signs [S1]
        patches = identify_patches(sign_config)
        
        if not patches:
            table[sign_config] = None
            continue
            
        # Decision trees are pre-computed to ensure minimal tests [S1]
        decision_tree = build_optimal_decision_tree(patches)
        
        table[sign_config] = {
            'tree': decision_tree,
            'patches': patches
        }
    return table

def identify_patches(sign_config):
    """
    Detect patches and their boundary edge intersections from corner signs.
    Implemented as a table-driven lookup based on topology [S1].
    """
    # Returns list of patches, each containing boundary indices and candidate triangulations
    pass # Offline construction based on sign configuration topology [S1]

def build_optimal_decision_tree(patches):
    """
    Construct a decision tree with minimal depth for a patch.
    Uses a search algorithm to find the optimal set of 4-point tests [S1].
    """
    # Offline optimization: minimizes depth bounded by log2(N) where N is triangulations [S1]
    pass # Pre-computed offline to ensure minimal number of tests [S1]

def get_vertex_position(x, y, z, corner_idx):
    """
    Convert corner index to 3D coordinates relative to cell origin.
    """
    coords = [
        (0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0),
        (0, 0, 1), (1, 0, 1), (1, 1, 1), (0, 1, 1)
    ]
    cx, cy, cz = coords[corner_idx]
    return (x + cx, y + cy, z + cz)

def lerp(p1, p2, t):
    return (p1[0] + t*(p2[0]-p1[0]), p1[1] + t*(p2[1]-p1[1]), p1[2] + t*(p2[2]-p1[2]))

def subtract(p1, p2):
    return (p1[0]-p2[0], p1[1]-p2[1], p1[2]-p2[2])

def cross_product(v1, v2):
    return (
        v1[1]*v2[2] - v1[2]*v2[1],
        v1[2]*v2[0] - v1[0]*v2[2],
        v1[0]*v2[1] - v1[1]*v2[0]
    )

def dot_product(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
```

**Parameters:**
*   **Isovalue:** The threshold scalar value [G].
*   **Cell Geometry:** The method assumes cubic cells, but can be extended to arbitrary convex polyhedra [S1].
*   **Decision Tree Depth:** The maximum depth for any patch in a cubic cell is 5 [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** If all vertices of a 4-point test lie on the same plane, either branch of the decision tree can be taken [S1].
*   **Precision:** Linear interpolation assumes trilinearity. If the implicit function is not well-approximated by trilinear interpolation, the edge intersections may be inaccurate, potentially affecting the convex hull property [S1].
*   **Boundary Conditions:** The method relies on shared faces between cells. If the grid is not regular or if cells are not properly aligned, topological consistency may break [S1].

## Subtopics

### [4-point test](1224-4-point-test.md)
A geometric filtering method used in convex contouring to identify which triangles from a set of candidate triangulations belong to the convex contour of volumetric data. It solves the ambiguity of surface configuration by enforcing the convex hull property, ensuring that all edge intersections and negative corners lie on one side of any triangle in the contour. It distinguishes itself from sibling methods like [Multi-resolution Convex Contouring](1287-multi-resolution-convex-contouring.md) by operating as a local geometric predicate on edge intersections rather than a hierarchical data structure strategy.

### [Multi-resolution Convex Contouring](1287-multi-resolution-convex-contouring.md)
A view-dependent surface extraction method that accelerates rendering by contouring volumetric data at varying grid resolutions based on the viewer's distance. It extends Convex Contouring with Decision Trees by introducing transition cells to maintain topological consistency across resolution boundaries, significantly reducing triangle count and processing time while preserving visual detail. Unlike uniform contouring, it trades off some implementation complexity for performance gains, though its speed is still largely dominated by the processing of regular cells.

## Operating characteristics
*   **Time Complexity:** The algorithm is table-driven, making the per-cell processing time constant on average [S1]. The decision tree traversal is logarithmic with respect to the number of candidate triangulations, with a maximum depth of 5 for cubic cells [S1]. On average, only 1.88 tests are required to determine the triangulation of a patch [S1].
*   **Memory Complexity:** The memory usage is dominated by the lookup table, which stores decision trees for all 256 sign configurations [S1]. The lookup table stores a compact decision tree for each sign configuration [S1].
*   **Measured Performance:** The implementation performs over 2 million point classifications per second [S1].
*   **Strengths:**
    *   Generates convex negative space inside each cell, enabling fast collision detection [S1].
    *   Topologically consistent surfaces without manual table creation [S1].
    *   Fast execution due to pre-computed decision trees [S1].
*   **Limitations:**
    *   Requires pre-computation of lookup tables, which may be complex to implement [S1].
    *   Performance is dominated by regular cell processing in multi-resolution variants [S1].
*   **When to Use:** Suitable for applications requiring fast collision detection and real-time rendering of isosurfaces, such as medical visualization or scientific data exploration [S1].
*   **When Not to Use:** Not suitable for non-convex negative spaces or when the trilinear assumption is invalid [S1].

## Background
Convex Contouring was introduced in a paper titled "Convex contouring of volumetric data" [S1]. The method was developed to address the limitations of Marching Cubes, particularly the lack of convexity in the negative space and the susceptibility of manually created lookup tables to errors [S1]. The algorithm uses an automatically constructed look-up table that stores compact decision trees by sign configurations, allowing for dynamic triangulation of isosurface patches [S1]. The technique has been extended to multi-resolution contouring and arbitrary convex, space-filling polyhedra [S1].

## Sources
[S1] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
