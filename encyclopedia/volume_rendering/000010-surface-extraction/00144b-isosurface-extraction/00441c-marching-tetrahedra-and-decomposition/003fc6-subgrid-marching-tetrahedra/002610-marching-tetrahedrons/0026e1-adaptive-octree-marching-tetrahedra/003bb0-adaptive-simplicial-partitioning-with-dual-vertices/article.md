# Adaptive Simplicial Partitioning with Dual Vertices

Also called: Adaptive simplicial partitioning, Dual vertex placement on simplicial partitions

## Overview
Adaptive Simplicial Partitioning with Dual Vertices is an isosurface extraction method that constructs a simplicial complex from an adaptive octree to facilitate mesh generation [S1]. It differs from standard Marching Tetrahedra by introducing "dual vertices" within octree cells (cubes, faces, edges) rather than using only the grid vertices [S1]. These dual vertices are positioned to minimize distance to local tangent planes, allowing for better approximation of sharp features and reducing the sliver triangles common in naive tetrahedralizations [S1]. It builds on [Adaptive Octree Marching Tetrahedra](3897-adaptive-octree-marching-tetrahedra.md) by refining the geometric representation within the leaf nodes and their shared boundaries [S1].

## How it works
**Inputs:** An adaptive octree representing a scalar field $F(\mathbf{p})$, a target isovalue $I$, and a small epsilon $\epsilon$ (e.g., $10^{-6}$) to prevent boundary degeneracy [S1].
**Outputs:** A set of tetrahedra forming the isosurface mesh [S1].

**Data Structures:**
*   **Octree Node:** Contains spatial bounds, scalar values at vertices, and a flag indicating if it is a leaf (minimal 3-cell) [S1].
*   **Dual Vertex:** A 4D point $\bar{p} = \langle \mathbf{p}, F(\mathbf{p}) \rangle \in \mathbb{R}^4$, where $\mathbf{p} \in \mathbb{R}^3$ is the spatial position and $F(\mathbf{p})$ is the function value [S1].
    *   For 0-cells (vertices), $\bar{p}$ coincides with the octree vertex [S1].
    *   For $m$-cells ($m > 0$), $\mathbf{p}$ is constrained to lie strictly inside the cell, scaled by $1-\epsilon$ [S1].

**Algorithm Steps:**

1.  **Identify Minimal Cells:** Traverse the octree to find all minimal $m$-cells. A minimal $m$-cell is a cell of dimension $m$ that does not contain any other cells of dimension $m$ as a subset [S1]. Minimal 3-cells are leaf cubes. Minimal 2-cells are faces shared by adjacent leaf cubes (or exposed faces of the domain). Minimal 1-cells are edges shared by adjacent minimal faces. Minimal 0-cells are vertices [S1].
2.  **Sample Field Data:** For each minimal $m$-cell, sample the scalar field $F(\mathbf{p})$ and the normal vector $\bar{n}_i = \langle \nabla F(\mathbf{p}_i), -1 \rangle$ at each octree vertex $\mathbf{p}_i$ on the boundary of the cell [S1]. If $m < 3$, restrict the gradient $\nabla F(\mathbf{p}_i)$ to the subspace defined by the cell by projecting it onto that subspace [S1].
3.  **Compute Dual Vertices:** For each minimal $m$-cell, compute the dual vertex $\bar{x} = \langle \mathbf{x}, F(\mathbf{x}) \rangle$ by minimizing the squared distance to the tangent planes defined by the boundary samples [S1]. The tangent plane at vertex $\mathbf{p}_i$ is defined by normal $\bar{n}_i$ and point $\bar{p}_i$. Solve the constrained optimization problem:
    $$ \min_{\bar{x}} \sum_i (\bar{n}_i \cdot \bar{x} - \bar{n}_i \cdot \bar{p}_i)^2 $$
    subject to the constraint that the spatial coordinates $\mathbf{x}$ lie within the cell scaled by $1-\epsilon$ [S1]. This is a quadratic minimization with box constraints. A standard method for solving this is described by Lindstrom et al. [LS01] [S1]. Pitfall: In regions of high curvature, the function value $F(\mathbf{x})$ derived from this plane intersection may be a poor estimate (consistently high or low), potentially causing dimpling [S1].
4.  **Recursive Tetrahedralization:** Construct the simplicial complex recursively based on dimension $m$ [S1]. Base Case ($m=0$): Each minimal 0-cell (vertex) is a 0-simplex (point) consisting of its dual vertex [S1]. Recursive Step: For each minimal $(i+1)$-cell with a dual vertex, connect this dual vertex to all $i$-simplices defined on the boundary of the $(i+1)$-cell. This forms a set of $(i+1)$-simplices [S1]. Example: Edges (1-simplices) are formed by connecting edge-dual vertices to the 0-simplices (vertices) at their endpoints. Example: Faces (2-simplices) are formed by connecting face-dual vertices to the 1-simplices (edges) on their boundary. Example: Cubes (3-simplices/tetrahedra) are formed by connecting cube-dual vertices to the 2-simplices (faces) on their boundary [S1].
5.  **Isosurface Extraction:** Apply the standard Marching Tetrahedra algorithm [DK91] to the resulting tetrahedra (3-simplices) to extract the isosurface. The scalar values at the dual vertices are used to determine which edges of the tetrahedra intersect the isovalue [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_simplicial_partitioning(octree, isovalue, epsilon):
    """
    Extracts isosurface using adaptive simplicial partitioning.
    
    Args:
        octree: Root node of the adaptive octree.
        isovalue: The target scalar value for the isosurface.
        epsilon: Small value to constrain dual vertices away from cell boundaries.
        
    Returns:
        List of tetrahedra (each defined by 4 indices into the dual vertex list).
    """
    # Step 1: Identify minimal cells
    minimal_cells = find_minimal_cells(octree)
    
    # Step 2 & 3: Compute dual vertices
    dual_vertices = {}  # Map from cell_id to 4D tuple (x, y, z, f_val)
    
    for cell in minimal_cells:
        boundary_vertices = get_boundary_vertices(cell)
        samples = []
        
        for p_i in boundary_vertices:
            f_val = evaluate_field(p_i)
            grad = compute_gradient(p_i)
            
            # Restrict gradient to cell subspace if m < 3
            if cell.dimension < 3:
                grad = project_gradient_onto_subspace(grad, cell)
            
            # Normal vector in 4D: <grad, -1>
            normal_4d = (grad[0], grad[1], grad[2], -1.0)
            point_4d = (p_i[0], p_i[1], p_i[2], f_val)
            samples.append((point_4d, normal_4d))
        
        # Solve for dual vertex position x
        # Minimize sum_i (n_i . x - n_i . p_i)^2
        # subject to x spatial coords in cell scaled by (1-epsilon)
        
        # Extract spatial normals and targets for least squares
        A = []
        b = []
        for point_4d, normal_4d in samples:
            spatial_normal = (normal_4d[0], normal_4d[1], normal_4d[2])
            # n_i . p_i (4D dot product)
            dot_product = sum(n * p for n, p in zip(normal_4d, point_4d))
            A.append(spatial_normal)
            b.append(dot_product)
            
        # Solve min ||A x - b||^2 subject to bounds using projected gradient descent [G]
        x_spatial = solve_box_constrained_least_squares(A, b, cell.bounds, epsilon)
        
        # Estimate function value F(x)
        # Source notes F(x) from plane intersection can be inaccurate in high curvature.
        # We evaluate the field directly at the computed spatial position.
        f_x = evaluate_field(x_spatial)
            
        dual_vertices[cell.id] = (x_spatial[0], x_spatial[1], x_spatial[2], f_x)

    # Step 4: Recursive Tetrahedralization
    simplices = {0: [], 1: [], 2: []}  # Map dimension to list of simplices (lists of vertex indices)
    
    # 0-simplices: points
    for cell in minimal_cells:
        if cell.dimension == 0:
            simplices[0].append([cell.id])
            
    # 1-simplices: edges
    for cell in minimal_cells:
        if cell.dimension == 1:
            boundary_0_cells = get_boundary_cells(cell, dimension=0)
            simplex_vertices = [cell.id]
            for b_cell in boundary_0_cells:
                simplex_vertices.append(b_cell.id)
            simplices[1].append(simplex_vertices)
            
    # 2-simplices: faces
    for cell in minimal_cells:
        if cell.dimension == 2:
            boundary_1_cells = get_boundary_cells(cell, dimension=1)
            simplex_vertices = [cell.id]
            for b_cell in boundary_1_cells:
                # Each 1-simplex is an edge. Retrieve its vertices.
                for edge in simplices[1]:
                    if set(edge) == set([b_cell.id]):
                        simplex_vertices.extend(edge)
                        break
            simplices[2].append(simplex_vertices)
            
    # 3-simplices: tetrahedra
    tetrahedra = []
    for cell in minimal_cells:
        if cell.dimension == 3:
            boundary_2_cells = get_boundary_cells(cell, dimension=2)
            
            # Connect cube dual vertex to 2-simplices (faces) on boundary
            for b_cell in boundary_2_cells:
                # Find the corresponding 2-simplex (face)
                face_vertices = None
                for face in simplices[2]:
                    if b_cell.id in face:
                        face_vertices = face
                        break
                
                if face_vertices is None:
                    continue
                    
                # Marching Tetrahedra requires triangular faces.
                # If the face is a quad (4 vertices), split it into two triangles.
                if len(face_vertices) == 4:
                    tri1 = [face_vertices[0], face_vertices[1], face_vertices[2]]
                    tri2 = [face_vertices[0], face_vertices[2], face_vertices[3]]
                    tetrahedra.append([cell.id] + tri1)
                    tetrahedra.append([cell.id] + tri2)
                else:  # Triangle
                    tetrahedra.append([cell.id] + face_vertices)

    # Step 5: Isosurface Extraction
    mesh = []
    for tet in tetrahedra:
        vals = [dual_vertices[v][3] for v in tet]
        intersections = marching_tetrahedra_step(tet, vals, isovalue)
        mesh.extend(intersections)
        
    return mesh

def find_minimal_cells(node):
    """Traverses the octree to collect all minimal cells of all dimensions."""
    cells = []
    if node.is_leaf:
        cells.append(node)  # Minimal 3-cell
        # In a complete implementation, shared faces, edges, and vertices 
        # would be collected from adjacent leaf nodes. For this self-contained example,
        # we assume the octree structure exposes them or they are handled by a 
        # higher-level traversal that populates minimal_cells.
    else:
        for child in node.children:
            cells.extend(find_minimal_cells(child))
    return cells

def get_boundary_vertices(cell):
    """Returns the octree vertices on the boundary of the cell."""
    return cell.vertices

def compute_gradient(p):
    """Returns the gradient vector at point p using finite differences or stored data."""
    # Standard central difference approximation [G]
    h = 1e-5
    dx = (evaluate_field((p[0]+h, p[1], p[2])) - evaluate_field((p[0]-h, p[1], p[2]))) / (2*h)
    dy = (evaluate_field((p[0], p[1]+h, p[2])) - evaluate_field((p[0], p[1]-h, p[2]))) / (2*h)
    dz = (evaluate_field((p[0], p[1], p[2]+h)) - evaluate_field((p[0], p[1], p[2]-h))) / (2*h)
    return (dx, dy, dz)

def project_gradient_onto_subspace(grad, cell):
    """Projects gradient onto the subspace defined by the cell dimension."""
    if cell.dimension == 0:
        return (0.0, 0.0, 0.0)
    elif cell.dimension == 1:
        # Project onto edge direction
        edge_dir = normalize(cell.vertices[1][0]-cell.vertices[0][0], cell.vertices[1][1]-cell.vertices[0][1], cell.vertices[1][2]-cell.vertices[0][2])
        dot = sum(g*d for g, d in zip(grad, edge_dir))
        return (dot*edge_dir[0], dot*edge_dir[1], dot*edge_dir[2])
    elif cell.dimension == 2:
        # Project onto face plane (simplified: zero out normal component)
        # Assumes cell has a stored normal or computes it from vertices [G]
        return grad  # Placeholder for exact projection; standard projection subtracts normal component
    return grad

def solve_box_constrained_least_squares(A, b, bounds, epsilon):
    """Solves min ||Ax - b||^2 subject to bounds using projected gradient descent [G]."""
    # bounds: ((min_x, min_y, min_z), (max_x, max_y, max_z))
    # Scale bounds by (1-epsilon)
    lower = tuple(b * (1 - epsilon) for b in bounds[0])
    upper = tuple(b * (1 - epsilon) for b in bounds[1])
    
    x = [(lower[i] + upper[i]) / 2.0 for i in range(3)]
    lr = 0.01
    for _ in range(100):
        # Compute gradient of ||Ax - b||^2: 2 * A^T (Ax - b)
        Ax_b = [sum(a * xi for a, xi in zip(row, x)) - bi for row, bi in zip(A, b)]
        grad = [0.0, 0.0, 0.0]
        for i in range(3):
            for row, diff in zip(A, Ax_b):
                grad[i] += 2.0 * row[i] * diff
        # Update
        x = [x[i] - lr * grad[i] for i in range(3)]
        # Project onto box constraints
        x = [max(lower[i], min(upper[i], x[i])) for i in range(3)]
    return tuple(x)

def get_boundary_cells(cell, dimension):
    """Returns cells of specified dimension on the boundary of the cell."""
    # Returns list of cell objects representing boundary elements
    return cell.boundary_cells[dimension]

def marching_tetrahedra_step(tet, vals, isovalue):
    """Standard Marching Tetrahedra implementation for a single tetrahedron."""
    # Determines intersections and returns triangle vertices [G]
    intersections = []
    # Standard MT lookup table and interpolation logic would go here
    return intersections
```

## Subtopics

### [Triangulation Improvement via Isosurface Projection](9051-triangulation-improvement-via-isosurface-projection.md)
A post-processing step to reduce sliver triangles by moving dual vertices onto the isosurface while keeping them within their respective cells.

## Operating characteristics
*   **Time Complexity:** Dominated by the octree traversal and the solution of the least-squares problem for each minimal cell. The least-squares problem is small (constant size per cell) but involves solving a constrained quadratic program [S1].
*   **Memory Complexity:** Proportional to the number of minimal cells in the octree, as each requires storing a dual vertex [S1].
*   **Strengths:** Better approximation of sharp features compared to standard grid-based methods; reduces T-intersections by constraining dual vertices within cells; adaptive resolution allows for efficient representation of complex geometries [S1].
*   **Limitations:** The function value estimation at dual vertices can be inaccurate in high-curvature regions, leading to visual artifacts like dimpling. The recursive tetrahedralization and subsequent Marching Tetrahedra step add complexity [S1].
*   **When to use:** When high-quality isosurfaces with sharp features are required from adaptive data, and standard Marching Cubes/Tetrahedra produce insufficiently smooth or accurate results [S1].

## Background
The method was introduced by Lindstrom et al. in the paper "Isosurfaces Over Simplicial Partitions of Multiresolution Grids" [S1]. It extends the concept of simplicial partitions to multiresolution grids (octrees), allowing for adaptive refinement. The approach addresses the limitations of uniform grid methods and standard adaptive methods by providing a more flexible geometric representation within the cells [S1].

## Sources
[S1] Isosurfaces Over Simplicial Partitions of Multiresolution Grids, http://faculty.cs.tamu.edu/schaefer/research/iso_simplicial.pdf, fetched 2026-10-29
