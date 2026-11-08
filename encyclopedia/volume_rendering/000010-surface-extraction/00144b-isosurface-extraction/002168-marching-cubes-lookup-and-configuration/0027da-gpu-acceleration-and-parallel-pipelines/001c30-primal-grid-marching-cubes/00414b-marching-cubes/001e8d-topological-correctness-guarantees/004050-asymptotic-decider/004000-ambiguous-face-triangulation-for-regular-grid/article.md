# Ambiguous Face Triangulation for Regular Grid

Also called: Ambiguous Face Triangulation, Ambiguous Face Resolution

## Overview
Ambiguous Face Triangulation is a method for resolving topological ambiguities in isosurface extraction on regular grids when a single grid face has sign changes along all four of its edges, resulting in four intersection vertices on that face. It sets itself apart from the parent [Asymptotic Decider](790-asymptotic-decider.md) by using a geometric connectivity rule based on gradient orientation rather than a scalar interpolation test to determine the triangulation, ensuring that adjacent cells fit together topologically.

## How it works
**Inputs and Outputs:**
- Input: A regular grid with a scalar field $F$. For each cell, the signs of $F$ at the 8 vertices. Specifically, the configuration where exactly one face has sign changes on all four edges (Cases 4 and 6 in the standard 256-case classification).
- Output: A set of triangles forming the isosurface approximation within the cell and its neighbors.

**Data Structures:**
- Grid vertices: $V_{ijk}$ with scalar values $F_{ijk}$.
- Intersection points: Calculated via linear interpolation on edges where signs differ.
- Face configuration: A boolean mask or index identifying the ambiguous face.

**Steps:**
1. **Identify Ambiguity:** Detect if a cell has a face where $F$ changes sign on all four edges. Let this face be $f$. The four vertices of the triangulation lie on the edges of $f$.
2. **Pairwise Connection:** Connect the four vertices on face $f$ pairwise. The connection segments must be parallel to one of the vectors $(1,1,0)$, $(1,0,1)$, or $(0,1,1)$, depending on the orientation of the face [S1, unofficial].
3. **Neighbor Consistency:** Ensure the choice of pairing guarantees that the triangulations of two adjacent cells sharing the face $f$ fit together. The ambiguity is due to the sign configuration on the shared face. The individual triangulations of the two cells must be compatible [S1, unofficial].
4. **Block Analysis:** Consider the block of two cubes adjacent to the ambiguous face. The border of the union of these two cubes does not contain ambiguities because the sign changes along the edges of the shared face prevent a parametrization in the direction of those edges [S1, unofficial].
5. **Parametrization:** Both cubes are parametrizable in the vertical direction (the direction perpendicular to the ambiguous face's plane, assuming the face is horizontal/vertical). The vertical component of the gradient $\nabla F$ does not disappear, so the union of the two cells is parametrizable in the vertical direction [S1, unofficial].
6. **Joining Possibilities:** There are only three possibilities to join two ambiguous cells, determined by the types of the two cells (Type 4 or Type 6). The valid joins are:
   - Two Type 4 cells.
   - One Type 4 and one Type 6 cell.
   - Two Type 6 cells [S1, unofficial].
7. **Triangulation Choice:** For the ambiguous face, there are two ways to connect the vertices pairwise. Both choices lead to the same isotopic approximation. Choose either way, provided the triangulations for both cells fit together [S1, unofficial].

**Formulas:**
- Intersection point on edge between $V_a$ and $V_b$:
  $$ P = V_a + \frac{F_a}{F_a - F_b} (V_b - V_a) $$
- Gradient condition for vertical parametrizability:
  $$ \frac{\partial F}{\partial z} \neq 0 \quad \text{(in the vertical direction)} $$

**Parameters:**
- None specific. The method relies on the grid structure and scalar field values.

**Edge Cases and Pitfalls:**
- **Closed Loop on Face:** If the surface intersects a grid face in a closed curve lying completely within that face, special handling is required. Both adjacent cubes must be parametrizable in the direction perpendicular to the face. The "bubble" must not leave the cube on the opposite face, as this would require the gradient to change more than $\pi/2$. The bubble is continuously flattened towards the face by linear interpolation and pushed through to remove the intersection loop [S1, unofficial].
- **Surface Passing Through Vertices:** Handle by considering $F$ to be strictly positive whenever $F \ge 0$ [S1, unofficial].
- **Multiple Intersections on Edge:** If the surface intersects an edge more than once, the gradient must be perpendicular to the edge at some point between intersections. This case is handled by the general isotopic approximation constraints [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resolve_ambiguous_face(cell1, cell2, ambiguous_face):
    """
    Resolves triangulation for two adjacent cells sharing an ambiguous face.
    cell1, cell2: Objects containing vertex values and geometry.
    ambiguous_face: The face shared by cell1 and cell2 with 4 sign-changing edges.
    """
    # 1. Identify intersection points on the 4 edges of the ambiguous face
    intersections = []
    for edge in ambiguous_face.edges:
        v1, v2 = edge.vertices
        f1, f2 = cell1.value(v1), cell1.value(v2) # or cell2 if v1,v2 are shared
        if f1 * f2 < 0:
            # Linear interpolation
            t = f1 / (f1 - f2)
            p = v1 + t * (v2 - v1)
            intersections.append(p)
    
    # 2. Determine face orientation to select parallel vector
    # Vectors: (1,1,0), (1,0,1), (0,1,1)
    # Depends on whether the face is XY, XZ, or YZ oriented
    orientation = get_face_orientation(ambiguous_face)
    
    # 3. Choose pairing based on orientation to ensure compatibility
    # The choice must be consistent for both cell1 and cell2
    if orientation == 'XY':
        # Segments parallel to (1,1,0) or (1,-1,0) depending on vertex order
        # Standard approach: connect opposite vertices such that segments are parallel
        # to the diagonal of the face
        triangulation = connect_pairwise_parallel(intersections, orientation)
    elif orientation == 'XZ':
        triangulation = connect_pairwise_parallel(intersections, orientation)
    elif orientation == 'YZ':
        triangulation = connect_pairwise_parallel(intersections, orientation)
    
    # 4. Verify compatibility with neighbor
    # The triangulation must fit with the neighbor's triangulation for the same face
    # Since both cells share the face, the pairing is identical
    return triangulation

def handle_closed_loop_on_face(face, cell1, cell2):
    """
    Handles the case where the surface intersects a face in a closed loop.
    """
    # Verify both cubes are parametrizable perpendicular to the face
    if not is_parametrizable_perpendicular(cell1, face.normal) or \
       not is_parametrizable_perpendicular(cell2, face.normal):
        raise Exception("Cannot resolve closed loop: gradient vanishes in perpendicular direction")
    
    # Check bubble does not leave opposite face
    if bubble_leaves_opposite_face(face, cell1) or bubble_leaves_opposite_face(face, cell2):
        raise Exception("Bubble leaves cube: gradient change > pi/2")
    
    # Flatten bubble and push through face
    flattened_surface = flatten_bubble_towards_face(face, cell1, cell2)
    remove_intersection_loop(flattened_surface)
```

## Subtopics

### Closed Loop Removal on Grid Faces
A variant handling the case where the surface intersects a grid face in a closed curve lying completely within that face. It requires verifying that both adjacent cubes are parametrizable in the direction perpendicular to the face and that the "bubble" does not leave the cube on the opposite face (which would imply a gradient change of more than $\pi/2$). The bubble is continuously flattened towards the face by linear interpolation and pushed through to remove the intersection loop [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Constant time per ambiguous face, as it involves a fixed number of geometric checks and interpolations.
- **Memory Complexity:** Constant, requiring only local vertex and edge data.
- **Strengths:** Guarantees topological correctness (isotopy) by ensuring adjacent cells fit together. Handles the specific ambiguity of 4 vertices on one face.
- **Limitations:** Only applies to regular grids. Requires the gradient to not vanish in the vertical direction for the block of two cells. Does not handle multiple intersections on a single edge without additional constraints.
- **When to use:** When using Marching Cubes on regular grids and encountering Cases 4 and 6 ambiguities.
- **When not to use:** When the grid is not regular, or when the gradient vanishes in the direction perpendicular to the ambiguous face.

## Background
Originated in the context of isotopic approximation of implicit curves and surfaces on regular grids. The method addresses the topological ambiguities in Cases 4 and 6 of the Marching Cubes algorithm, where a single face has sign changes on all four edges. It was developed to ensure that the resulting triangulation is isotopic to the true isosurface, avoiding topological errors like tunnels or disconnected components. The approach relies on the geometric properties of the gradient and the structure of the regular grid.

## Sources
[S1] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29 (unofficial)
