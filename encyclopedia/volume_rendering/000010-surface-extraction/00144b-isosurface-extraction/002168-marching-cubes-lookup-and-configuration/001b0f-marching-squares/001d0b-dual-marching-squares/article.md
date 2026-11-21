# Dual Marching Squares

Also called: Schaefer's and Warren's Dual Marching Cubes (2D variant)

## Overview
Dual Marching Squares (DMS) is a 2D surface extraction algorithm that generates a polygonal mesh from a scalar field by placing a single vertex per grid cell (or quadtree leaf) at the cell's center (dual grid position) and connecting these vertices into quads [S1, S2, unofficial]. Unlike standard Marching Squares, which places vertices on cell edges, DMS produces smoother results and avoids topological inconsistencies and self-intersections common in other techniques, though it may lose sharp features due to the fixed center placement [S1, S2, unofficial]. It builds on [Marching Squares](918-marching-squares.md) by replacing edge-based vertex placement with a dual-grid approach.

## How it works
**Inputs:** A 2D scalar field $f(x,y)$, an isovalue $T$, a minimum leaf size $min\_size$, and a grid scaling factor $grid\_size$.
**Outputs:** A list of vertices (position, normal, color) and a list of quad indices forming the mesh.

**Data Structures:**
1.  **QuadtreeNode**: Represents a region of the grid.
    *   `position`: `Vector2` (top-left corner of the node in grid units).
    *   `size`: `int` (side length of the square node).
    *   `children`: `QuadtreeNode[4]` (references to 4 child nodes, null if leaf).
    *   `dualgrid_pos`: `Vector2` (offset from `position` to the dual vertex, typically `0.5 * size`).
    *   `isovalue`: `float` (sampled value at `dualgrid_pos`).
    *   `normal`: `Vector2` (gradient at `dualgrid_pos`, normalized).
    *   `leaf`: `bool` (true if this node is a leaf).
    *   `vertex_index`: `int` (index in the global vertex list, valid if leaf).
    *   `index`: `int` (internal node identifier).
2.  **VertexPositionColorNormal**:
    *   `Position`: `Vector3` (3D position, Z=0 for 2D).
    *   `Color`: `Color` (e.g., RGB 210, 220, 210).
    *   `Normal`: `Vector3` (2D normal extended to 3D with Z=0).
3.  **Connectivity Tables** (used during mesh construction):
    *   `edges[4][2]`: `{{0, 2}, {1, 3}, {0, 1}, {2, 3}}`. Maps edge index to corner indices.
    *   `edge_children[2][2][2]`: `{{{2, 0}, {3, 1}}, {{1, 0}, {3, 2}}}`. Maps parent edge children to child quad edges.
    *   `vertex_children[2][2][2]`: `{{{2, 3}, {0, 1}}, {{1, 3}, {0, 2}}}`. Maps parent vertex children to child quad vertices.
    *   `middle_points[4]`: `{3, 2, 1, 0}`.
    *   `connect_points[4]`: `{0, 2, 3, 1}`.

**Parameters:**
*   `threshold`: The error threshold for adaptive splitting. Default: Not given in the sources.
*   `min_size`: Minimum leaf node size. Default: Not given in the sources.
*   `minSplitDistanceDiagonalFactor`: Scaling factor for diagonal distance check. Default: 1000.0f [S2, unofficial].

**Steps:**

1.  **Initialize Quadtree**: Create root node with `position = (0,0)`, `size = grid_size`, `leaf = false`.
2.  **Build Quadtree (Recursive Splitting)**:
    Call `TrySplit(node, min_size, threshold, grid_size, vertices)`.
    
    *   **Check Split Conditions**: A node becomes a leaf if *any* of the following are true:
        1.  `size <= min_size`.
        2.  `GetError(node, threshold) < threshold`.
        3.  The sample at the center is far from the threshold: $|f(center) - T| > size \times \sqrt{2} \times minSplitDistanceDiagonalFactor$.
    
    *   **If Leaf**:
        1.  Calculate `dualgrid_pos = position + (0.5 * size, 0.5 * size)`.
        2.  Sample `isovalue = f(dualgrid_pos)`.
        3.  Compute `normal = gradient(f, dualgrid_pos)`.
        4.  Create `VertexPositionColorNormal`:
            *   `Position = (position + dualgrid_pos) * grid_size`.
            *   `Normal = (normal.x, normal.y, 0)`.
            *   `Color = (210, 220, 210)`.
        5.  Add vertex to `vertices` list. Store its index in `node.vertex_index`.
        6.  Set `node.leaf = true`. Return.
    
    *   **If Not Leaf**:
        1.  Set `node.leaf = false`.
        2.  Split into 4 children. For `i` in 0..3:
            *   Create new `QuadtreeNode`.
            *   `child.index = i`.
            *   `child.size = size / 2`.
            *   `child.position = position + (size / 2) * (i / 2, i % 2)`.
            *   `child.children = [null, null, null, null]`.
            *   `node.children[i] = child`.
            *   Recursively call `TrySplit(child, min_size, threshold, grid_size, vertices)`.

3.  **Construct Mesh (Connectivity)**:
    Call `ConstructTreeGrid(root)` to generate indices. This traverses the tree to connect leaf nodes into quads.
    
    *   **ProcessFace(node)**:
        1.  If `node` is leaf, return.
        2.  Recursively call `ProcessFace` on all 4 children.
        3.  Process 4 edges between children (see `ProcessEdge`).
        4.  Process 4 vertices of the node (see `ProcessVertices`).

    *   **ProcessEdge(q1, q2, edge_index)**:
        Handles adjacency between sibling nodes `q1` and `q2` along a specific edge.
        1.  If both `q1` and `q2` are leaves, return (edge already processed or not needed).
        2.  If either is not a leaf, recursively process sub-edges using `edge_children` mapping.
        3.  If both are leaves, call `ProcessVertices` on the 4 leaf nodes surrounding this edge.

    *   **ProcessVertices(q1, q2, q3, q4)**:
        Takes 4 `QuadtreeNodes` representing the corners of a quad.
        1.  If any child is not a leaf, resolve it to its middle-point child and recurse.
        2.  If all are leaves:
            *   Create a `Cell` (quad).
            *   Get vertex indices from `q1..q4.vertex_index`.
            *   Add indices to the mesh using `connect_points` order: `(q1, q3, q4, q2)` (indices 0, 2, 3, 1).
            *   Set `Cell` positions, values, and normals from the leaf nodes' `dualgrid_pos`, `isovalue`, and `normal`.

**Error Calculation (GetError):**
To determine if a node should split, calculate error based on bilinear interpolation:
1.  Sample `center_value` at 5 positions: 4 corners and center.
2.  Compute `interpolated` value at the center using bilinear interpolation of the 4 corner values.
3.  `error += |center_value - interpolated| / max(1.0, |gradient|)`.
4.  If `error >= threshold`, return `true` (split).
5.  [G] Pitfall: If sampled points have very similar values, error may be negligible even for complex surfaces. Fix: Sample more points (e.g., midpoints, random points) or use a maximum container size [S2, unofficial].

**Bilinear Interpolation (Interpolate):**
$$f(p) = f_{00}(1-p_x)(1-p_y) + f_{01}(1-p_x)p_y + f_{10}p_x(1-p_y) + f_{11}p_x p_y$$
where $p_x, p_y$ are normalized coordinates within the cell [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_dual_marching_squares(grid, isovalue, min_size, threshold, grid_size):
    """
    grid: 2D array of floats
    isovalue: float
    min_size: int
    threshold: float
    grid_size: int (total resolution)
    """
    root = QuadtreeNode(position=(0,0), size=grid_size, leaf=False)
    vertices = []
    
    try_split(root, min_size, threshold, grid_size, vertices, grid, isovalue)
    
    indices = []
    construct_tree_grid(root, indices)
    
    return vertices, indices

def try_split(node, min_size, threshold, grid_size, vertices, grid, isovalue):
    if node.size <= min_size:
        create_leaf(node, grid, grid_size, vertices, isovalue)
        return
    
    error = get_error(node, grid, isovalue, threshold)
    if error < threshold:
        create_leaf(node, grid, grid_size, vertices, isovalue)
        return
        
    center_val = sample_grid(grid, node.position + (node.size/2, node.size/2))
    diag_factor = 1000.0 # minSplitDistanceDiagonalFactor
    if abs(center_val - isovalue) > node.size * sqrt(2) * diag_factor:
        create_leaf(node, grid, grid_size, vertices, isovalue)
        return

    # Split
    node.leaf = False
    half = node.size // 2
    for i in range(4):
        child = QuadtreeNode(
            position=(node.position[0] + (i//2)*half, node.position[1] + (i%2)*half),
            size=half,
            leaf=False
        )
        node.children[i] = child
        try_split(child, min_size, threshold, grid_size, vertices, grid, isovalue)

def create_leaf(node, grid, grid_size, vertices, isovalue):
    node.leaf = True
    dual_pos = (node.position[0] + node.size/2, node.position[1] + node.size/2)
    val = sample_grid(grid, dual_pos)
    norm = compute_gradient(grid, dual_pos)
    
    pos_3d = ((node.position[0] + dual_pos[0]) * grid_size, 
              (node.position[1] + dual_pos[1]) * grid_size, 0)
    
    vertex = VertexPositionColorNormal(
        position=pos_3d,
        color=(210, 220, 210),
        normal=(norm[0], norm[1], 0)
    )
    node.vertex_index = len(vertices)
    vertices.append(vertex)

def get_error(node, grid, isovalue, threshold):
    corners = [
        sample_grid(grid, node.position),
        sample_grid(grid, (node.position[0]+node.size, node.position[1])),
        sample_grid(grid, (node.position[0], node.position[1]+node.size)),
        sample_grid(grid, (node.position[0]+node.size, node.position[1]+node.size))
    ]
    center = sample_grid(grid, (node.position[0]+node.size/2, node.position[1]+node.size/2))
    
    # Bilinear interpolation at center
    px, py = 0.5, 0.5
    f00, f01, f10, f11 = corners[0], corners[1], corners[2], corners[3]
    interpolated = f00*(1-px)*(1-py) + f01*(1-px)*py + f10*px*(1-py) + f11*px*py
    
    grad = compute_gradient(grid, (node.position[0]+node.size/2, node.position[1]+node.size/2))
    grad_mag = max(1.0, abs(grad[0]) + abs(grad[1])) # Simplified gradient magnitude
    
    error = abs(center - interpolated) / grad_mag
    return error

def construct_tree_grid(node, indices):
    if node.leaf:
        return
    
    for child in node.children:
        construct_tree_grid(child, indices)
        
    # Process edges and vertices
    # Simplified: In a real implementation, this requires traversing the tree structure
    # to identify shared edges/vertices between siblings.
    # For brevity, we assume a helper function handles the quadtree topology traversal.
    process_face(node, indices)

def process_face(node, indices):
    if node.leaf:
        return
    
    # Process children edges and vertices
    # This part requires complex recursion to handle multi-resolution connectivity
    # See notes for edge_children and vertex_children mappings
    pass
```

## Subtopics

### Dual Marching Squares Error Function
An optimization that solves a Quadratic Error Function (QEF) to position vertices more accurately relative to the implicit surface, addressing the loss of sharp features inherent in the base method [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the quadtree depth. In the worst case (uniform grid), it is $O(N)$ where $N$ is the number of cells. Adaptive splitting reduces this for smooth regions.
- **Memory Complexity**: $O(N)$ for vertices and indices. Quadtree overhead adds a constant factor.
- **Performance**: Produces smoother results than Marching Squares but may require more complex connectivity handling [S1, unofficial].
- **Strengths**: Avoids topological inconsistencies and self-intersections; handles concave surfaces well [S1, unofficial].
- **Limitations**: Loses sharp features due to center placement; error calculation can be flawed if sampled points have similar values [S2, unofficial].
- **When to use**: When smooth surfaces are preferred and sharp features are not critical.
- **When not to use**: When sharp features must be preserved (use Dual Marching Squares Error Function variant) or when edge-based precision is required (use Marching Squares).

## Background
Dual Marching Squares is the 2D variant of Dual Marching Cubes, originally proposed by Schaefer and Warren [S1, unofficial]. It is used in flow visualization and scientific visualization to extract isosurfaces from scalar fields, such as those from computational fluid dynamics simulations [S1, unofficial]. It was implemented in various visualization pipelines, including VTK [S1, unofficial].

## Sources
[S1] Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
[S2] isosurface, https://github.com/Lin20/isosurface, fetched 2026-10-29 (unofficial)
