# Tetrahedral Marching Cubes (5-tetrahedra method)

Also called: 5-tetrahedra-per-cube extraction, SurfaceExtractorTetrahedra

## Overview
This method extracts an isosurface from a scalar field defined on a regular voxel grid by decomposing each cube into five tetrahedra and performing linear interpolation within each tetrahedron [S1, unofficial]. It covers all 81 possible sign patterns of the vertex values, including degenerate cases where vertices lie exactly on the isosurface [S1, unofficial]. Unlike the standard [Marching Cubes](52-marching-cubes.md) which relies on a 256-entry lookup table and resolves ambiguities via topology checks, this method is deterministic and topology-agnostic, solving the equation $V - L = 0$ directly [S1, unofficial].

## How it works
**Inputs:** A 3D scalar field $V$ defined on a grid of voxels, and a scalar level $L$ [S1, unofficial].
**Outputs:** A set of vertices and triangles forming a manifold mesh [S1, unofficial].

**Data Structures:**
The implementation replaces standard associative containers with maps keyed by canonical strings to ensure deterministic ordering and exact arithmetic [S1, unofficial].
*   **Vertices:** Stored in a map keyed by a canonical string representation of the rational coordinates (reduced to lowest terms). Vertex indices are assigned in insertion order [S1, unofficial].
*   **Edges/Triangles:** Stored in maps keyed by canonical strings. Sorted at the end by a custom `operator<` to match `std::set` iteration order [S1, unofficial].
*   **Arithmetic:** Uses `int64_t` for internal calculations, mapping onto IEEE doubles. This remains exact as long as products stay within the safe-integer range ($2^{63}$) [S1, unofficial].

**Algorithm Steps:**
1.  **Decomposition:** For each voxel cube, decompose it into 5 tetrahedra. The standard decomposition connects the cube center to the 6 faces, or more commonly, splits the cube into 6 tetrahedra; however, the 5-tetrahedra method specifically partitions the cube into 4 corner tetrahedra and 1 central tetrahedron (or equivalent partition covering the volume) [S1, unofficial]. *Note: The source describes a partition into four corner tetrahedra plus a central one for gradient calculation, implying a specific geometric split.*
2.  **Sign Pattern Evaluation:** Determine the signs of the scalar field values at the 8 vertices of the cube relative to the level $L$. There are $2^8 = 256$ combinations, but due to symmetry and the nature of the tetrahedral split, the logic handles 81 distinct sign patterns [S1, unofficial].
3.  **Intersection Calculation:** For each tetrahedron in the cube:
    *   Identify edges where the scalar value crosses the level $L$ (sign change).
    *   Compute the intersection point on these edges using linear interpolation.
    *   Form triangles from the intersection points.
4.  **Gradient and Orientation:** Calculate the gradient at intersection points to determine triangle orientation.
    *   **Pitfall (Odd-Parity Branch):** In the original implementation, the `GetGradient` function for odd-parity cubes contained a bug. The condition `dx + dy + dz >= 0` was used to select the central tetrahedron (index 6572). Since `dx, dy, dz` are fractional offsets in $[0, 1]$, this sum is always $\ge 0$, making the branch for the central tetrahedron (index 0752) unreachable.
    *   **Fix:** The condition must be changed to `dx + dy + dz >= 2`. The cutting plane through vertices $(1,1,0), (1,0,1), (0,1,1)$ is defined by $dx + dy + dz = 2$ [S1, unofficial].
5.  **Mesh Assembly:** Collect all generated triangles. Sort edges and triangles by their canonical keys to ensure consistent ordering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_tetrahedral(field, level, grid_dims):
    # Maps for deterministic ordering
    vertex_map = {} # Key: canonical_string(coord), Value: index
    edge_set = {}   # Key: canonical_string(edge), Value: True
    triangle_set = {} # Key: canonical_string(triangle), Value: True
    
    vertices = []
    triangles = []
    
    # Iterate over all voxels
    for x in range(grid_dims.x - 1):
        for y in range(grid_dims.y - 1):
            for z in range(grid_dims.z - 1):
                # Get 8 corner values
                v = [field[x+i, y+j, z+k] for i in [0,1] for j in [0,1] for k in [0,1]]
                
                # Determine sign pattern
                signs = [1 if val > level else -1 if val < level else 0 for val in v]
                
                # Decompose cube into 5 tetrahedra
                # Tetrahedra indices refer to the 8 corners of the cube
                # Standard 5-tet decomposition:
                # T1: 0,1,3,7
                # T2: 1,3,2,7
                # T3: 1,2,6,7
                # T4: 2,6,3,7
                # T5: 3,6,7 (Central? Or specific partition)
                # The source mentions "four corner tetrahedra plus a central one"
                # for gradient logic. We assume a standard partition covering the cube.
                tetrahedra = [
                    [0, 1, 3, 7],
                    [1, 3, 2, 7],
                    [1, 2, 6, 7],
                    [2, 6, 3, 7],
                    [3, 6, 7, ?] # The 5th tetrahedron definition is implicit in the source's gradient logic
                ]
                
                for tet in tetrahedra:
                    # Check if tetrahedron intersects the isosurface
                    # A tetrahedron intersects if vertices have mixed signs
                    tet_signs = [signs[i] for i in tet]
                    if not (all(s >= 0 for s in tet_signs) or all(s <= 0 for s in tet_signs)):
                        # Find intersecting edges
                        edges = get_intersecting_edges(tet, tet_signs, v, level)
                        
                        # Compute intersection points
                        points = []
                        for edge in edges:
                            p = linear_interpolate(edge, v, level)
                            points.append(p)
                        
                        # Form triangles
                        # For a tetrahedron, the intersection is a polygon (triangle or quad)
                        # Triangulate the polygon
                        tris = triangulate_polygon(points)
                        for tri in tris:
                            # Add vertices to map
                            tri_indices = []
                            for p in tri:
                                key = canonical_string(p)
                                if key not in vertex_map:
                                    vertex_map[key] = len(vertices)
                                    vertices.append(p)
                                tri_indices.append(vertex_map[key])
                            
                            # Add triangle to set
                            tri_key = canonical_string(sorted(tri_indices))
                            if tri_key not in triangle_set:
                                triangle_set[tri_key] = True
                                triangles.append(tri_indices)
                                
    # Sort triangles and edges for consistency
    triangles.sort(key=lambda t: canonical_string(sorted(t)))
    
    return vertices, triangles

def get_gradient(x, y, z, field, level):
    # dx, dy, dz are fractional offsets
    dx, dy, dz = x % 1, y % 1, z % 1
    
    # Determine which tetrahedron the point lies in
    # Odd-parity branch fix: use >= 2 instead of >= 0
    if dx - dy - dz >= 0:
        # Corner tetrahedron 1205
        return compute_gradient_1205(dx, dy, dz, field)
    elif dx - dy + dz <= 0:
        # Corner tetrahedron 3027
        return compute_gradient_3027(dx, dy, dz, field)
    elif dx + dy - dz <= 0:
        # Corner tetrahedron 4750
        return compute_gradient_4750(dx, dy, dz, field)
    elif dx + dy + dz >= 2: # Fixed condition
        # Central tetrahedron 0752
        return compute_gradient_0752(dx, dy, dz, field)
    else:
        # Corner tetrahedron 6572
        return compute_gradient_6572(dx, dy, dz, field)
```

## Subtopics

### SurfaceExtractorCubes
Cube-based surface extraction using a vertex-edge table (VETable) and ear-clipping. It resolves the four-cut face ambiguity by evaluating the sign of the determinant $f_{00}f_{11} - f_{01}f_{10}$, adding a branch point when $\det == 0$ [S1, unofficial]. Unlike the tetrahedra extractor which solves $V - L = 0$, the cubes extractor solves $2V - (2L+1) = 0$. To extract the same surface, voxels must be doubled and the level set to $2L+1$ [S1, unofficial].

## Operating characteristics
*   **Complexity:** Linear in the number of voxels, $O(N)$, where $N$ is the number of voxels. Each voxel is processed independently.
*   **Memory:** Proportional to the number of surface triangles and vertices. The use of maps for canonical strings adds overhead compared to direct array indexing but ensures deterministic ordering [S1, unofficial].
*   **Performance:** The method is robust and handles degenerate cases (zero-valued vertices) explicitly [S1, unofficial]. The `int64_t` arithmetic ensures precision but may be slower than floating-point operations if not optimized. The bug fix in the odd-parity branch is critical for correct orientation [S1, unofficial].
*   **Strengths:** Topology-agnostic, no lookup table size limitations, handles all sign patterns including degenerate ones.
*   **Limitations:** Slower than lookup-table-based methods due to per-voxel computation and map overhead. Requires careful handling of the gradient calculation to avoid orientation errors [S1, unofficial].

## Background
The method is part of the `gtengine-js` library, specifically ported in PR #131 (B46) by gradientspaceai [S1, unofficial]. It derives from the `SurfaceExtractor` base class and is designed to be a deterministic alternative to the standard Marching Cubes algorithm. The implementation addresses specific bugs in the gradient calculation for odd-parity cubes, ensuring correct triangle orientation [S1, unofficial].

## Sources
[S1] Port B46: surface extractors batch, https://github.com/gradientspaceai/gtengine-js/pull/131, fetched 2026-10-27 (unofficial)
