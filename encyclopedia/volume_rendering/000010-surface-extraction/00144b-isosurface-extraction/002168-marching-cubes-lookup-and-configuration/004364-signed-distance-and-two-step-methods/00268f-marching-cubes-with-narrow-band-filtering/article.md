# Marching Cubes with Narrow-Band Filtering

Also called: MeshReconstruction::MarchCube

## Overview
This method reconstructs a triangle mesh from a Signed Distance Function (SDF) by evaluating the SDF on a grid of cubes, filtering cubes based on their distance to the surface (narrow-band filtering), and applying the Marching Cubes algorithm to intersected cubes [S1, unofficial]. It sets itself apart from standard Marching Cubes by skipping voxels where the distance from the center to the surface exceeds the cube's geometric bounds, significantly accelerating reconstruction for sparse or bounded SDFs [S1, unofficial].

## How it works
**Inputs:**
*   `sdf`: A function $S: \mathbb{R}^3 \to \mathbb{R}$ representing the Signed Distance Function.
*   `domain`: A `Rect3` structure defining the bounding box of the reconstruction volume, with fields `min` (Vec3) and `size` (Vec3).
*   `cubeSize`: A Vec3 defining the dimensions of each voxel/cube. If not provided, it is calculated from `NumCubes`.
*   `isoLevel`: The scalar value defining the isosurface (typically 0 for SDFs).
*   `sdfGrad`: An optional gradient function for vertex normals. If `nullptr`, numerical approximation is used [G].

**Outputs:**
*   `mesh`: A triangle mesh data structure containing vertices and indices.

**Data Structures:**
*   `Rect3`: Contains `min` (Vec3) and `size` (Vec3).
*   `Cube`: Contains `pos[8]` (Vec3 array of vertex positions) and `sdf[8]` (double array of SDF values at vertices).
*   `signConfigToIntersectedEdges`: A static array of 256 integers (bitmasks) mapping vertex sign configurations to intersected edges.
*   `edges`: A static array of 12 `Edge` structs. Each `Edge` has `edgeFlag` (12-bit integer, powers of 2: 1, 2, 4... 2048), `vert0` (int 0-7), and `vert1` (int 0-7).

**Parameters:**
*   `NumCubes`: Integer constant, default 50. Used to calculate default `cubeSize` as `domain.size * (1.0 / NumCubes)` [S1, unofficial].
*   `isoLevel`: Default 0 [S1, unofficial].

**Steps:**
1.  **Calculate Grid Dimensions:**
    $$ \text{NumX} = \lceil \text{domain.size.x} / \text{cubeSize.x} \rceil $$
    $$ \text{NumY} = \lceil \text{domain.size.y} / \text{cubeSize.y} \rceil $$
    $$ \text{NumZ} = \lceil \text{domain.size.z} / \text{cubeSize.z} \rceil $$
2.  **Calculate Geometric Constants:**
    $$ \text{HalfCubeDiag} = \|\text{cubeSize}\| / 2.0 $$
    $$ \text{HalfCubeSize} = \text{cubeSize} \times 0.5 $$
3.  **Iterate Grid:** For each grid index $(ix, iy, iz)$ where $0 \le ix < \text{NumX}$, etc.:
    a. Calculate cube minimum corner:
       $$ \text{min} = \text{domain.min} + \{ ix \cdot \text{cubeSize.x}, iy \cdot \text{cubeSize.y}, iz \cdot \text{cubeSize.z} \} $$
    b. Calculate cube center:
       $$ \text{cubeCenter} = \text{min} + \text{HalfCubeSize} $$
    c. Evaluate SDF at center and calculate distance to isosurface:
       $$ \text{dist} = | \text{sdf}(\text{cubeCenter}) - \text{isoLevel} | $$
    d. **Narrow-Band Filter:** If $\text{dist} > \text{HalfCubeDiag}$, the cube cannot intersect the isosurface. Continue to next cube [S1, unofficial].
    e. Construct `Cube` object:
       i. Set `Cube.pos[v]` for $v \in [0,7]$ to the 8 vertices of the cube defined by `min` and `cubeSize`.
       ii. Set `Cube.sdf[v] = sdf(Cube.pos[v])`.
    f. **Classify Cube:** Compute vertex signs $s_v = \text{sign}(\text{Cube.sdf}[v] - \text{isoLevel})$. Form an 8-bit configuration index $C = \sum_{v=0}^{7} s_v \cdot 2^v$.
    g. **Find Intersections:** Use `signConfigToIntersectedEdges[C]` to identify intersected edges. For each intersected edge $e$ (defined by `edges[e_idx]`):
       i. Interpolate intersection point $P$ on the edge between vertices $v_0 = \text{Cube.pos}[e.\text{vert0}]$ and $v_1 = \text{Cube.pos}[e.\text{vert1}]$ with SDF values $d_0 = \text{Cube.sdf}[v_0]$ and $d_1 = \text{Cube.sdf}[v_1]$:
          $$ \mu = \frac{\text{isoLevel} - d_0}{d_1 - d_0} $$
          $$ P = v_0 + (v_1 - v_0) \cdot \mu $$
          (Handle division by zero or near-zero denominator by returning $v_0$ if $|d_1 - d_0| < \epsilon$) [S1, unofficial].
    h. **Triangulate:** Use the standard Marching Cubes `triangleTable` (from parent article) to generate triangles from the intersection points. Add triangles to `mesh`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants from source [S1, unofficial]
signConfigToIntersectedEdges = [
    0x0, 0x109, 0x203, 0x30a, 0x406, 0x50f, 0x605, 0x70c,
    0x80c, 0x905, 0xa0f, 0xb06, 0xc0a, 0xd03, 0xe09, 0xf00,
    0x190, 0x99, 0x393, 0x29a, 0x596, 0x49f, 0x795, 0x69c,
    0x99c, 0x895, 0xb9f, 0xa96, 0xd9a, 0xc93, 0xf99, 0xe90,
    0x230, 0x339, 0x33, 0x13a, 0x636, 0x73f, 0x435, 0x53c,
    0xa3c, 0xb35, 0x83f, 0x936, 0xe3a, 0xf33, 0xc39, 0xd30,
    0x3a0, 0x2a9, 0x1a3, 0xaa, 0x7a6, 0x6af, 0x5a5, 0x4ac,
    0xbac, 0xaa5, 0x9af, 0x8a6, 0xfaa, 0xea3, 0xda9, 0xca0,
    0x460, 0x569, 0x663, 0x76a, 0x66, 0x16f, 0x265, 0x36c,
    0xc6c, 0xd65, 0xe6f, 0xf66, 0x86a, 0x963, 0xa69, 0xb60,
    0x5f0, 0x4f9, 0x7f3, 0x6fa, 0x1f6, 0xff, 0x3f5, 0x2fc,
    0xdfc, 0xcf5, 0xfff, 0xef6, 0x9fa, 0x8f3, 0xbf9, 0xaf0,
    0x650, 0x759, 0x453, 0x55a, 0x256, 0x35f, 0x55, 0x15c,
    0xe5c, 0xf55, 0xc5f, 0xd56, 0xa5a, 0xb53, 0x859, 0x950,
    0x7c0, 0x6c9, 0x5c3, 0x4ca, 0x3c6, 0x2cf, 0x1c5, 0xcc,
    0xfcc, 0xec5, 0xdcf, 0xcc6, 0xbca, 0xac3, 0x9c9, 0x8c0,
    0x8c0, 0x9c9, 0xac3, 0xbca, 0xcc6, 0xdcf, 0xec5, 0xfcc,
    0xcc, 0x1c5, 0x2cf, 0x3c6, 0x4ca, 0x5c3, 0x6c9, 0x7c0,
    0x950, 0x859, 0xb53, 0xa5a, 0xd56, 0xc5f, 0xf55, 0xe5c,
    0x15c, 0x55, 0x35f, 0x256, 0x55a, 0x453, 0x759, 0x650,
    0xaf0, 0xbf9, 0x8f3, 0x9fa, 0xef6, 0xfff, 0xcf5, 0xdfc,
    0x2fc, 0x3f5, 0xff, 0x1f6, 0x6fa, 0x7f3, 0x4f9, 0x5f0,
    0xb60, 0xa69, 0x963, 0x86a, 0xf66, 0xe6f, 0xd65, 0xc6c,
    0x36c, 0x265, 0x16f, 0x66, 0x76a, 0x663, 0x569, 0x460,
    0xca0, 0xda9, 0xea3, 0xfaa, 0x8a6, 0x9af, 0xaa5, 0xbac,
    0x4ac, 0x5a5, 0x6af, 0x7a6, 0xaa, 0x1a3, 0x2a9, 0x3a0,
    0xd30, 0xc39, 0xf33, 0xe3a, 0x936, 0x83f, 0xb35, 0xa3c,
    0x53c, 0x435, 0x73f, 0x636, 0x13a, 0x33, 0x339, 0x230,
    0xe90, 0xf99, 0xf00, 0xe09, 0xd03, 0xc0a, 0xb06, 0xa0f,
    0x905, 0x80c, 0x70c, 0x605, 0x50f, 0x406, 0x30a, 0x203,
    0x109, 0x0
]

edges = [
    {"edgeFlag": 1, "vert0": 0, "vert1": 1},
    {"edgeFlag": 2, "vert0": 1, "vert1": 2},
    {"edgeFlag": 4, "vert0": 2, "vert1": 3},
    {"edgeFlag": 8, "vert0": 3, "vert1": 0},
    {"edgeFlag": 16, "vert0": 4, "vert1": 5},
    {"edgeFlag": 32, "vert0": 5, "vert1": 6},
    {"edgeFlag": 64, "vert0": 6, "vert1": 7},
    {"edgeFlag": 128, "vert0": 7, "vert1": 4},
    {"edgeFlag": 256, "vert0": 0, "vert1": 4},
    {"edgeFlag": 512, "vert0": 1, "vert1": 5},
    {"edgeFlag": 1024, "vert0": 2, "vert1": 6},
    {"edgeFlag": 2048, "vert0": 3, "vert1": 7}
]

# Placeholder for triangleTable from parent article [Parent]
# triangleTable is a 256-entry array of arrays of edge indices
triangleTable = [...] 

def lerp_vertex(isoLevel, v1, v2, p1, p2, eps=1e-5):
    if abs(isoLevel - v1) < eps:
        return p1
    if abs(isoLevel - v2) < eps:
        return p1
    if abs(v1 - v2) < eps:
        return p1
    mu = (isoLevel - v1) / (v2 - v1)
    return p1 + (p2 - p1) * mu

def marching_cubes_narrow_band(sdf, domain, cubeSize=None, isoLevel=0.0, NumCubes=50):
    if cubeSize is None:
        cubeSize = domain.size * (1.0 / NumCubes)
    
    numX = math.ceil(domain.size.x / cubeSize.x)
    numY = math.ceil(domain.size.y / cubeSize.y)
    numZ = math.ceil(domain.size.z / cubeSize.z)
    
    halfCubeDiag = math.sqrt(cubeSize.x**2 + cubeSize.y**2 + cubeSize.z**2) / 2.0
    halfCubeSize = cubeSize * 0.5
    
    mesh = Mesh()
    
    for ix in range(numX):
        for iy in range(numY):
            for iz in range(numZ):
                min_corner = domain.min + Vec3(ix * cubeSize.x, iy * cubeSize.y, iz * cubeSize.z)
                cubeCenter = min_corner + halfCubeSize
                dist = abs(sdf(cubeCenter) - isoLevel)
                
                if dist > halfCubeDiag:
                    continue
                
                # Construct cube vertices and SDF values
                pos = [
                    min_corner,
                    min_corner + Vec3(cubeSize.x, 0, 0),
                    min_corner + Vec3(cubeSize.x, cubeSize.y, 0),
                    min_corner + Vec3(0, cubeSize.y, 0),
                    min_corner + Vec3(0, 0, cubeSize.z),
                    min_corner + Vec3(cubeSize.x, 0, cubeSize.z),
                    min_corner + Vec3(cubeSize.x, cubeSize.y, cubeSize.z),
                    min_corner + Vec3(0, cubeSize.y, cubeSize.z)
                ]
                sdf_vals = [sdf(p) for p in pos]
                
                # Compute configuration index
                config = 0
                for v in range(8):
                    if sdf_vals[v] < isoLevel:
                        config |= (1 << v)
                
                # Get intersected edges
                intersected_edges_mask = signConfigToIntersectedEdges[config]
                
                # Find intersection points
                intersections = []
                for e_idx, edge in enumerate(edges):
                    if intersected_edges_mask & edge["edgeFlag"]:
                        v0_idx = edge["vert0"]
                        v1_idx = edge["vert1"]
                        p_int = lerp_vertex(isoLevel, sdf_vals[v0_idx], sdf_vals[v1_idx], pos[v0_idx], pos[v1_idx])
                        intersections.append(p_int)
                
                # Triangulate
                # Note: triangleTable indices refer to the 12 edges, not the intersections list directly.
                # We must map edge indices to intersection points.
                edge_to_inter = {}
                for e_idx, edge in enumerate(edges):
                    if intersected_edges_mask & edge["edgeFlag"]:
                        edge_to_inter[e_idx] = intersections.pop(0)
                
                tri_indices = triangleTable[config]
                for i in range(0, len(tri_indices), 3):
                    e0 = tri_indices[i]
                    e1 = tri_indices[i+1]
                    e2 = tri_indices[i+2]
                    
                    p0 = edge_to_inter[e0]
                    p1 = edge_to_inter[e1]
                    p2 = edge_to_inter[e2]
                    
                    mesh.add_triangle(p0, p1, p2)
                    
    return mesh
```

## Subtopics

### SDF to PLY Conversion
Converts SDF data directly to PLY format, potentially using similar narrow-band techniques but focusing on output format [Sibling].

### Two-Step Surface Reconstruction Method
Uses a low-cost approximation to screen voxels before high-accuracy evaluation, offering a different acceleration strategy than simple distance filtering [Sibling].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the grid, but effectively much less due to narrow-band filtering skipping non-intersecting cubes.
*   **Memory Complexity**: $O(1)$ auxiliary space per cube, plus $O(V)$ for the output mesh where $V$ is the number of vertices.
*   **Performance**: Fast due to precomputed lookup tables and narrow-band approach excluding cubes far from the surface [S1, unofficial].
*   **Strengths**: Efficient for SDFs where the surface is localized; avoids evaluating SDF for all voxels.
*   **Limitations**: Requires SDF or a function that can be evaluated at arbitrary points; grid resolution affects accuracy.
*   **When to use**: When reconstructing meshes from SDFs or implicit functions where the domain is large but the surface is confined.
*   **When not to use**: When the surface fills the entire domain or when SDF evaluation is extremely expensive and narrow-band filtering doesn't skip enough cubes.

## Background
The method is implemented in the `MeshReconstruction` library by Magnus2 [S1, unofficial]. It builds on the classic Marching Cubes algorithm [S1, unofficial] and is designed to be self-contained with no dependencies, requiring C++14 [S1, unofficial]. It is licensed under the MIT License [S1, unofficial].

## Sources
[S1] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
