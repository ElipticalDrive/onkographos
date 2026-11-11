# Exact General Remeshing Scheme for Conservative Voxelization

Also called: 2D Conservative Remeshing, 2D Conservative Voxelization, Conservative Voxelization of Tetrahedra, Direct remap, Exact general remeshing scheme, Physically Conservative Voxelization

## Overview
This method voxelizes a single tetrahedron onto a Cartesian grid by computing physically conservative moments (volume, linear, quadratic) for a specified polynomial order [S1, unofficial]. It ensures that the total mass in each grid cell exactly equals the integral of the input density field over the cell, avoiding aliasing noise [S1, unofficial]. Unlike standard rasterization or Sutherland-Hodgman clipping, which can fail due to geometric degeneracy and roundoff errors, this scheme uses planar graph representations to maintain topological consistency during clipping, guaranteeing robustness even when vertices lie on clip planes [S1, unofficial].

## How it works
The algorithm computes the intersection of a tetrahedron with a Cartesian grid cell (voxel) and integrates a polynomial density field over this intersection [S1, unofficial]. It is divided into three phases: Searching, Clipping, and Reduction [S1, unofficial].

### Inputs and Outputs
- **Input**: A tetrahedron defined by four vertices $\mathbf{x}_0, \mathbf{x}_1, \mathbf{x}_2, \mathbf{x}_3$; a Cartesian grid cell (voxel) defined by its eight vertices; a polynomial density field defined on the tetrahedron (typically up to quadratic order) [S1, unofficial].
- **Output**: The conservative moments (integrals of $1, x, y, z, x^2, xy, \dots$) of the density field over the voxel [S1, unofficial].

### Data Structures
**Polyhedron Vertex**:
Each vertex in the planar graph representation contains:
1. Spatial coordinates $(x, y, z)$ [S1, unofficial].
2. Three pointers to neighboring vertices (forming the edges of the planar graph) [S1, unofficial].
3. A byte flag indicating if the vertex has been clipped [S1, unofficial].
4. Four floating-point numbers storing the signed distance to each of the tetrahedron's four faces [S1, unofficial].

**Tetrahedron Faces**:
Each face $f$ is defined by a vertex $\mathbf{x}_f$ and a unit normal $\mathbf{n}_f$ [S1, unofficial]. The signed distance of a point $\mathbf{x}$ to face $f$ is:
$$ d_f = (\mathbf{x} - \mathbf{x}_f) \cdot \mathbf{n}_f $$
Points with $d_f \le 0$ are "behind" or "outside" the face; points with $d_f > 0$ are "inside" or "in front" [S1, unofficial].

### Steps

#### 1. Searching
Determine which voxels intersect the tetrahedron boundary [S1, unofficial].
1. **Classification**: For each voxel, check the signed distances of its 8 vertices to all 4 faces of the tetrahedron [S1, unofficial].
    - **Type 1 (Outside)**: All 8 vertices have $d_f \le 0$ for at least one face $f$. Ignore these voxels [S1, unofficial].
    - **Type 2 (Inside)**: All 8 vertices have $d_f > 0$ for all 4 faces. These are fully contained [S1, unofficial].
    - **Type 3 (Boundary)**: Voxels that are neither Type 1 nor Type 2. These require clipping [S1, unofficial].
2. **Optimization**: Use Binary Space Partitioning (BSP) to find Type 3 voxels efficiently [S1, unofficial].
    - Recursively split the grid bounding box along its longest dimension [S1, unofficial].
    - Stop recursion if:
        1. All 8 vertices of the box are outside one face (ignore box) [S1, unofficial].
        2. All 8 vertices are inside all faces (process box with a loop for Type 2 voxels) [S1, unofficial].
        3. The box contains a single voxel (process as Type 3) [S1, unofficial].
    - Complexity: $O(g^2 \log g)$ vs $O(g^3)$ for brute force, where $g$ is the linear grid dimension [S1, unofficial].

#### 2. Clipping
Construct the convex polyhedron resulting from the intersection of the voxel (cube) and the tetrahedron [S1, unofficial].
1. **Initialization**: Initialize the planar graph of the cubical voxel [S1, unofficial]. Compute signed distances for all 8 vertices to the 4 tetrahedron faces [S1, unofficial].
2. **Iterative Clipping**: For each of the 4 faces of the tetrahedron:
    - Find a vertex in the current graph with $d_f \le 0$ (behind the plane) [S1, unofficial]. If none exist, the polyhedron is entirely in front; skip this face [S1, unofficial].
    - Perform a Depth-First Search (DFS) traversal starting from this vertex [S1, unofficial].
    - For each vertex $\mathbf{v}$ visited in the traversal:
        - If $\mathbf{v}$ is already marked clipped, continue [S1, unofficial].
        - If $\mathbf{v}$ is in front of the plane ($d_f > 0$):
            - Identify the previous vertex $\mathbf{v}_{prev}$ in the edge traversal [S1, unofficial].
            - Calculate the intersection point $\mathbf{x}_p$ between the edge $(\mathbf{v}_{prev}, \mathbf{v})$ and the clip plane [S1, unofficial].
            - Insert a new vertex at $\mathbf{x}_p$ into the graph, connecting it to $\mathbf{v}_{prev}$ and $\mathbf{v}$ [S1, unofficial].
            - Mark $\mathbf{v}_{prev}$ as clipped [S1, unofficial].
            - Compute the signed distances of $\mathbf{x}_p$ to the remaining unclipped faces using weighted averaging (see Pitfalls) [S1, unofficial].
        - Mark $\mathbf{v}$ as clipped [S1, unofficial].
    - The traversal continues until all vertices behind the plane are processed and the graph topology is updated [S1, unofficial].
3. **Result**: The remaining graph represents the clipped polyhedron (intersection of voxel and tetrahedron). The graph structure preserves topological validity (faces are loops) [S1, unofficial].

#### 3. Reduction
Integrate the polynomial density field over the Type 1 and Type 3 voxels [S1, unofficial].
1. **Type 1 (Inside)**: The voxel is a full axis-aligned cube. Integrate the polynomial analytically over the cube [S1, unofficial].
2. **Type 3 (Clipped)**:
    - **Simplicial Decomposition**: Decompose the clipped polyhedron into tetrahedra [S1, unofficial].
        - Fix the origin $\mathbf{0}$ as a common vertex for all tetrahedra [S1, unofficial].
        - Traverse the planar graph to identify face loops [S1, unofficial].
        - For each face loop, let the vertices be $\mathbf{v}_1, \mathbf{v}_2, \dots, \mathbf{v}_k$ in order [S1, unofficial].
        - Form tetrahedra using the origin and each edge of the loop: Tetrahedron $i$ has vertices $\{\mathbf{0}, \mathbf{v}_1, \mathbf{v}_i, \mathbf{v}_{i+1}\}$ (with $\mathbf{v}_{k+1} = \mathbf{v}_2$) [S1, unofficial].
    - **Integration**: Compute the integral of the polynomial density over each tetrahedron and sum the results [S1, unofficial].
    - **Coordinate Transformation**: The integration formulas are given in barycentric coordinates. Convert Cartesian coordinates $\mathbf{X}$ to barycentric coordinates $\zeta_i$:
        $$ \mathbf{X} = \mathbf{X}_0 \zeta_0 + \mathbf{X}_1 \zeta_1 + \mathbf{X}_2 \zeta_2 + \mathbf{X}_3 \zeta_3 $$
        where $\mathbf{X}_i$ are the tetrahedron vertices [S1, unofficial].
    - **Polynomial Integral Formula**: The integral of a monomial $\zeta_0^a \zeta_1^b \zeta_2^c \zeta_3^d$ over a tetrahedron $\Omega_t$ with volume $V$ is:
        $$ \int_{\Omega_t} \zeta_0^a \zeta_1^b \zeta_2^c \zeta_3^d \, d\Omega_t = 6 V \frac{a! b! c! d!}{(a + b + c + d + 3)!} $$
        Substitute the Cartesian expansion into this formula to compute moments like $\int x^2 d\Omega_t$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def signed_distance(vertex, face_point, face_normal):
    """Calculate signed distance from vertex to face."""
    return sum((vertex[i] - face_point[i]) * face_normal[i] for i in range(3))

def intersect_edge(v0, d0, v1, d1):
    """
    Calculate intersection point of edge (v0, v1) with plane.
    v0 is in front (d0 > 0), v1 is behind (d1 <= 0).
    """
    denom = d0 - d1
    return tuple((d0 * v1[i] - d1 * v0[i]) / denom for i in range(3))

def tetrahedron_volume(v0, v1, v2, v3):
    """Volume of tetrahedron with vertices v0, v1, v2, v3."""
    m = [[v1[i]-v0[i], v2[i]-v0[i], v3[i]-v0[i]] for i in range(3)]
    det = (m[0][0] * (m[1][1]*m[2][2] - m[1][2]*m[2][1]) -
           m[0][1] * (m[1][0]*m[2][2] - m[1][2]*m[2][0]) +
           m[0][2] * (m[1][0]*m[2][1] - m[1][1]*m[2][0]))
    return abs(det) / 6.0

def barycentric_integral(V, a, b, c, d):
    """Integral of barycentric monomial over tetrahedron."""
    return 6.0 * V * math.factorial(a) * math.factorial(b) * math.factorial(c) * math.factorial(d) / math.factorial(a + b + c + d + 3)

def clip_polyhedron(vertices, tet_faces):
    """
    Clip a voxel (cube) against the 4 faces of a tetrahedron.
    Returns the planar graph of the resulting polyhedron.
    """
    nodes = []
    for v in vertices:
        nodes.append({'pos': v, 'neighbors': [], 'clipped': False, 'distances': []})
    
    cube_edges = [(0,1), (1,2), (2,3), (3,0), (4,5), (5,6), (6,7), (7,4),
                  (0,4), (1,5), (2,6), (3,7)]
    for i, j in cube_edges:
        nodes[i]['neighbors'].append(nodes[j])
        nodes[j]['neighbors'].append(nodes[i])
        
    for node in nodes:
        node['distances'] = [signed_distance(node['pos'], f['point'], f['normal']) for f in tet_faces]
        
    for i, face in enumerate(tet_faces):
        start_node = None
        for node in nodes:
            if node['distances'][i] <= 0 and not node['clipped']:
                start_node = node
                break
        
        if start_node is None:
            continue
            
        stack = [start_node]
        while stack:
            curr = stack.pop()
            if curr['clipped']:
                continue
            curr['clipped'] = True
            d_curr = curr['distances'][i]
            
            for neighbor in curr['neighbors']:
                if neighbor['clipped']:
                    continue
                d_next = neighbor['distances'][i]
                
                if d_curr > 0 and d_next <= 0:
                    new_pos = intersect_edge(curr['pos'], d_curr, neighbor['pos'], d_next)
                    new_node = {'pos': new_pos, 'neighbors': [], 'clipped': False, 'distances': []}
                    
                    curr['neighbors'].remove(neighbor)
                    neighbor['neighbors'].remove(curr)
                    curr['neighbors'].append(new_node)
                    neighbor['neighbors'].append(new_node)
                    new_node['neighbors'].extend([curr, neighbor])
                    
                    new_node['distances'] = [signed_distance(new_pos, f['point'], f['normal']) for f in tet_faces]
                    new_node['distances'][i] = 0.0
                    
                    curr['clipped'] = True
                elif d_curr <= 0:
                    stack.append(neighbor)
                    
    return [n for n in nodes if not n['clipped']]

def integrate_polyhedron(clipped_nodes):
    """
    Integrate polynomial density over the clipped polyhedron.
    Uses simplicial decomposition (fan from origin).
    """
    total_integral = 0.0
    visited_edges = set()
    face_loops = []
    
    for node in clipped_nodes:
        for neighbor in node['neighbors']:
            edge = (id(node), id(neighbor))
            if edge in visited_edges:
                continue
            visited_edges.add(edge)
            
            loop = [node]
            curr = neighbor
            while curr != node:
                loop.append(curr)
                visited_edges.add((id(curr), curr['neighbors'][0]))
                curr = curr['neighbors'][0]
            face_loops.append(loop)
            
    origin = (0.0, 0.0, 0.0)
    for loop in face_loops:
        v1 = loop[0]['pos']
        for i in range(1, len(loop) - 1):
            v2 = loop[i]['pos']
            v3 = loop[i+1]['pos']
            V = tetrahedron_volume(origin, v1, v2, v3)
            total_integral += barycentric_integral(V, 0, 0, 0, 0)
            
    return total_integral
```

### Parameters
- **Polynomial Order**: The method supports moment integrals up to quadratic order (e.g., $1, x, y, z, x^2, xy, \dots$) [S1, unofficial].
- **Constants (2D Demo)**: In the 2D demonstration, $r_0 = 0.45$ (on a grid of side length 1.0), $S_0 = 0.2$, and $T_0 = \pi/2$ [S1, unofficial].

### Edge Cases and Pitfalls
- **Geometric Degeneracy**: Vertices lying on the clip plane within roundoff error can cause Sutherland-Hodgman to duplicate or omit vertices, creating invalid topology. This method avoids this by using topological consistency (planar graphs) and making geometric decisions combinatorially [S1, unofficial].
- **Divide-by-Zero**: The intersection formula $\mathbf{x}_p = \frac{d_0 \mathbf{x}_1 - d_1 \mathbf{x}_0}{d_0 - d_1}$ is safe because strict inequalities ($d_0 > 0, d_1 \le 0$) guarantee $d_0 - d_1 > 0$ [S1, unofficial].
- **Numerical Stability**: The weighted averaging method for intersection points is more stable than parametric forms, especially when vertices are near the clip plane [S1, unofficial].
- **Boundary Degeneracy**: Near the boundaries of the domain, the old and new meshes may be exactly degenerate (lying on top of each other), which the method handles robustly [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Brute-force vertex checking scales as $O(g^3)$, where $g$ is the linear grid dimension. Binary space partitioning scales as $O(g^2 \log g)$ [S1, unofficial].
- **Memory Complexity**: Depends on the number of boundary voxels and the size of the planar graph representation. Each vertex stores coordinates, 3 pointers, 1 byte, and 4 floats [S1, unofficial].
- **Performance**: BSP is theoretically superior but has higher overhead; brute-force performs better practically on coarse grids [S1, unofficial].
- **Strengths**: Physically conservative (mass/momentum/energy conservation); robust to geometric degeneracy; supports higher-order polynomial interpolation; exact analytic integration [S1, unofficial].
- **Limitations**: Complex implementation due to graph management; higher overhead for BSP on coarse grids; limited to convex polyhedra intersections (tetrahedron vs. cube) [S1, unofficial].
- **When to use**: When conservation of physical quantities (mass, momentum, energy) is critical, such as in cosmological N-body simulations, plasma physics, or hydrodynamics, and when aliasing noise from standard voxelization is unacceptable [S1, unofficial].

## Background
The method was introduced by Devon Powell and Tom Abel in 2014 in the paper "An exact general remeshing scheme applied to conservative voxelization" [S1, unofficial]. It was developed for the analysis of cosmological N-body data, specifically for dark matter simulations, where global conservation of mass, momentum, and energy is required when solving Poisson equations [S1, unofficial]. The core clipping algorithm is based on ideas by Sugihara (1994) regarding planar graph representations of convex polyhedra to ensure topological consistency [S1, unofficial]. The method has applications in computer graphics, continuum mechanics, and hydrodynamics [S1, unofficial].

## Sources
[S1] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
