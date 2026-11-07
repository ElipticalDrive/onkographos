# Marching Cubes with Narrow-Band Filtering

Also called: MeshReconstruction::MarchCube

## Overview
This method reconstructs a triangle mesh from a Signed Distance Function (SDF) by evaluating the SDF on a regular grid of cubes, filtering out cubes that are too far from the surface (narrow-band filtering), and applying the standard Marching Cubes algorithm only to the intersected cubes [S1, unofficial]. It differs from standard Marching Cubes by skipping non-intersecting cells, significantly reducing computation time for sparse or bounded domains [S1, unofficial].

## How it works
**Inputs:**
- `domain`: A 3D bounding box defined by `min` (Vec3) and `size` (Vec3).
- `sdf`: A function `Fun3s` mapping `Vec3` position to a scalar distance value.
- `cubeSize`: Optional Vec3 defining the side length of each grid cell. If not provided, defaults to `domain.size / 50` [S1, unofficial].
- `isoLevel`: The scalar value at which the surface is extracted. Default is 0 [S1, unofficial].
- `sdfGrad`: Optional function returning the gradient of the SDF for vertex normals. If `nullptr`, numerical approximation is used [G].

**Outputs:**
- `mesh`: A triangle mesh containing vertices and indices.

**Data Structures:**
- `Cube`: Represents a single grid cell.
  - `pos[8]`: Array of 8 vertex positions (Vec3).
  - `sdf[8]`: Array of 8 scalar SDF values at vertices.
  - `intersect`: Result of intersection test, containing intersection points on edges.
- `signConfigToIntersectedEdges`: A lookup table of 256 integers (12-bit bitmasks). Index is the configuration of vertex signs (0-255). Value indicates which of the 12 edges are intersected.
- `edges`: Array of 12 `Edge` structs.
  - `edgeFlag`: 12-bit integer ($2^k$ for edge $k$).
  - `vert0`, `vert1`: Indices of the two vertices defining the edge (0-7).

**Steps:**

1.  **Calculate Grid Dimensions:**
    $$
    \text{NumX} = \lceil \text{domain.size.x} / \text{cubeSize.x} \rceil
    $$
    $$
    \text{NumY} = \lceil \text{domain.size.y} / \text{cubeSize.y} \rceil
    $$
    $$
    \text{NumZ} = \lceil \text{domain.size.z} / \text{cubeSize.z} \rceil
    $$
    [S1, unofficial]

2.  **Calculate Geometric Constants:**
    $$
    \text{HalfCubeDiag} = \|\text{cubeSize}\| / 2.0
    $$
    $$
    \text{HalfCubeSize} = \text{cubeSize} \times 0.5
    $$
    [S1, unofficial]

3.  **Iterate Grid:**
    For each grid index $(ix, iy, iz)$ where $0 \le ix < \text{NumX}$, $0 \le iy < \text{NumY}$, $0 \le iz < \text{NumZ}$:
    a. Calculate cube minimum corner:
       $$
       \text{min} = \text{domain.min} + \{ix \cdot \text{cubeSize.x}, iy \cdot \text{cubeSize.y}, iz \cdot \text{cubeSize.z}\}
       $$
       [S1, unofficial]
    b. Calculate cube center:
       $$
       \text{cubeCenter} = \text{min} + \text{HalfCubeSize}
       $$
       [S1, unofficial]
    c. Calculate distance to iso-surface:
       $$
       \text{dist} = |\text{sdf}(\text{cubeCenter}) - \text{isoLevel}|
       $$
       [S1, unofficial]

4.  **Narrow-Band Filter:**
    If $\text{dist} > \text{HalfCubeDiag}$, the cube is too far from the surface to intersect it. Continue to the next cube [S1, unofficial].

5.  **Construct Cube:**
    Create a `Cube` object using `domain` and `sdf`. Sample the SDF at the 8 vertices of the cube to populate `Cube.sdf` and `Cube.pos` [S1, unofficial].

6.  **Intersect:**
    Call `cube.Intersect(isoLevel)`. This determines which edges of the cube are intersected by the isosurface using the `signConfigToIntersectedEdges` lookup table and linear interpolation [S1, unofficial].

7.  **Triangulate:**
    Call `Triangulate(intersect, sdfGrad, mesh)`. Generate triangles from the intersection points and add them to the mesh. Use `sdfGrad` for normals if provided, otherwise compute numerically [G] [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_narrow_band(domain, sdf, cube_size=None, iso_level=0.0, sdf_grad=None):
    if cube_size is None:
        cube_size = domain.size / 50.0

    half_cube_diag = vector_norm(cube_size) / 2.0
    half_cube_size = cube_size * 0.5

    num_x = ceil(domain.size.x / cube_size.x)
    num_y = ceil(domain.size.y / cube_size.y)
    num_z = ceil(domain.size.z / cube_size.z)

    mesh = Mesh()

    for ix in range(num_x):
        for iy in range(num_y):
            for iz in range(num_z):
                min_corner = domain.min + vec3(ix * cube_size.x, iy * cube_size.y, iz * cube_size.z)
                cube_center = min_corner + half_cube_size
                
                dist = abs(sdf(cube_center) - iso_level)
                
                if dist > half_cube_diag:
                    continue

                cube = Cube(min_corner, cube_size, sdf)
                intersect = cube.intersect(iso_level)
                triangulate(intersect, sdf_grad, mesh)

    return mesh

class Cube:
    def __init__(self, min_corner, size, sdf):
        self.pos = [
            min_corner,
            min_corner + vec3(size.x, 0, 0),
            min_corner + vec3(size.x, size.y, 0),
            min_corner + vec3(0, size.y, 0),
            min_corner + vec3(0, 0, size.z),
            min_corner + vec3(size.x, 0, size.z),
            min_corner + vec3(size.x, size.y, size.z),
            min_corner + vec3(0, size.y, size.z)
        ]
        self.sdf = [sdf(p) for p in self.pos]

    def intersect(self, iso_level):
        # Determine configuration index based on signs of sdf values relative to iso_level
        config = 0
        for i in range(8):
            if self.sdf[i] < iso_level:
                config |= (1 << i)
        
        # Lookup intersected edges bitmask
        edges_mask = sign_config_to_intersected_edges[config]
        
        intersections = []
        for edge_idx in range(12):
            if edges_mask & (1 << edge_idx):
                edge = edges[edge_idx]
                # Linear interpolation to find intersection point
                p1 = self.pos[edge.vert0]
                p2 = self.pos[edge.vert1]
                v1 = self.sdf[edge.vert0]
                v2 = self.sdf[edge.vert1]
                
                # Handle edge cases for precision
                if abs(iso_level - v1) < 1e-5:
                    pt = p1
                elif abs(v1 - v2) < 1e-5:
                    pt = p1
                else:
                    mu = (iso_level - v1) / (v2 - v1)
                    pt = p1 + (p2 - p1) * mu
                intersections.append(pt)
        
        return intersections

# Global lookup tables (simplified representation)
sign_config_to_intersected_edges = [ ... ] # 256 entries as per source
edges = [ ... ] # 12 entries as per source
```

## Subtopics

### [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md)
Builds a mesh by advancing a front from seed points. Choose when the domain is unknown or sparse, unlike this method which requires a predefined grid.

### [Isosurface Extraction API](836-isosurface-extraction-api.md)
A generic interface for extraction methods. Use this if you need a standardized interface rather than a specific implementation.

### [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md)
Uses vertex positions and normals to create meshes. Choose when you have gradient information and want to preserve sharp features better than linear interpolation.

### [Transvoxel](6277-transvoxel.md)
Handles adaptive octree grids. Choose when the resolution needs to vary significantly across the domain, whereas this method uses a uniform grid.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of cubes in the grid. However, the narrow-band filter reduces the effective $N$ to only those cubes near the surface.
- **Memory Complexity:** $O(N)$ for the grid traversal and mesh storage. The lookup table is constant size ($256 \times 4$ bytes).
- **Performance:** Fast due to precomputed lookup tables and the exclusion of cubes far from the surface [S1, unofficial].
- **Strengths:** Simple to implement, efficient for bounded SDFs, produces watertight meshes if the grid is fine enough.
- **Limitations:** Uniform grid resolution may lead to aliasing or excessive memory use for large domains with small features. Does not adapt resolution locally.
- **When to use:** When the domain is well-defined and bounded, and the SDF is available.
- **When not to use:** When the domain is unbounded or the surface is extremely complex relative to the grid size, requiring adaptive refinement.

## Background
The method is implemented in the `MeshReconstruction` library by Magnus2 [S1, unofficial]. It combines the classic Marching Cubes algorithm [S1, unofficial] with a narrow-band filtering technique to optimize performance by skipping non-intersecting cubes. The library is self-contained, requires C++14, and is licensed under MIT [S1, unofficial].

## Sources
[S1] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
