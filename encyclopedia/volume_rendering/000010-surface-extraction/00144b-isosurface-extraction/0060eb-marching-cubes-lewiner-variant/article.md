# Marching Cubes (Lewiner variant)
Also called: MCB
## Overview
Marching Cubes (Lewiner variant) is an isosurface extraction method that resolves topological ambiguities in the standard Marching Cubes algorithm by performing local interior and face tests on ambiguous cube configurations [S1, unofficial]. It differs from standard implementations by guaranteeing a manifold, non-self-intersecting mesh for all 256 cube cases, whereas standard implementations often require heuristic disambiguation or produce non-manifold geometry in ambiguous cases [S1, unofficial]. It builds on [Isosurface extraction](30-isosurface-extraction.md).

## How it works
**Inputs:** A 3D scalar field defined on a regular grid with dimensions `dim[3]`, processing bounds `lo[3]` and `hi[3]`, and a scalar threshold `isolevel`.
**Outputs:** A triangular mesh consisting of a list of triangles, where each triangle is defined by three vertex indices.

**Data Structures:**
*   **Scalar Grid:** A 3D array of floats storing the scalar values at voxel centers or vertices. The implementation assumes access to values at integer coordinates $(i, j, k)$.
*   **Vertex Cache:** To avoid redundant interpolation, pre-computed vertex positions on shared edges are stored. The implementation uses specific accessors `get_x_vert`, `get_y_vert`, `get_z_vert` and setters `set_x_vert`, `set_y_vert`, `set_z_vert` indexed by edge coordinates.
*   **Triangle:** A struct containing three integers `v1`, `v2`, `v3` representing vertex indices in the output mesh.
*   **Lookup Tables:**
    *   `triTable[256][16]`: Maps the 256 cube configurations (bitmasks of vertex signs) to a sequence of triangle edge indices. Each row contains 16 integers; the first integer indicates the number of triangles (0, 1, or 2), followed by pairs of edge indices.
    *   `test3[24]`: A lookup table for ambiguous cases (Cases 3, 4, 5, 6, 7, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 126, 127, 128, 129, 130, 131, 132, 133, 134, 135, 136, 137, 138, 139, 140, 141, 142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 152, 153, 154, 155, 156, 157, 158, 159, 160, 161, 162, 163, 164, 165, 166, 167, 168, 169, 170, 171, 172, 173, 174, 175, 176, 177, 178, 179, 180, 181, 182, 183, 184, 185, 186, 187, 188, 189, 190, 191, 192, 193, 194, 195, 196, 197, 198, 199, 200, 201, 202, 203, 204, 205, 206, 207, 208, 209, 210, 211, 212, 213, 214, 215, 216, 217, 218, 219, 220, 221, 222, 223, 224, 225, 226, 227, 228, 229, 230, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 242, 243, 244, 245, 246, 247, 248, 249, 250, 251, 252, 253, 254, 255) that identifies which face to test for ambiguity resolution. The values correspond to specific ambiguous configurations (e.g., 5, 1, 4, etc., representing face indices or test types).
    *   `tilingTable`: Contains the specific triangulation edge indices for the ambiguous cases, dependent on the result of the face test.

**Algorithm Steps:**

1.  **Initialization:** Initialize an empty list for triangles. Initialize the vertex cache for edge intersections.
2.  **Volume Traversal:** Iterate through each unit cube in the grid from `lo` to `hi`. Let the cube be defined by corner $(i, j, k)$.
3.  **Configuration Evaluation:** For the current cube, evaluate the sign of the scalar value $S$ at each of its 8 vertices relative to `isolevel`. Construct a 8-bit configuration index `cfg` where the $n$-th bit is 1 if $S_n > isolevel$ and 0 otherwise. The vertex ordering follows the standard Marching Cubes convention (e.g., vertex 0 at $(i,j,k)$, vertex 1 at $(i+1,j,k)$, etc.).
4.  **Lookup:** Retrieve the triangle count and edge indices from `triTable[cfg]`.
5.  **Ambiguity Resolution:** If `cfg` corresponds to an ambiguous case (Cases 3, 4, 5, 6, 7, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 126, 127, 128, 129, 130, 131, 132, 133, 134, 135, 136, 137, 138, 139, 140, 141, 142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 152, 153, 154, 155, 156, 157, 158, 159, 160, 161, 162, 163, 164, 165, 166, 167, 168, 169, 170, 171, 172, 173, 174, 175, 176, 177, 178, 179, 180, 181, 182, 183, 184, 185, 186, 187, 188, 189, 190, 191, 192, 193, 194, 195, 196, 197, 198, 199, 200, 201, 202, 203, 204, 205, 206, 207, 208, 209, 210, 211, 212, 213, 214, 215, 216, 217, 218, 219, 220, 221, 222, 223, 224, 225, 226, 227, 228, 229, 230, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 242, 243, 244, 245, 246, 247, 248, 249, 250, 251, 252, 253, 254, 255), perform the following:
    a.  Consult `test3` to determine which face to test.
    b.  Call `test_face` or `test_interior` to evaluate the scalar field's behavior on that face or interior.
    c.  Select the appropriate triangulation from `tilingTable` based on the test result.
6.  **Intersection Computation:** For each edge index in the selected triangulation:
    a.  Identify the two vertices of the cube connected by the edge.
    b.  Compute the intersection point using linear interpolation:
        $$ P = V_1 + \frac{isolevel - S_1}{S_2 - S_1} (V_2 - V_1) $$
        where $V_1, V_2$ are the cube vertex positions and $S_1, S_2$ are their scalar values.
    c.  Store the intersection point in the vertex cache using the appropriate axis-specific setter (`set_x_vert`, `set_y_vert`, `set_z_vert`) to ensure shared edges between adjacent cubes yield identical vertex positions.
7.  **Triangle Generation:** For each triangle in the triangulation, create a `Triangle` struct with the indices of the three computed intersection points.
8.  **Mesh Assembly:** Add the triangle to the output mesh list.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def run(mcb):
    """Main entry point for Marching Cubes Lewiner variant."""
    triangles = []
    # Initialize vertex cache
    vertex_cache = {} 
    
    for k in range(mcb.lo[2], mcb.hi[2]):
        for j in range(mcb.lo[1], mcb.hi[1]):
            for i in range(mcb.lo[0], mcb.hi[0]):
                # 1. Evaluate configuration
                cfg = 0
                for v in range(8):
                    # Get scalar value at vertex v of cube (i,j,k)
                    s = get_scalar(mcb, i, j, k, v)
                    if s > mcb.isolevel:
                        cfg |= (1 << v)
                
                # 2. Process cube
                process_cube(mcb, i, j, k, cfg, triangles, vertex_cache)
    
    return triangles

def process_cube(mcb, i, j, k, cfg, triangles, vertex_cache):
    """Process a single unit cube."""
    # 3. Check for ambiguity
    if is_ambiguous_case(cfg):
        # 4. Resolve ambiguity
        face_to_test = test3[cfg]
        test_result = test_face(mcb, i, j, k, face_to_test)
        triangulation = get_tiling_table_entry(cfg, test_result)
    else:
        # 5. Standard lookup
        triangulation = triTable[cfg]
    
    # 6. Generate triangles
    num_triangles = triangulation[0]
    for t in range(num_triangles):
        edge_indices = triangulation[1 + 3*t : 1 + 3*t + 3]
        v1 = get_intersection(mcb, i, j, k, edge_indices[0], vertex_cache)
        v2 = get_intersection(mcb, i, j, k, edge_indices[1], vertex_cache)
        v3 = get_intersection(mcb, i, j, k, edge_indices[2], vertex_cache)
        triangles.append(Triangle(v1, v2, v3))

def get_intersection(mcb, i, j, k, edge_idx, vertex_cache):
    """Compute or retrieve intersection point for a given edge."""
    # Determine edge vertices and cache key
    edge_key = get_edge_key(i, j, k, edge_idx)
    
    if edge_key in vertex_cache:
        return vertex_cache[edge_key]
    
    # Linear interpolation
    v1_pos, v2_pos = get_edge_vertices(edge_idx)
    s1 = get_scalar(mcb, i, j, k, v1_pos)
    s2 = get_scalar(mcb, i, j, k, v2_pos)
    
    if abs(s2 - s1) < 1e-9:
        # Avoid division by zero; midpoint
        t = 0.5
    else:
        t = (mcb.isolevel - s1) / (s2 - s1)
    
    point = lerp(v1_pos, v2_pos, t)
    vertex_cache[edge_key] = point
    return point

def is_ambiguous_case(cfg):
    """Check if configuration is one of the ambiguous cases."""
    # Cases 3, 4, 5, 6, 7, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 126, 127, 128, 129, 130, 131, 132, 133, 134, 135, 136, 137, 138, 139, 140, 141, 142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 152, 153, 154, 155, 156, 157, 158, 159, 160, 161, 162, 163, 164, 165, 166, 167, 168, 169, 170, 171, 172, 173, 174, 175, 176, 177, 178, 179, 180, 181, 182, 183, 184, 185, 186, 187, 188, 189, 190, 191, 192, 193, 194, 195, 196, 197, 198, 199, 200, 201, 202, 203, 204, 205, 206, 207, 208, 209, 210, 211, 212, 213, 214, 215, 216, 217, 218, 219, 220, 221, 222, 223, 224, 225, 226, 227, 228, 229, 230, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 242, 243, 244, 245, 246, 247, 248, 249, 250, 251, 252, 253, 254, 255
    ambiguous_cases = {3, 4, 5, 6, 7, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 70, 71, 72, 73, 74, 75, 76, 77, 78, 79, 80, 81, 82, 83, 84, 85, 86, 87, 88, 89, 90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100, 101, 102, 103, 104, 105, 106, 107, 108, 109, 110, 111, 112, 113, 114, 115, 116, 117, 118, 119, 120, 121, 122, 123, 124, 125, 126, 127, 128, 129, 130, 131, 132, 133, 134, 135, 136, 137, 138, 139, 140, 141, 142, 143, 144, 145, 146, 147, 148, 149, 150, 151, 152, 153, 154, 155, 156, 157, 158, 159, 160, 161, 162, 163, 164, 165, 166, 167, 168, 169, 170, 171, 172, 173, 174, 175, 176, 177, 178, 179, 180, 181, 182, 183, 184, 185, 186, 187, 188, 189, 190, 191, 192, 193, 194, 195, 196, 197, 198, 199, 200, 201, 202, 203, 204, 205, 206, 207, 208, 209, 210, 211, 212, 213, 214, 215, 216, 217, 218, 219, 220, 221, 222, 223, 224, 225, 226, 227, 228, 229, 230, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 242, 243, 244, 245, 246, 247, 248, 249, 250, 251, 252, 253, 254, 255}
    return cfg in ambiguous_cases

def test_face(mcb, i, j, k, face_idx):
    """Test the interior of a specific face of the cube."""
    # Implementation details for face testing depend on the specific ambiguous case
    # This is a placeholder for the actual logic defined in the source
    # Typically involves checking the scalar value at the center of the face
    # or along a diagonal
    pass

def get_tiling_table_entry(cfg, test_result):
    """Retrieve the correct triangulation for an ambiguous case."""
    # Implementation details for retrieving the tiling table entry
    pass
```

**Parameters:**
*   `isolevel` (float): The scalar threshold value. Vertices are interpolated where data crosses this value.
*   `dim[3]` (int): Volume dimensions.
*   `lo[3]` (int): Lower processing bounds.
*   `hi[3]` (int): Upper processing bounds.

**Edge Cases and Pitfalls:**
*   **Precision:** Linear interpolation can suffer from precision issues if scalar values are very close to `isolevel`. A small epsilon should be used to avoid division by zero.
*   **Boundary Conditions:** The algorithm assumes the scalar field is defined for all vertices in the range `[lo, hi]`. Accessing out-of-bounds vertices must be handled (e.g., by clamping or assuming zero).
*   **Degenerate Input:** If the scalar field is constant, no intersections are generated. If the field has flat regions exactly at `isolevel`, the sign test may be ambiguous; a consistent tie-breaking rule (e.g., `>=` vs `>`) must be chosen.

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the number of cubes in the processing volume. Each cube is processed in constant time.
**Memory Complexity:** $O(N)$ for the output mesh. The vertex cache size is proportional to the number of unique edges, which is also $O(N)$.
**Measured Performance:** Not given in the sources.
**Scaling:** Scales linearly with volume size. Parallelization is straightforward as cubes are independent.
**Strengths:** Guarantees manifold output for all cases. Simple to implement.
**Limitations:** Requires lookup tables. Ambiguity resolution adds computational overhead compared to standard Marching Cubes.

## Background
The Marching Cubes (Lewiner variant) was developed by Thomas Lewiner at the Department of Mathematics, PUC-Rio. Version 0.2 was released on 12/08/2002. A C translation was performed by Ziad S. Saad on November 30, 2004 [S1, unofficial]. It is used in medical imaging and scientific visualization for robust surface extraction.

## Sources
[S1] neurolabusc/nii2mesh, https://github.com/neurolabusc/nii2mesh, fetched 2026-10-26 (unofficial)
