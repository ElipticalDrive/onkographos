# Pre-triangulation of Convex Contour
Also called: Pre-triangulation

## Overview
Pre-triangulation of Convex Contour is a method used during the construction of lookup tables for isosurface extraction to eliminate triangulations that cannot appear on the convex contour of the volumetric data. It solves the problem of combinatorial explosion in possible surface configurations by using geometric constraints derived from the convex hull property. Unlike the [4-point test](1224-4-point-test.md), which is used at runtime to select among remaining candidates, this method statically prunes the search space during table generation based solely on the sign configuration of the cell corners.

## How it works
The method identifies triangles that are geometrically impossible to lie on the convex hull of the negative space. It relies on the observation that for a triangle on the convex contour, the half-space on its front-facing side must contain (or partially contain) every other cell edge exhibiting a sign change.

**Inputs and Outputs:**
- **Input:** A cell sign configuration (binary mask of 8 corners).
- **Output:** A subset of valid border triangles for a given patch topology, or a flag indicating a triangle is ruled out.

**Data Structures:**
- **Cell Corners ($C_i$):** The 8 vertices of the voxel, indexed $i=1 \dots 8$.
- **Cell Edges:** Defined by pairs of corners. A sign change occurs on an edge if the scalar values at its endpoints have different signs relative to the isovalue.
- **Border Triangles:** Triangles formed by selecting one endpoint from each of three distinct cell edges that intersect the isosurface.

**Steps:**
1. **Identify Intersecting Edges:** Determine all cell edges that exhibit a sign change. Let these edges be $E_1, E_2, \dots, E_k$.
2. **Select Candidate Triangles:** Consider a triangle $T$ formed by three vertices $v_1, v_2, v_3$, where each $v_j$ lies on a distinct intersecting edge $E_{a}, E_{b}, E_{c}$.
3. **Construct Border Triangles:** For the three edges $E_a=(C_1, C_2)$, $E_b=(C_3, C_4)$, and $E_c=(C_5, C_6)$, construct the "border triangles" by taking the endpoints of these edges in order. Specifically, form triangles such as $(C_1, C_3, C_5)$, $(C_1, C_3, C_6)$, $(C_1, C_4, C_5)$, etc., covering all combinations of endpoints from the three edges.
4. **Check Back-Facing Condition:** For each border triangle $T_{border}$, determine its orientation (front/back). Check if any other intersecting edge $E_{other}$ (distinct from the three defining $T$) lies completely to the back of *all* non-degenerate border triangles formed by the three edges.
   - An edge lies "completely to the back" if all points on the edge are on the negative side of the plane defined by the border triangle.
5. **Prune:** If such an edge $E_{other}$ exists, then any triangle $T$ whose vertices belong to the three defining edges is ruled out from the convex contour. If no such edge exists, the triangle is not ruled out by this specific geometric test.

**Formulas:**
Let $P$ be the plane defined by a border triangle with vertices $A, B, C$. The plane equation is:
$$ \mathbf{n} \cdot (\mathbf{x} - \mathbf{A}) = 0 $$
where $\mathbf{n} = (\mathbf{B} - \mathbf{A}) \times (\mathbf{C} - \mathbf{A})$.
An edge defined by endpoints $\mathbf{U}$ and $\mathbf{V}$ lies completely to the back of $P$ if:
$$ \mathbf{n} \cdot (\mathbf{U} - \mathbf{A}) < 0 \quad \text{and} \quad \mathbf{n} \cdot (\mathbf{V} - \mathbf{A}) < 0 $$
(Note: Strict inequality assumes non-degenerate triangles and general position. Degenerate cases are ignored.)

**Parameters:**
- **Isovalue:** Implicitly handled by the sign configuration.
- **Convex Hull Property:** The fundamental constraint ensuring the front-facing half-space contains other sign-changing edges.

**Edge Cases and Pitfalls:**
- **Degenerate Border Triangles:** If three selected endpoints are collinear or coplanar in a way that forms a zero-area triangle, the border triangle is degenerate and ignored in the back-facing check.
- **Partial Front-Facing:** If an intersecting edge lies partially to the front of at least one border triangle, the triangle is not ruled out. This preserves potential valid configurations.
- **Precision:** Floating-point comparisons for "back" vs "front" must handle near-zero values carefully to avoid incorrect pruning due to numerical noise.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_triangle_ruled_out(cell_corners, sign_mask, edge_intersections, triangle_vertices):
    """
    Check if a triangle defined by triangle_vertices is ruled out by pre-triangulation.
    
    Args:
        cell_corners: List of 8 3D vectors for voxel corners.
        sign_mask: List of 8 booleans indicating sign of each corner.
        edge_intersections: List of edges (pairs of corner indices) that have sign changes.
        triangle_vertices: List of 3 corner indices forming the candidate triangle.
        
    Returns:
        True if the triangle is ruled out, False otherwise.
    """
    # Identify the three edges that the triangle vertices lie on
    tri_edge_indices = []
    for v in triangle_vertices:
        edge_idx = None
        for i, edge in enumerate(edge_intersections):
            if v in edge:
                edge_idx = i
                break
        if edge_idx is None:
            return False # Vertex not on an intersecting edge, invalid input
        tri_edge_indices.append(edge_idx)
    
    # Get the three edges
    e1_idx, e2_idx, e3_idx = tri_edge_indices
    e1 = edge_intersections[e1_idx]
    e2 = edge_intersections[e2_idx]
    e3 = edge_intersections[e3_idx]
    
    # Construct all border triangles from endpoints of e1, e2, e3
    border_triangles = []
    for c1 in e1:
        for c2 in e2:
            for c3 in e3:
                # Check for degeneracy (collinear or same point)
                if len({c1, c2, c3}) < 3:
                    continue
                border_triangles.append((c1, c2, c3))
    
    # Check if any other intersecting edge lies completely to the back of ALL border triangles
    for other_edge_idx, other_edge in enumerate(edge_intersections):
        if other_edge_idx in tri_edge_indices:
            continue # Skip the edges defining the triangle
        
        # Check if this edge is completely to the back of all non-degenerate border triangles
        completely_back = True
        for bt in border_triangles:
            c1, c2, c3 = bt
            # Calculate normal
            v1 = cell_corners[c2] - cell_corners[c1]
            v2 = cell_corners[c3] - cell_corners[c1]
            normal = cross(v1, v2)
            
            # Check both endpoints of the other edge
            u = cell_corners[other_edge[0]]
            v = cell_corners[other_edge[1]]
            
            # Check if both are strictly on the back side
            if dot(normal, u - cell_corners[c1]) >= 0 or dot(normal, v - cell_corners[c1]) >= 0:
                completely_back = False
                break
        
        if completely_back:
            return True # Triangle is ruled out
            
    return False
```

## Subtopics

### [Decision tree for triangulation](1356-decision-tree-for-triangulation.md)
A pre-computed structure used to distinguish the correct triangulation from all possible triangulations in set $S$ by performing a minimal set of 4-point tests. It builds on [Pre-triangulation of Convex Contour](1226-pre-triangulation-of-convex-contour.md) by resolving the remaining ambiguity in the convex contour patches. Unlike [Marching Cubes](52-marching-cubes.md), which relies on a static lookup table for all cases, this method uses a dynamic traversal of a decision tree to select the triangulation that ensures the resulting surface is manifold and crack-less.

## Operating characteristics
- **Time Complexity:** Pre-triangulation is performed offline during table construction. The cost is proportional to the number of sign configurations and the number of possible triangulations per patch. At runtime, it adds no overhead as the results are stored in the lookup table.
- **Memory Complexity:** Reduces the size of the lookup table by storing only valid triangulations, potentially saving memory compared to storing all combinatorial possibilities.
- **Strengths:** Significantly reduces the number of candidate triangulations that need to be considered at runtime, leading to faster surface extraction. Ensures topological consistency with the convex hull property.
- **Limitations:** Only applies to convex contouring. Requires careful handling of degenerate cases. The pre-computation can be complex for high-genus patches.
- **When to use:** When extracting isosurfaces where the convex hull property is desired and runtime performance is critical.
- **When not to use:** When non-convex features are required or when the overhead of table construction is prohibitive for dynamic data.

## Background
The method was introduced in the context of convex contouring of volumetric data [S1]. It addresses the issue that brute-force enumeration of triangulations for patches with complex topologies (e.g., genus-2) results in a large number of redundant configurations that never appear on the convex hull. By leveraging the spatial restrictions of edge intersections, the method prunes these invalid configurations during table construction. This approach ensures that the resulting surface respects the convex hull property, avoiding artifacts that can arise from arbitrary triangulations [S1].

## Sources
[S1] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
