# Extended Marching Cubes Lookup Table

Also called: 3-label Marching Cubes, Extended MC Lookup Table

## Overview
The Extended Marching Cubes Lookup Table is a surface extraction method that eliminates degenerate, zero-area triangles by distinguishing scalar values equal to the isovalue from those strictly greater than it [S1, unofficial]. Unlike standard Marching Cubes, which treats equal and greater values identically, this method uses a 3-label system ('+', '-', '=') to ensure robust triangle generation without post-processing [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A 3D scalar field sampled on a regular grid, an isovalue $\sigma$, and a grid cube $\Phi$ (a convex polyhedron, typically a unit cube) with 8 vertices [S1, unofficial].
- **Output**: A set of triangles forming the isosurface patch for the cube [S1, unofficial].

**Data Structures**
- **Vertex Labels**: Each vertex $v_i$ of the cube is assigned a label $L_i \in \{+, -, =\}$ based on its scalar value $f(v_i)$ relative to $\sigma$:
  - $L_i = '+'$ if $f(v_i) > \sigma$
  - $L_i = '-'$ if $f(v_i) < \sigma$
  - $L_i = '='$ if $f(v_i) = \sigma$ [S1, unofficial].
- **Lookup Table**: A table with $3^8 = 6561$ entries, indexed by the 8-tuple of vertex labels $(L_1, \dots, L_8)$ [S1, unofficial].

**Algorithm Steps**
For each cube $\Phi$ in the grid:
1. **Label Assignment**: Determine the label for each of the 8 vertices of $\Phi$ based on the scalar field values and the isovalue [S1, unofficial].
2. **Table Lookup**: Retrieve the precomputed isosurface patch from the extended lookup table using the cube's label configuration [S1, unofficial].
3. **Patch Construction (Convex Hull Method)**: If the lookup table entry is generated dynamically or needs verification, the patch is constructed as follows [S1, unofficial]:
   - Define a set $W_\Phi$ containing:
     - The midpoints of all **bipolar edges** of $\Phi$. An edge is bipolar if one endpoint has label '+' and the other has label '-' [S1, unofficial].
     - All vertices of $\Phi$ labeled '+' or '=' [S1, unofficial].
   - Compute the convex hull $CH = \text{conv}(W_\Phi)$ [S1, unofficial].
   - **Case 1: 3D Convex Hull**: If $CH$ is three-dimensional:
     - Triangulate the boundary $\partial CH$ [S1, unofficial].
     - Return all triangles in this triangulation that do **not** lie on the boundary of the cube $\Phi$ [S1, unofficial].
   - **Case 2: Planar Convex Hull**: If $CH$ is a subset of some facet (face) of $\Phi$:
     - Return a triangulation of $CH$ [S1, unofficial].
   - **Case 3: Otherwise**: If $CH$ is 1D or 2D but not contained in a facet of $\Phi$, return the empty set [S1, unofficial].

**Parameters**
- **Label Set**: $\{+, -, =\}$ [S1, unofficial].
- **Table Size**: 6561 entries [S1, unofficial].

**Pitfalls**
- **Degenerate Triangles**: Standard Marching Cubes creates degenerate triangles when multiple isosurface vertices are mapped to the same grid vertex (e.g., when one vertex is '+' and others are '-', the single '+' vertex causes all triangle vertices to collapse to that point) [S1, unofficial]. The extended table avoids this by treating '=' vertices differently from '+' vertices [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_marching_cubes(grid, isovalue):
    """
    Computes the isosurface mesh for a 3D scalar grid using the Extended Marching Cubes method.
    
    Args:
        grid: 3D list/array of scalar values. Dimensions (nx, ny, nz).
        isovalue: The threshold scalar value.
        
    Returns:
        List of triangles, where each triangle is a tuple of 3 3D points.
    """
    triangles = []
    nx, ny, nz = len(grid), len(grid[0]), len(grid[0][0])
    
    # Iterate over all grid cubes
    for x in range(nx - 1):
        for y in range(ny - 1):
            for z in range(nz - 1):
                # Extract 8 vertices of the current cube
                verts = []
                scalars = []
                for i in range(2):
                    for j in range(2):
                        for k in range(2):
                            verts.append((x + i, y + j, z + k))
                            scalars.append(grid[x + i][y + j][z + k])
                
                # Step 1: Assign labels
                labels = []
                for val in scalars:
                    if val > isovalue:
                        labels.append('+')
                    elif val < isovalue:
                        labels.append('-')
                    else:
                        labels.append('=')
                
                # Step 2: Build set W_phi
                # Bipolar edges: one endpoint '+', one endpoint '-'
                edge_pairs = [
                    (0,1), (1,2), (2,3), (3,0), (0,4), (1,5), (2,6), (3,7),
                    (4,5), (5,6), (6,7), (7,4)
                ]
                W_phi = []
                for i, j in edge_pairs:
                    if (labels[i] == '+' and labels[j] == '-') or (labels[i] == '-' and labels[j] == '+'):
                        mid = [(verts[i][m] + verts[j][m]) / 2.0 for m in range(3)]
                        W_phi.append(mid)
                for i, label in enumerate(labels):
                    if label in ('+', '='):
                        W_phi.append(verts[i])
                
                if not W_phi:
                    continue
                
                # Step 3: Compute convex hull and triangulate
                # Returns list of triangular faces (indices into W_phi)
                hull_faces = _compute_convex_hull_faces(W_phi)
                
                if not hull_faces:
                    continue
                
                # Step 4: Filter triangles based on convex hull dimensionality and cube boundary
                # Check if hull is 3D (has volume)
                is_3d = False
                for face in hull_faces:
                    p0, p1, p2 = W_phi[face[0]], W_phi[face[1]], W_phi[face[2]]
                    # Volume of tetrahedron formed by face and origin (or any interior point)
                    # Simplified: check if points are not coplanar
                    v1 = [p1[m]-p0[m] for m in range(3)]
                    v2 = [p2[m]-p0[m] for m in range(3)]
                    cross = [v1[1]*v2[2]-v1[2]*v2[1], v1[2]*v2[0]-v1[0]*v2[2], v1[0]*v2[1]-v1[1]*v2[0]]
                    if any(abs(c) > 1e-9 for c in cross):
                        is_3d = True
                        break
                
                for face in hull_faces:
                    tri_pts = [W_phi[face[k]] for k in range(3)]
                    
                    if is_3d:
                        # Return triangles not lying on the boundary of the cube Phi
                        if not _on_cube_boundary(tri_pts, verts):
                            triangles.append(tuple(tri_pts))
                    else:
                        # If planar and subset of a facet, return triangulation of CH
                        if _on_cube_boundary(tri_pts, verts):
                            triangles.append(tuple(tri_pts))
                            
    return triangles

def _compute_convex_hull_faces(points):
    """Computes triangular faces of the 3D convex hull using half-space method."""
    n = len(points)
    if n < 4:
        return []
    faces = []
    for i in range(n):
        for j in range(i + 1, n):
            for k in range(j + 1, n):
                v1 = [points[j][m] - points[i][m] for m in range(3)]
                v2 = [points[k][m] - points[i][m] for m in range(3)]
                normal = [v1[1]*v2[2]-v1[2]*v2[1], v1[2]*v2[0]-v1[0]*v2[2], v1[0]*v2[1]-v1[1]*v2[0]]
                all_inside = True
                for p in points:
                    dot = sum((p[m] - points[i][m]) * normal[m] for m in range(3))
                    if dot > 1e-9:
                        all_inside = False
                        break
                if all_inside:
                    faces.append((i, j, k))
    return faces

def _on_cube_boundary(tri_pts, cube_verts):
    """Checks if all three triangle vertices lie on the same cube face plane."""
    # Cube faces are defined by constant coordinates (0 or 1 in local cube space)
    # We check if all points share the same x, y, or z coordinate within tolerance
    for coord in range(3):
        vals = [round(p[coord]) for p in tri_pts]
        if vals[0] == vals[1] == vals[2]:
            return True
    return False
```

## Operating characteristics
- **Time Complexity**: The lookup table access is $O(1)$ per cube [G]. The convex hull construction for each cube operates on at most 8 points, making it constant time, but the geometric overhead is higher than standard MC due to the larger table and hull computations [S1, unofficial].
- **Memory Complexity**: The lookup table requires $3^8 = 6561$ entries, significantly larger than the standard $2^8 = 256$ entries [S1, unofficial].
- **Strengths**: Guarantees no degenerate (zero-area) triangles [S1, unofficial]. Avoids small area, edges, or angles often produced by standard MC [S1, unofficial].
- **Limitations**: Larger lookup table size [S1, unofficial].
- **When to use**: When mesh quality is critical and degenerate triangles are unacceptable, and the scalar field allows for exact or well-defined equality checks [G].

## Background
The method was introduced by S. Raman and R. Wenger in 2008 [S1, unofficial]. It builds on convex hull-based algorithms for generating isosurface lookup tables by Lachaud and Montanvert [LM00] and Bhaniramka et al. [BWC00, BWC04]. The work addresses the limitation of standard Marching Cubes, which can produce degenerate triangles when scalar values equal the isovalue [S1, unofficial].

## Sources
[S1] Extended Marching Cubes, http://www.cse.ohio-state.edu/~wenger/publications/isomesh.pdf, fetched 2026-10-29 (unofficial)
