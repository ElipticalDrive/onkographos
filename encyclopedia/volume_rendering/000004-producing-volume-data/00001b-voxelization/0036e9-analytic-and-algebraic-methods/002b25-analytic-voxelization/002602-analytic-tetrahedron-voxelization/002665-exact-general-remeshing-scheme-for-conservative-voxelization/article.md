# Exact General Remeshing Scheme for Conservative Voxelization

Also called: 2D Conservative Remeshing, 2D Conservative Voxelization, Conservative Voxelization of Tetrahedra, Direct remap, Exact general remeshing scheme, Physically Conservative Voxelization

## Overview
This method voxelizes a single tetrahedron onto a Cartesian grid by computing physically conservative moments (volume, linear, quadratic) for a specified polynomial order [S1, unofficial]. It ensures that the total mass in each grid cell exactly equals the integral of the input density field over the cell, avoiding aliasing noise [S1, unofficial]. Unlike standard rasterization or Sutherland-Hodgman clipping, which can fail due to geometric degeneracy and roundoff errors, this scheme uses planar graph representations to maintain topological consistency during clipping, guaranteeing robustness even when vertices lie on clip planes [S1, unofficial].

## How it works
The algorithm computes the intersection of a tetrahedron with a Cartesian grid cell (voxel) and integrates a polynomial density field over this intersection. It is divided into three phases: Searching, Clipping, and Reduction.

### Inputs and Outputs
- **Input**: A tetrahedron defined by four vertices $\mathbf{x}_0, \mathbf{x}_1, \mathbf{x}_2, \mathbf{x}_3$; a Cartesian grid cell (voxel) defined by its eight vertices; a polynomial density field defined on the tetrahedron (typically up to quadratic order).
- **Output**: The conservative moments (integrals of $1, x, y, z, x^2, xy, \dots$) of the density field over the voxel.

### Data Structures
**Polyhedron Vertex**:
Each vertex in the planar graph representation contains:
1.  Spatial coordinates $(x, y, z)$.
2.  Three pointers to neighboring vertices (forming the edges of the planar graph).
3.  A byte flag indicating if the vertex has been clipped.
4.  Four floating-point numbers storing the signed distance to each of the tetrahedron's four faces [S1, unofficial].

**Tetrahedron Faces**:
Each face $f$ is defined by a vertex $\mathbf{x}_f$ and a unit normal $\mathbf{n}_f$. The signed distance of a point $\mathbf{x}$ to face $f$ is:
$$ d_f = (\mathbf{x} - \mathbf{x}_f) \cdot \mathbf{n}_f $$
Points with $d_f \le 0$ are "behind" or "outside" the face; points with $d_f > 0$ are "inside" or "in front" [S1, unofficial].

### Steps

#### 1. Searching
Determine which voxels intersect the tetrahedron boundary.
1.  **Classification**: For each voxel, check the signed distances of its 8 vertices to all 4 faces of the tetrahedron.
    -   **Type 1 (Outside)**: All 8 vertices have $d_f \le 0$ for at least one face $f$. Ignore these voxels.
    -   **Type 2 (Inside)**: All 8 vertices have $d_f > 0$ for all 4 faces. These are fully contained.
    -   **Type 3 (Boundary)**: Voxels that are neither Type 1 nor Type 2. These require clipping.
2.  **Optimization**: Use Binary Space Partitioning (BSP) to find Type 3 voxels efficiently.
    -   Recursively split the grid bounding box along its longest dimension.
    -   Stop recursion if:
        1.  All 8 vertices of the box are outside one face (ignore box).
        2.  All 8 vertices are inside all faces (process box with a loop for Type 2 voxels).
        3.  The box contains a single voxel (process as Type 3).
    -   Complexity: $O(g^2 \log g)$ vs $O(g^3)$ for brute force, where $g$ is the linear grid dimension [S1, unofficial].

#### 2. Clipping
Construct the convex polyhedron resulting from the intersection of the voxel (cube) and the tetrahedron.
1.  **Initialization**: Initialize the planar graph of the cubical voxel. Compute signed distances for all 8 vertices to the 4 tetrahedron faces.
2.  **Iterative Clipping**: For each of the 4 faces of the tetrahedron:
    -   Find a vertex in the current graph with $d_f \le 0$ (behind the plane). If none exist, the polyhedron is entirely in front; skip this face.
    -   Perform a Depth-First Search (DFS) traversal starting from this vertex.
    -   For each vertex $\mathbf{v}$ visited in the traversal:
        -   If $\mathbf{v}$ is already marked clipped, continue.
        -   If $\mathbf{v}$ is in front of the plane ($d_f > 0$):
            -   Identify the previous vertex $\mathbf{v}_{prev}$ in the edge traversal.
            -   Calculate the intersection point $\mathbf{x}_p$ between the edge $(\mathbf{v}_{prev}, \mathbf{v})$ and the clip plane.
            -   Insert a new vertex at $\mathbf{x}_p$ into the graph, connecting it to $\mathbf{v}_{prev}$ and $\mathbf{v}$.
            -   Mark $\mathbf{v}_{prev}$ as clipped.
            -   Compute the signed distances of $\mathbf{x}_p$ to the remaining unclipped faces using weighted averaging (see Pitfalls).
        -   Mark $\mathbf{v}$ as clipped.
    -   The traversal continues until all vertices behind the plane are processed and the graph topology is updated.
3.  **Result**: The remaining graph represents the clipped polyhedron (intersection of voxel and tetrahedron). The graph structure preserves topological validity (faces are loops) [S1, unofficial].

#### 3. Reduction
Integrate the polynomial density field over the Type 1 and Type 3 voxels.
1.  **Type 1 (Inside)**: The voxel is a full axis-aligned cube. Integrate the polynomial analytically over the cube.
2.  **Type 3 (Clipped)**:
    -   **Simplicial Decomposition**: Decompose the clipped polyhedron into tetrahedra.
        -   Fix the origin $\mathbf{0}$ as a common vertex for all tetrahedra.
        -   Traverse the planar graph to identify face loops.
        -   For each face loop, let the vertices be $\mathbf{v}_1, \mathbf{v}_2, \dots, \mathbf{v}_k$ in order.
        -   Form tetrahedra using the origin and each edge of the loop: Tetrahedron $i$ has vertices $\{\mathbf{0}, \mathbf{v}_i, \mathbf{v}_{i+1}\}$ (with $\mathbf{v}_{k+1} = \mathbf{v}_1$).
    -   **Integration**: Compute the integral of the polynomial density over each tetrahedron and sum the results.
    -   **Coordinate Transformation**: The integration formulas are given in barycentric coordinates. Convert Cartesian coordinates $\mathbf{X}$ to barycentric coordinates $\zeta_i$:
        $$ \mathbf{X} = \mathbf{X}_0 \zeta_0 + \mathbf{X}_1 \zeta_1 + \mathbf{X}_2 \zeta_2 + \mathbf{X}_3 \zeta_3 $$
        where $\mathbf{X}_i$ are the tetrahedron vertices.
    -   **Polynomial Integral Formula**: The integral of a monomial $\zeta_0^a \zeta_1^b \zeta_2^c \zeta_3^d$ over a tetrahedron $\Omega_t$ with volume $V$ is:
        $$ \int_{\Omega_t} \zeta_0^a \zeta_1^b \zeta_2^c \zeta_3^d \, d\Omega_t = 6 V \frac{a! b! c! d!}{(a + b + c + d + 3)!} $$
        Substitute the Cartesian expansion into this formula to compute moments like $\int x^2 d\Omega_t$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def signed_distance(vertex, face_point, face_normal):
    """Calculate signed distance from vertex to face."""
    return dot(vertex - face_point, face_normal)

def intersect_edge(v0, d0, v1, d1):
    """
    Calculate intersection point of edge (v0, v1) with plane.
    v0 is in front (d0 > 0), v1 is behind (d1 <= 0).
    """
    # Weighted average formula: xp = (d0*v1 - d1*v0) / (d0 - d1)
    # Denominator d0 - d1 is guaranteed > 0 by strict inequalities
    denom = d0 - d1
    return (d0 * v1 - d1 * v0) / denom

def clip_polyhedron_to_tetrahedron(voxel_vertices, tet_faces):
    """
    Clip a voxel (cube) against the 4 faces of a tetrahedron.
    Returns the planar graph of the resulting polyhedron.
    """
    # Initialize graph with voxel vertices
    graph = initialize_cube_graph(voxel_vertices)
    
    # Precompute signed distances for all vertices to all 4 faces
    # Store in vertex data structure
    for v in graph.vertices:
        v.distances = [signed_distance(v.pos, f.point, f.normal) for f in tet_faces]
        
    # Clip against each face
    for i, face in enumerate(tet_faces):
        # Find a starting vertex behind the plane (dist <= 0)
        start_v = None
        for v in graph.vertices:
            if v.distances[i] <= 0 and not v.clipped:
                start_v = v
                break
        
        if start_v is None:
            # Entire polyhedron is in front of this face; no clipping needed
            continue
            
        # Traverse graph using DFS to clip
        stack = [start_v]
        while stack:
            v = stack.pop()
            if v.clipped:
                continue
            
            # Find neighbors in the planar graph
            neighbors = get_neighbors(v)
            
            for neighbor in neighbors:
                if neighbor.clipped:
                    continue
                    
                d_curr = v.distances[i]
                d_next = neighbor.distances[i]
                
                if d_curr > 0 and d_next <= 0:
                    # Edge crosses plane
                    # Insert new vertex at intersection
                    new_pos = intersect_edge(v.pos, d_curr, neighbor.pos, d_next)
                    new_v = create_vertex(new_pos)
                    
                    # Update graph connectivity: insert new_v between v and neighbor
                    insert_vertex_into_edge(v, neighbor, new_v)
                    
                    # Compute new distances for new_v to remaining faces
                    for j, f in enumerate(tet_faces):
                        if j != i:
                            new_v.distances[j] = signed_distance(new_pos, f.point, f.normal)
                        else:
                            new_v.distances[j] = 0.0 # On the plane
                    
                    # Mark v as clipped (it is now "behind" or on the boundary of the retained region relative to traversal)
                    # Note: The algorithm marks vertices as visited/clipped to avoid reprocessing
                    v.clipped = True 
                    
                elif d_curr <= 0:
                    # Vertex is behind, continue traversal
                    stack.append(neighbor)
                    
                # If d_curr > 0 and d_next > 0, edge is fully in front, no action needed for this edge in this clip pass
                # But we must ensure we traverse all connected components behind the plane
                # The DFS logic above handles connectivity. 
                # If a vertex is in front, we don't traverse *from* it to find new vertices, 
                # but we might need to traverse *to* it if it connects to a behind vertex?
                # The standard Sugihara implementation traverses the boundary of the "behind" region.
                # Simplified logic: traverse all vertices reachable from start_v that are <= 0.
                # When crossing from <=0 to >0, insert intersection.
                
    return graph

def integrate_polyhedron(graph, density_poly):
    """
    Integrate polynomial density over the clipped polyhedron.
    Uses simplicial decomposition (fan from origin).
    """
    total_integral = 0.0
    
    # Decompose into tetrahedra by traversing face loops
    visited_faces = set()
    
    for v in graph.vertices:
        if v.clipped: # In the final graph, 'clipped' might mean 'processed' or 'removed'. 
                      # Assuming graph only contains retained vertices.
            continue
            
        # Traverse face loops
        # A face is a loop of vertices. 
        # For each face, form tetrahedra with origin.
        face_loop = get_face_loop(v)
        if face_loop in visited_faces:
            continue
        visited_faces.add(face_loop)
        
        # Form tetrahedra: (Origin, face_loop[i], face_loop[i+1])
        for i in range(len(face_loop)):
            v1 = face_loop[i]
            v2 = face_loop[(i + 1) % len(face_loop)]
            
            # Tetrahedron vertices: Origin(0,0,0), v1, v2, and a third point?
            # The paper says: "Each edge, the starting vertex, and the origin form a fan of tetrahedra."
            # This implies the tetrahedra are formed by the origin and the triangular faces of the polyhedron.
            # A triangular face is defined by 3 vertices. 
            # If the polyhedron face is a triangle (v_a, v_b, v_c), the tetrahedron is (0, v_a, v_b, v_c).
            # If the face is a polygon, it is triangulated.
            
            # Simplified: Assume face is triangulated or we iterate edges.
            # Let's assume we have a list of triangular faces from the graph traversal.
            pass 
            
    # For each tetrahedron T with vertices X0, X1, X2, X3:
    # 1. Calculate Volume V
    # 2. Expand density polynomial in Cartesian coords
    # 3. Convert to Barycentric coords
    # 4. Apply formula: 6 * V * (a!b!c!d!) / (a+b+c+d+3)!
    
    return total_integral

def calculate_tetrahedron_volume(v0, v1, v2, v3):
    """Volume of tetrahedron with vertices v0, v1, v2, v3."""
    # V = 1/6 * |det(v1-v0, v2-v0, v3-v0)|
    # Or using the determinant formula from the paper:
    # V = 1/6 * det | x0 x1 x2 x3 |
    #               | y0 y1 y2 y3 |
    #               | z0 z1 z2 z3 |
    # Note: The paper's determinant formula likely implies a homogeneous coordinate or specific arrangement.
    # Standard formula:
    M = matrix([v1-v0, v2-v0, v3-v0])
    return abs(det(M)) / 6.0
```

### Parameters
-   **Polynomial Order**: The method supports moment integrals up to quadratic order (e.g., $1, x, y, z, x^2, xy, \dots$) [S1, unofficial].
-   **Constants (2D Demo)**: In the 2D demonstration, $r_0 = 0.45$ (on a grid of side length 1.0), $S_0 = 0.2$, and $T_0 = \pi/2$ [S1, unofficial].

### Edge Cases and Pitfalls
-   **Geometric Degeneracy**: Vertices lying on the clip plane within roundoff error can cause Sutherland-Hodgman to duplicate or omit vertices, creating invalid topology. This method avoids this by using topological consistency (planar graphs) and making geometric decisions combinatorially [S1, unofficial].
-   **Divide-by-Zero**: The intersection formula $\mathbf{x}_p = \frac{d_0 \mathbf{x}_1 - d_1 \mathbf{x}_0}{d_0 - d_1}$ is safe because strict inequalities ($d_0 > 0, d_1 \le 0$) guarantee $d_0 - d_1 > 0$ [S1, unofficial].
-   **Numerical Stability**: The weighted averaging method for intersection points is more stable than parametric forms, especially when vertices are near the clip plane [S1, unofficial].
-   **Boundary Degeneracy**: Near the boundaries of the domain, the old and new meshes may be exactly degenerate (lying on top of each other), which the method handles robustly [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: Brute-force vertex checking scales as $O(g^3)$, where $g$ is the linear grid dimension. Binary space partitioning scales as $O(g^2 \log g)$ [S1, unofficial].
-   **Memory Complexity**: Depends on the number of boundary voxels and the size of the planar graph representation. Each vertex stores coordinates, 3 pointers, 1 byte, and 4 floats [S1, unofficial].
-   **Performance**: BSP is theoretically superior but has higher overhead; brute-force performs better practically on coarse grids [S1, unofficial].
-   **Strengths**: Physically conservative (mass/momentum/energy conservation); robust to geometric degeneracy; supports higher-order polynomial interpolation; exact analytic integration.
-   **Limitations**: Complex implementation due to graph management; higher overhead for BSP on coarse grids; limited to convex polyhedra intersections (tetrahedron vs. cube) [S1, unofficial].
-   **When to use**: When conservation of physical quantities (mass, momentum, energy) is critical, such as in cosmological N-body simulations, plasma physics, or hydrodynamics, and when aliasing noise from standard voxelization is unacceptable [S1, unofficial].
-   **When not to use**: For simple visualization where conservation is not required; for very coarse grids where brute-force is faster and simplicity is preferred; for non-convex input meshes without decomposition [S1, unofficial].

## Background
The method was introduced by Devon Powell and Tom Abel in 2014 in the paper "An exact general remeshing scheme applied to conservative voxelization" [S1, unofficial]. It was developed for the analysis of cosmological N-body data, specifically for dark matter simulations, where global conservation of mass, momentum, and energy is required when solving Poisson equations. The core clipping algorithm is based on ideas by Sugihara (1994) regarding planar graph representations of convex polyhedra to ensure topological consistency. The method has applications in computer graphics, continuum mechanics, and hydrodynamics [S1, unofficial].

## Sources
[S1] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
