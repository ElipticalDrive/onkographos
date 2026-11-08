# Isotopic Approximation of Implicit Curves and Surfaces

Also called: ApproximateCurve, Isotopic Approximation

## Overview
The Isotopic Approximation algorithm generates a piecewise linear mesh that is regularly isotopic to an implicit curve in $\mathbb{R}^2$ or surface in $\mathbb{R}^3$, ensuring the approximation has the same topology as the original manifold [S1]. Unlike grid-based schemes that may miss critical details and fail to guarantee correct topology, this method uses interval arithmetic to adaptively subdivide space via a quadtree (curves) or octree (surfaces), guaranteeing that every leaf cell is locally parametrizable [S1]. It is distinguished from sibling methods by its reliance on a simple interval test to ensure isotopy without computing critical points, making it fast and suitable for visualization and time-dependent surface morphing [S1].

## How it works
### Inputs and Outputs
**Inputs:**
1.  $F$: A smooth implicit function $F: \mathbb{R}^2 \to \mathbb{R}$ (for curves) or $F: \mathbb{R}^3 \to \mathbb{R}$ (for surfaces).
2.  $B$: A bounding box (square in $\mathbb{R}^2$, cube in $\mathbb{R}^3$) containing the implicit manifold $S = F^{-1}(0)$.
3.  $F$ must have $0$ as a regular value, meaning $\nabla F$ is non-zero at every point on $S$ [S1].

**Outputs:**
A piecewise linear approximation of $S$ consisting of line segments (curves) or triangles (surfaces) [S1].

### Data Structures
**Quadtree/Octree Node:**
Each node represents a spatial cell (square or cube).
*   `level`: Integer depth of subdivision.
*   `bounds`: Interval bounds defining the cell geometry.
*   `children`: List of 4 children (quadtree) or 8 children (octree), or `null` if leaf.
*   `is_leaf`: Boolean flag.

**Interval Arithmetic Types:**
*   `Interval`: Represents a range $[min, max]$ with error bounds.
*   `VectorInterval`: Interval vector for gradient components.

### Algorithm Steps
The algorithm proceeds in three phases: Subdivision, Balancing, and Mesh Construction.

1.  **Initialization**: Initialize the tree $T$ with the root node representing the bounding box $B$.
2.  **Adaptive Subdivision**: Recursively subdivide $T$ until all leaf cells $C$ satisfy the termination condition:
    $$0 \notin \Box F(C) \quad \lor \quad \langle \Box \nabla F(C), \Box \nabla F(C) \rangle > 0$$
    *   $\Box F(C)$ is the inclusion function evaluating $F$ over the cell $C$.
    *   $\Box \nabla F(C)$ is the inclusion function evaluating the gradient $\nabla F$ over $C$.
    *   The condition $0 \notin \Box F(C)$ implies the manifold does not intersect $C$.
    *   The condition $\langle \Box \nabla F(C), \Box \nabla F(C) \rangle > 0$ implies the gradient norm is strictly positive over $C$. Specifically, at least one component of the gradient (e.g., $F_x$ or $F_y$) does not contain 0 in its interval, implying $F$ is strictly monotonic in that direction, making the cell locally parametrizable [S1].
3.  **Balancing**: Balance the tree so that neighboring cells differ in size by at most a factor of two. This ensures that a single side of a cell may consist of multiple quadtree/octree edges if neighbors are subdivided [S1].
4.  **Vertex Placement**: For each edge of the tree structure:
    *   Evaluate $F$ at the two endpoints of the edge.
    *   If the signs of $F$ at the endpoints are opposite, place a vertex at the midpoint of the edge [S1].
5.  **Mesh Construction**: For each leaf cell $C$:
    *   Identify the vertices placed on the boundary of $C$.
    *   **Case 1 (Two Vertices)**: If the leaf contains exactly two vertices, connect them with a line segment (curve) or triangle edge (surface) [S1].
    *   **Case 2 (Four Vertices)**: If the leaf contains four vertices, the correctness proof guarantees that two of them lie on the same side of the cell. Connect each of these two vertices to one of the other two vertices, forming two triangles (surface) or a specific configuration for curves [S1].
    *   *Note*: The algorithm handles only these cases. A single side of a cell may consist of two tree edges if the neighbor is subdivided, meaning a cell is surrounded by 4 to 8 tree edges [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def IsotopicApproximation(F, B, max_depth=10):
    """
    Generates an isotopic piecewise linear approximation of F=0 within bounding box B.
    F: Function handle.
    B: Bounding box (Interval2 or Interval3).
    max_depth: Optional limit to handle singularities.
    Returns: List of edges (2D) or triangles (3D).
    """
    # 1. Initialize Quadtree/Octree
    root = Node(bounds=B, level=0)
    tree = Tree(root)
    
    # 2. Subdivision Phase
    queue = [root]
    while queue:
        node = queue.pop(0)
        
        # Check termination condition for current cell
        # Compute inclusion of F over node.bounds
        F_interval = interval_eval(F, node.bounds)
        
        # Compute inclusion of Gradient over node.bounds
        # Returns VectorInterval [Fx_interval, Fy_interval, (Fz_interval)]
        grad_interval = interval_gradient_eval(F, node.bounds)
        
        # Condition: 0 not in F_interval OR norm(grad_interval)^2 > 0
        zero_not_in_F = not interval_contains_zero(F_interval)
        
        # Compute squared norm of gradient interval
        # <grad, grad> = sum(component^2)
        grad_norm_sq = interval_squared_norm(grad_interval)
        
        # Check if strictly positive
        # Note: interval > 0 means min(grad_norm_sq) > 0
        grad_positive = interval_min(grad_norm_sq) > 0
        
        if zero_not_in_F or grad_positive:
            node.is_leaf = True
        else:
            node.is_leaf = False
            # Subdivide
            children = node.subdivide()
            for child in children:
                queue.append(child)
                
    # 3. Balancing Phase
    balance_tree(tree)
    
    # 4. Vertex Placement
    # Collect all unique edges in the tree structure
    all_edges = get_all_tree_edges(tree)
    vertices_on_edges = []
    
    for edge in all_edges:
        # Evaluate F at endpoints
        val_start = F(edge.start_point)
        val_end = F(edge.end_point)
        
        # Check sign change
        # Use strict sign check; if one is zero, treat as sign change or handle carefully
        if sign(val_start) != sign(val_end):
            mid = midpoint(edge.start_point, edge.end_point)
            vertex = Vertex(position=mid, edge=edge)
            vertices_on_edges.append(vertex)
            edge.has_vertex = True
            edge.vertex = vertex
            
    # 5. Mesh Construction
    mesh_elements = []
    
    for leaf in tree.leaves():
        # Get vertices on the boundary of this leaf
        boundary_vertices = get_vertices_on_leaf_boundary(leaf, vertices_on_edges)
        
        if len(boundary_vertices) == 2:
            # Connect the two vertices
            mesh_elements.append(connect(boundary_vertices[0], boundary_vertices[1]))
        elif len(boundary_vertices) == 4:
            # Find the two vertices on the same side
            # Group vertices by the face/edge they lie on
            sides = group_vertices_by_side(boundary_vertices)
            
            # Identify side with 2 vertices
            side_with_two = find_side_with_two_vertices(sides)
            v1, v2 = side_with_two.vertices
            
            # The other two vertices are on different sides
            other_vertices = []
            for s in sides:
                if s != side_with_two:
                    other_vertices.extend(s.vertices)
            
            # Connect v1 and v2 to the other vertices
            # The proof guarantees a valid pairing exists
            mesh_elements.append(connect(v1, other_vertices[0]))
            mesh_elements.append(connect(v2, other_vertices[1]))
            
    return mesh_elements

def interval_squared_norm(grad_interval):
    """Computes <grad, grad> using interval arithmetic."""
    # For 2D: Fx^2 + Fy^2
    # For 3D: Fx^2 + Fy^2 + Fz^2
    # Squaring an interval [a,b]:
    # if a*b >= 0: [min(a^2, b^2), max(a^2, b^2)]
    # if a*b < 0: [0, max(a^2, b^2)]
    sum_sq = Interval(0, 0)
    for comp in grad_interval.components:
        sum_sq = sum_sq + interval_square(comp)
    return sum_sq

def interval_square(interval):
    a = interval.min
    b = interval.max
    if a * b >= 0:
        return Interval(min(a**2, b**2), max(a**2, b**2))
    else:
        return Interval(0, max(a**2, b**2))
```

### Parameters
*   **Minimal Subdivision Level**: Can be defined to improve the Hausdorff distance between the surface and the approximation [S1].
*   **Maximal Subdivision Level**: Can be used to improve speed and handle surfaces with singularities. In this case, color coding can identify regions where the approximation might be topologically incorrect [S1].

### Edge Cases and Pitfalls
*   **Singularities**: If $F$ has singularities (where $\nabla F = 0$), the condition $\langle \nabla F, \nabla F \rangle > 0$ may never be met, leading to infinite subdivision. A maximal subdivision level must be enforced, and the resulting mesh may require color coding to indicate potential topological errors [S1].
*   **Boundary Conditions**: A single side of a cell may consist of two tree edges if the neighboring cell is subdivided. The algorithm must correctly identify vertices on these composite edges [S1].
*   **Precision**: Interval arithmetic must be used to prevent rounding errors that could incorrectly classify a cell as containing zero or not [S1].

## Subtopics

### Interval Arithmetic
A mathematical framework for computing with intervals to bound rounding errors and provide rigorous inclusion guarantees. Essential for the subdivision test. [Link to Interval Arithmetic article if available, otherwise generic link]

### Regular Isotopy
A topological property where two manifolds are equivalent under continuous deformation within the embedding space. This algorithm guarantees regular isotopy for implicit surfaces. [Link to Topology article if available]

## Operating characteristics
*   **Time Complexity**: The subdivision process terminates because squares/cubes shrink towards a point, and for a regular curve/surface, at least one of the two clauses ($0 \notin F(C)$ or $\langle \nabla F, \nabla F \rangle > 0$) converges to a non-zero value [S1]. The algorithm is fast because it uses a simple interval test and adds little overhead compared to other enumeration methods [S1].
*   **Memory Complexity**: Proportional to the number of leaf nodes in the balanced quadtree/octree.
*   **Strengths**: Guarantees correct topology (regular isotopy); does not require computing critical points; fast due to simple interval tests; flexible adaptive resolution.
*   **Limitations**: Requires $F$ to be smooth and $0$ to be a regular value (gradient non-zero on surface); singularities require special handling (max depth); interval arithmetic can be computationally expensive if not optimized.
*   **When to Use**: Visualization of implicit surfaces where piecewise linear approximations are required; constructing initial meshes for time-dependent surfaces (e.g., morphing) [S1].
*   **When Not to Use**: When the implicit function has singularities and high precision is required without fallback strategies; when performance is critical and interval arithmetic overhead is prohibitive.

## Background
The Isotopic Approximation algorithm was introduced by [S1] as one of the first methods to guarantee isotopy for implicit surface meshing. It falls into the category of enumeration methods but distinguishes itself by using a fast and simple interval test to decide subdivision, avoiding the need to compute critical points of the implicit function. The method is based on a quadtree (for curves) or octree (for surfaces) and uses interval arithmetic to ensure that leaf cells are locally parametrizable. The approach was developed to address the limitations of grid-based schemes, which can miss important surface details and fail to guarantee correct topology.

## Sources
[S1] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29
