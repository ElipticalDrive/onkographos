# Classical Dual Contouring with Estimated Normals

Also called: DC-est

## Overview
Classical Dual Contouring with Estimated Normals (DC-est) is a surface extraction method that generates a mesh from a Signed Distance Field (SDF) by placing a single vertex inside each grid cell intersected by the isosurface, determined by minimizing a Quadratic Error Function (QEF) derived from estimated surface normals. It differs from its siblings by relying on local numerical differentiation of the SDF for normal estimation rather than learned features or topological ray-intersection guarantees, producing results similar to or slightly better than Marching Cubes 33 (MC33) in Chamfer Distance and F1 scores, though with lower normal accuracy near sharp features [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D grid of signed distances (SDF) at resolutions such as $64^3$ or $128^3$ [S1].
- Output: A triangular mesh representing the isosurface (typically $f(x)=0$).

**Data Structures:**
- Grid: A 3D array of floats representing the SDF values.
- Cell Vertex: A 3D position $(x, y, z)$ stored inside each grid cell that contains an isosurface intersection.
- QEF: A system of linear equations $A\mathbf{x} = \mathbf{b}$ for each cell, where $A$ is a $3 \times 3$ matrix and $\mathbf{b}$ is a 3D vector, constructed from surface normals and intersection points.

**Steps:**
1. **Grid Traversal:** Iterate over all grid cells. Identify cells where the isosurface intersects by checking for sign changes in the SDF values at the cell's 8 corners.
2. **Normal Estimation:** For each grid point (corner) of the grid, compute the gradient (normal) by local differentiation over the SDF input [S1]. Let $\nabla f(p)$ be the gradient at grid point $p$.
3. **Intersection Point Calculation:** For each edge of the cell that crosses the isosurface, calculate the exact intersection point $p_i$ using linear interpolation between the corner values.
4. **QEF Construction:** For each intersected cell, construct a QEF based on the estimated normals at the grid points adjacent to the intersection points. For each intersection point $p_i$ with estimated normal $\mathbf{n}_i$, add the constraint $(\mathbf{n}_i \cdot (\mathbf{x} - p_i))^2$ to the error function. This results in a system $A\mathbf{x} = \mathbf{b}$ where:
   $$ A = \sum_i \mathbf{n}_i \mathbf{n}_i^T $$
   $$ \mathbf{b} = \sum_i (\mathbf{n}_i \cdot p_i) \mathbf{n}_i $$
5. **Vertex Placement:** Solve the QEF system for $\mathbf{x}$ to determine the position of the single vertex inside the cell. The solution minimizes the distance to the estimated surface planes defined by the normals and intersection points.
6. **Mesh Generation:** Connect the cell vertices using the standard Dual Contouring topology rules based on the intersection pattern (quadtangents) to form triangles [S1].

**Parameters:**
- Resolution: Typical input resolutions are $64^3$ or $128^3$ [S1].
- Differentiation Method: Local differentiation (e.g., central differences) is used for gradient estimation [S1].

**Edge Cases and Pitfalls:**
- Normal Accuracy: Normal estimation is not expected to be accurate, especially near sharp features [S1].
- Thin Sheets: The method may fail to recover parts of the ground truth shape that are thinner than one voxel, producing artifacts or missing geometry [S1].
- Degenerate Input: If the SDF is not smooth, local differentiation may produce noisy normals, leading to poor QEF solutions.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classical_dual_contouring_with_estimated_normals(sdf_grid, resolution):
    """
    Reconstructs a mesh from an SDF grid using classical dual contouring with estimated normals.
    
    Args:
        sdf_grid: 3D array of floats, shape (resolution, resolution, resolution).
        resolution: Integer, the side length of the cubic grid.
        
    Returns:
        vertices: List of 3D points.
        triangles: List of triplets of vertex indices.
    """
    vertices = []
    triangles = []
    
    # Precompute gradients at all grid points
    gradients = compute_gradients(sdf_grid, resolution)
    
    for x in range(resolution - 1):
        for y in range(resolution - 1):
            for z in range(resolution - 1):
                # Get the 8 corners of the current cell
                corners = [
                    sdf_grid[x, y, z],
                    sdf_grid[x+1, y, z],
                    sdf_grid[x, y+1, z],
                    sdf_grid[x+1, y+1, z],
                    sdf_grid[x, y, z+1],
                    sdf_grid[x+1, y, z+1],
                    sdf_grid[x, y+1, z+1],
                    sdf_grid[x+1, y+1, z+1]
                ]
                
                # Check if the cell intersects the isosurface
                if not intersects_isosurface(corners):
                    continue
                
                # Calculate intersection points and their associated normals
                intersection_points = []
                normals = []
                
                # Iterate over the 12 edges of the cube
                edges = get_cube_edges()
                for edge in edges:
                    p1_idx, p2_idx = edge
                    v1 = corners[p1_idx]
                    v2 = corners[p2_idx]
                    
                    # Linear interpolation to find intersection
                    if (v1 < 0 and v2 > 0) or (v1 > 0 and v2 < 0):
                        t = abs(v1) / (abs(v1) + abs(v2))
                        # Interpolate position
                        p1 = get_corner_position(p1_idx, x, y, z, resolution)
                        p2 = get_corner_position(p2_idx, x, y, z, resolution)
                        p_intersect = lerp(p1, p2, t)
                        
                        # Get normal from precomputed gradients
                        # Note: Which gradient to use? Typically the one at the midpoint or averaged.
                        # Classical DC-est uses local differentiation at grid points.
                        # We need to associate a normal with the intersection point.
                        # A common approach is to average the normals of the two endpoints.
                        n1 = gradients[p1_idx]
                        n2 = gradients[p2_idx]
                        n_intersect = normalize(lerp_vector(n1, n2, t))
                        
                        intersection_points.append(p_intersect)
                        normals.append(n_intersect)
                
                # Construct QEF
                A = zeros((3, 3))
                b = zeros(3)
                
                for p, n in zip(intersection_points, normals):
                    # Add constraint: (n . (x - p))^2
                    # This contributes n*n^T to A and (n.p)*n to b
                    outer_product = outer(n, n)
                    A += outer_product
                    b += dot(n, p) * n
                
                # Solve QEF: A * x = b
                # Use a least-squares solver or direct inversion if A is invertible
                try:
                    vertex = solve_linear_system(A, b)
                except SingularMatrixError:
                    # Fallback: use centroid of intersection points
                    vertex = mean(intersection_points)
                
                # Store vertex
                vertex_index = len(vertices)
                vertices.append(vertex)
                
                # Generate triangles based on the intersection pattern
                # This part depends on the specific DC topology rules
                # For simplicity, we assume a helper function exists
                tri_indices = generate_dc_triangles(intersection_points, vertex_index)
                triangles.extend(tri_indices)
    
    return vertices, triangles

def compute_gradients(sdf_grid, resolution):
    """Compute gradients at all grid points using central differences."""
    gradients = zeros((resolution, resolution, resolution, 3))
    for x in range(1, resolution - 1):
        for y in range(1, resolution - 1):
            for z in range(1, resolution - 1):
                dx = sdf_grid[x+1, y, z] - sdf_grid[x-1, y, z]
                dy = sdf_grid[x, y+1, z] - sdf_grid[x, y-1, z]
                dz = sdf_grid[x, y, z+1] - sdf_grid[x, y, z-1]
                gradients[x, y, z] = [dx, dy, dz]
    # Handle boundaries with one-sided differences or zero
    return gradients

def intersects_isosurface(corners):
    """Check if any corner is positive and any is negative."""
    return any(c > 0 for c in corners) and any(c < 0 for c in corners)

def get_corner_position(idx, x, y, z, resolution):
    """Map corner index to 3D position."""
    # Mapping: 0:(0,0,0), 1:(1,0,0), 2:(0,1,0), 3:(1,1,0), 4:(0,0,1), 5:(1,0,1), 6:(0,1,1), 7:(1,1,1)
    # Relative to cell (x,y,z)
    cx, cy, cz = x, y, z
    if idx == 0: return [cx, cy, cz]
    elif idx == 1: return [cx+1, cy, cz]
    elif idx == 2: return [cx, cy+1, cz]
    elif idx == 3: return [cx+1, cy+1, cz]
    elif idx == 4: return [cx, cy, cz+1]
    elif idx == 5: return [cx+1, cy, cz+1]
    elif idx == 6: return [cx, cy+1, cz+1]
    elif idx == 7: return [cx+1, cy+1, cz+1]

def get_cube_edges():
    """Return list of edge index pairs."""
    return [
        (0,1), (0,2), (0,4), (1,3), (1,5), (2,3), (2,6), (3,7), (4,5), (4,6), (5,7), (6,7)
    ]

def lerp(a, b, t):
    return a + t * (b - a)

def lerp_vector(a, b, t):
    return [a[i] + t * (b[i] - a[i]) for i in range(3)]

def normalize(v):
    mag = sqrt(sum(c*c for c in v))
    if mag == 0: return [0,0,0]
    return [c/mag for c in v]

def outer(a, b):
    return [[a[i]*b[j] for j in range(3)] for i in range(3)]

def dot(a, b):
    return sum(a[i]*b[i] for i in range(3))

def solve_linear_system(A, b):
    # Simple Gaussian elimination or use numpy if allowed
    # For self-containedness, we assume a basic solver
    # This is a placeholder for a real linear algebra solver
    # In practice, one would use a library or a robust implementation
    # Here we just return the mean of b for simplicity if A is singular
    # A real implementation would handle this properly
    return b # Placeholder

def generate_dc_triangles(intersection_points, vertex_index):
    """Generate triangles for the cell. This is a simplified placeholder."""
    # Real DC requires complex topology handling based on the intersection pattern
    # This is not fully specified in the sources, so we leave it as a comment
    # The sources do not provide the exact topology rules for DC-est
    return []
```

## Subtopics

### Dual Contouring Boundary Cap
A technique to make the Dual Contouring (DC) mesher produce a watertight, print-ready solid by capping open boundaries where the surface crosses the sample-cube face, similar to Marching Cubes after boundary sealing [S2, unofficial]. It marches an extra ring of padding cells against a virtual outside shell over the padded cell space $[-1, n]$; for box-face crossings, it places cell vertices flush at the face and connects them using extended edge-quad loops to form the cap geometry [S2, unofficial]. Controlled by a `capBoundary` boolean flag (default true), it acts as a no-op when the surface is strictly interior (all shell corners are outside), producing byte-identical output to the uncapped version [S2, unofficial]. Unlike the original DC implementation which shipped as an interior mesher leaving open boundaries, this method ensures the output is a closed solid suitable for 3D printing [S2, unofficial]. It is used in the Fracturing Fog application's `DualContourMesher.Build` and exported via `UserBulbMeshExporter.ExportDualContouring` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of grid cells, $O(N)$, where $N$ is the total number of cells in the grid. Each cell is processed independently.
- **Memory Complexity:** Linear with respect to the number of grid cells and the number of intersected cells. Storage for the SDF grid, gradients, and the resulting mesh.
- **Measured Performance:** On ABC test set and Thingi10K datasets at resolutions $64^3$ and $128^3$, DC-est produces results similar to or slightly better than MC33 in Chamfer Distance (CD) and F1 scores [S1].
- **Strengths:** Simple to implement, produces watertight meshes (with boundary cap), comparable accuracy to MC33.
- **Limitations:** Normal estimation is inaccurate near sharp features [S1]. May fail to recover thin sheets thinner than one voxel [S1]. Not topologically guaranteed like ray-intersection methods.
- **When to Use:** When a simple, fast surface extraction from SDF is needed and high-fidelity normal estimation is not critical. Suitable for 3D printing with the boundary cap enabled.
- **When Not to Use:** When high accuracy near sharp features is required, or when the input contains thin structures thinner than a voxel.

## Background
Classical Dual Contouring with Estimated Normals (DC-est) is a variant of the Dual Contouring algorithm that uses numerically estimated normals from the SDF. It was compared against Neural Dual Contouring (NDC) and other methods in the context of neural surface reconstruction [S1]. The method builds on the original Dual Contouring algorithm by Chernyaev and Lewiner et al. [S1]. The boundary cap variant was added to the Fracturing Fog application to ensure watertight outputs [S2, unofficial].

## Sources
[S1] Neural Dual Contouring, https://arxiv.org/abs/2202.01999v3, fetched 2026-10-26
[S2] feat(export): boundary cap for dual contouring (roadmap S9, #391), https://github.com/AloneButUnsober/FracturingFog/pull/448, fetched 2026-10-28 (unofficial)
