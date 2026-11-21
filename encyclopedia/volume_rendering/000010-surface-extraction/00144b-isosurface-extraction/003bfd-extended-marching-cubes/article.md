# Extended Marching Cubes

Also called: Feature Sensitive Surface Extraction

## Overview
Extended Marching Cubes (EMC) is a surface extraction algorithm that enhances the standard Marching Cubes method by preserving sharp geometric features (edges and corners) that are typically smoothed out by linear interpolation [S1][S3, unofficial]. It achieves this by computing surface normals at intersection points, detecting sharp angles between these normals, and using Quadratic Error Functions (QEFs) to place new vertices at the precise locations of these features [S1][S3, unofficial]. Unlike standard Marching Cubes, which connects edge intersections directly, EMC inserts feature points and forms triangle fans around them to maintain sharpness [S1].

## How it works
### Inputs and Outputs
- **Input**: A 3D scalar field (volume data) defined on a grid. The field function $f(x,y,z)$ returns a scalar value [S3, unofficial]. EMC variants may modify the field evaluation to return a 3D vector representing distances along axes [S3, unofficial].
- **Output**: A polygonal mesh (vertices and indices) representing the isosurface [S1].

### Data Structures
- **Grid Key**: Identifies the voxel cell being processed [S2, unofficial].
- **Vertices**: Array of 3D points. Some vertices are associated with grid keys (standard edge intersections), while feature vertices are not [S2, unofficial].
- **Normals**: Array of 3D unit vectors corresponding to vertices [S2, unofficial].
- **Features**: A set (e.g., `HashSet`) of indices identifying vertices that are feature points [S2, unofficial].
- **Tangent Planes**: For each edge crossing, a plane is defined by the intersection point and the surface normal at that point [S1][S2, unofficial].

### Step-by-Step Method

1.  **Traverse the Grid**: Iterate over each voxel cell in the primal grid [S2, unofficial].
2.  **Classify Cell**: Determine the `cube_index` (0-255) by comparing the scalar values at the 8 corners of the voxel against the isolevel [S2, unofficial].
3.  **Find Edge Crossings**: Identify which edges of the voxel intersect the isosurface. For each intersecting edge, compute the intersection point $P$ using linear interpolation [S3, unofficial]. Some implementations modify the field function to return a 3D distance vector $(d_x, d_y, d_z)$ and compute the intersection coordinates independently for each axis using linear interpolation on the respective components [S3, unofficial].
4.  **Sample Normals**: Compute the normal vector $\mathbf{n}$ at each intersection point $P$. This is typically approximated as the gradient of the scalar field:
    $$
    \mathbf{n} = \left( \frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z} \right)
    $$
    Numerically, this is estimated using forward differences with a small step size $E$ [S3, unofficial]:
    $$
    n_x = f(x + E, y, z) - f(x, y, z)
    $$
    $$
    n_y = f(x, y + E, z) - f(x, y, z)
    $$
    $$
    n_z = f(x, y, z + E) - f(x, y, z)
    $$
    Normalize $\mathbf{n}$ to unit length [S3, unofficial].
5.  **Detect Sharp Features**: For the current cell, examine the normals of the intersection points. A sharp feature is detected if the angle between adjacent intersection normals is below a threshold [S3, unofficial]. This is computed using the dot product:
    $$
    \cos(\theta) = \mathbf{n}_i \cdot \mathbf{n}_j
    $$
    If $\theta < \theta_{threshold}$ (e.g., 90 degrees, so $\cos(\theta) > 0$), a sharp feature exists [S3, unofficial].
6.  **Construct Tangent Planes**: For each intersection point $P_i$ with normal $\mathbf{n}_i$, define a tangent plane $\Pi_i$ passing through $P_i$ with normal $\mathbf{n}_i$ [S2, unofficial].
7.  **Solve for Feature Point (QEF)**: If a sharp feature is detected, solve for a new vertex position $V_{feat}$ that minimizes the distance to all relevant tangent planes. This is done by minimizing the Quadratic Error Function (QEF):
    $$
    E(V) = \sum_i (\mathbf{n}_i \cdot (V - P_i))^2
    $$
    This is a least-squares problem. The solution $V_{feat}$ is the point inside the voxel that best satisfies the plane constraints [S1][S2, unofficial].
    - *Pitfall*: If fewer than three planes are available (e.g., only 1 or 2 intersections define the feature), the system is underdetermined [S1]. The QEF minimization still yields a least-squares solution [S1].
8.  **Add Feature Vertex**: Insert $V_{feat}$ into the mesh vertex list. Mark it as a feature vertex (no associated grid key) and add its index to the `features` set [S2, unofficial].
9.  **Generate Triangles (Triangle Fan)**: For each loop of edge intersections in the cell (determined by the `cube_index`), create a triangle fan using the feature point as the center [S1][S2, unofficial].
    - Let the edge intersection points in the loop be $E_0, E_1, \dots, E_{k-1}$ [S2, unofficial].
    - For each $i$ from $0$ to $k-1$:
        - Let $j = (i + 1) \pmod k$.
        - Add triangles $(V_{feat}, E_i, E_j)$ [S2, unofficial].
10. **Post-Processing (Edge Flipping)**: After processing all cells, iterate over all edges in the generated mesh. If an edge connects two feature vertices, it may need to be "flipped" or rotated to ensure correct connectivity and avoid artifacts [S2, unofficial].
    - *Pitfall*: This step may require information from neighboring chunks or cells, introducing dependencies [S2, unofficial].
    - *Pitfall*: The algorithm may produce small triangle slivers on sharp edges [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def extended_marching_cubes(volume, isolevel, threshold_angle=90.0):
    """
    volume: 3D numpy array or list of lists of lists of scalar values.
    isolevel: scalar value for isosurface extraction.
    threshold_angle: angle in degrees below which a feature is considered sharp.
    """
    mesh_vertices = []
    mesh_indices = []
    feature_vertices = set()
    
    # Standard MC edge table (bitmask of crossed edges for each of 256 cases)
    EDGE_TABLE = [...] # Precomputed 256-entry array
    # Edge loops table: maps cube_index to list of loops, each loop is list of edge indices
    EDGE_LOOPS = [...] # Precomputed 256-entry array
    
    # Standard edge vertex indices (0-7)
    EDGE_VERTICES = [
        (0, 1), (1, 2), (2, 3), (3, 0),
        (4, 5), (5, 6), (6, 7), (7, 4),
        (0, 4), (1, 5), (2, 6), (3, 7)
    ]
    
    for x in range(volume.shape[0] - 1):
        for y in range(volume.shape[1] - 1):
            for z in range(volume.shape[2] - 1):
                # 1. Get corner values
                corners = [
                    volume[x, y, z], volume[x+1, y, z], volume[x+1, y+1, z], volume[x, y+1, z],
                    volume[x, y, z+1], volume[x+1, y, z+1], volume[x+1, y+1, z+1], volume[x, y+1, z+1]
                ]
                
                # 2. Classify cube
                cube_index = 0
                for i in range(8):
                    if corners[i] < isolevel:
                        cube_index |= (1 << i)
                
                # 3. Find edge crossings and normals
                edge_crossings = []
                normals = []
                edge_mask = EDGE_TABLE[cube_index]
                
                for e in range(12):
                    if (edge_mask >> e) & 1:
                        v0_idx, v1_idx = EDGE_VERTICES[e]
                        v0, v1 = corners[v0_idx], corners[v1_idx]
                        # Linear interpolation
                        t = (isolevel - v0) / (v1 - v0) if v1 != v0 else 0.0
                        # Interpolate position (unit cube coordinates)
                        p = [0.0, 0.0, 0.0]
                        for dim in range(3):
                            p[dim] = (v0_idx & (1 << dim)) * (1 - t) + (v1_idx & (1 << dim)) * t
                        edge_crossings.append(p)
                        
                        # Compute normal using forward differences
                        gx = volume[max(0, p[0]+1), max(0, p[1]), max(0, p[2])] - volume[max(0, p[0]), max(0, p[1]), max(0, p[2])]
                        gy = volume[max(0, p[0]), max(0, p[1]+1), max(0, p[2])] - volume[max(0, p[0]), max(0, p[1]), max(0, p[2])]
                        gz = volume[max(0, p[0]), max(0, p[1]), max(0, p[2]+1)] - volume[max(0, p[0]), max(0, p[1]), max(0, p[2])]
                        norm = [gx, gy, gz]
                        length = math.sqrt(sum(n**2 for n in norm))
                        if length > 0:
                            norm = [n / length for n in norm]
                        normals.append(norm)
                
                # 4. Detect sharp features
                is_feature = False
                for i in range(len(normals)):
                    for j in range(i + 1, len(normals)):
                        dot_product = sum(a * b for a, b in zip(normals[i], normals[j]))
                        angle = math.acos(max(-1.0, min(1.0, dot_product)))
                        if math.degrees(angle) < threshold_angle:
                            is_feature = True
                            break
                    if is_feature:
                        break
                
                # 5. Solve QEF if feature detected
                feature_point = None
                feat_idx = None
                if is_feature and len(edge_crossings) > 0:
                    # Construct planes: n_i . (x - p_i) = 0 => n_i . x = n_i . p_i
                    A = []
                    b = []
                    for p, n in zip(edge_crossings, normals):
                        A.append(n)
                        b.append(sum(a * bi for a, bi in zip(n, p)))
                    
                    # Solve least squares: (A^T A) x = A^T b
                    ATA = [[sum(A[k][i] * A[k][j] for k in range(len(A))) for j in range(3)] for i in range(3)]
                    ATb = [sum(A[k][i] * b[k] for k in range(len(A))) for i in range(3)]
                    
                    feature_point = solve_3x3_linear_system(ATA, ATb)
                    
                    feat_idx = len(mesh_vertices)
                    mesh_vertices.append(feature_point)
                    feature_vertices.add(feat_idx)
                
                # 6. Generate triangles
                loops = EDGE_LOOPS[cube_index]
                for loop in loops:
                    if feature_point is not None and feat_idx is not None:
                        # Triangle fan from feature point
                        for i in range(len(loop)):
                            j = (i + 1) % len(loop)
                            p1 = edge_crossings[loop[i]]
                            p2 = edge_crossings[loop[j]]
                            idx1 = len(mesh_vertices)
                            idx2 = len(mesh_vertices)
                            mesh_vertices.append(p1)
                            mesh_vertices.append(p2)
                            mesh_indices.extend([feat_idx, idx1, idx2])
                    else:
                        # Standard MC triangle generation would use TRIANGLE_TABLE here
                        pass
    
    # 7. Post-processing: Flip feature edges
    flip_feature_edges(mesh_vertices, mesh_indices, feature_vertices)
    
    return mesh_vertices, mesh_indices

def solve_3x3_linear_system(A, b):
    """Solve Ax = b for 3x3 system using Cramer's rule."""
    def det3(m):
        return (m[0][0] * (m[1][1]*m[2][2] - m[1][2]*m[2][1]) -
                m[0][1] * (m[1][0]*m[2][2] - m[1][2]*m[2][0]) +
                m[0][2] * (m[1][0]*m[2][1] - m[1][1]*m[2][0]))
    
    D = det3(A)
    if abs(D) < 1e-9:
        # Fallback to centroid if singular
        return [0.0, 0.0, 0.0]
    
    x = [0.0, 0.0, 0.0]
    for i in range(3):
        Ai = [row[:] for row in A]
        for r in range(3):
            Ai[r][i] = b[r]
        x[i] = det3(Ai) / D
    return x

def flip_feature_edges(vertices, indices, feature_set):
    """
    Iterate over edges. If both endpoints are feature vertices, mark for flipping.
    Simplified topology handling for demonstration.
    """
    flips = set()
    for i in range(0, len(indices), 3):
        v0, v1, v2 = indices[i], indices[i+1], indices[i+2]
        edges = [(v0, v1), (v1, v2), (v2, v0)]
        for e in edges:
            if e[0] in feature_set and e[1] in feature_set:
                flips.add(e)
    # Actual rotation logic depends on mesh data structure [S2, unofficial]
```

### Parameters
- **Isolevel**: The scalar value defining the surface. Typical values depend on the data range.
- **Threshold Angle**: The angle below which normals are considered to indicate a sharp feature. A common value is 90 degrees [S3, unofficial].
- **Step Size (E)**: Used in finite difference normal approximation. Not given in the sources.

### Edge Cases and Pitfalls
- **Underdetermined System**: If fewer than three planes are available for QEF solving (e.g., only 1 or 2 intersections), the system is underdetermined [S1]. The QEF minimization still yields a least-squares solution [S1].
- **Boundary Conditions**: Normal computation at volume boundaries requires special handling (e.g., one-sided differences) [G].
- **Slivers**: The algorithm may produce small, thin triangles (slivers) on sharp edges [S2, unofficial].
- **Dependencies**: The edge-flipping post-processing step may require information from neighboring chunks or cells, complicating parallel implementation [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Extended Marching Cubes was introduced to address the limitations of standard Marching Cubes in preserving sharp features [S1]. The method relies on Quadratic Error Functions (QEFs) to locate feature points, a technique also used in Dual Contouring [S1]. The algorithm was presented in the paper "Feature Sensitive Surface Extraction from Volume Data" [S3, unofficial][S1].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S2] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S3] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
