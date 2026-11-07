# Dual Mesh Building

Also called: Dual Mesh Construction, Flexible Dual Grid Mesh Extraction, Flexible Dual Grid to Mesh

## Overview
Dual Mesh Building constructs a quadrilateral mesh from a scalar volume grid by mapping the vertices of standard Marching Cubes (MC) polygons to vertices in a dual mesh, where each dual vertex represents an MC polygon and each dual face corresponds to an MC vertex [S1, unofficial][S2]. It solves the problem of generating manifold, watertight meshes with better-shaped elements and fewer vertices than standard MC [S2]. Unlike standard MC, which outputs triangles from edge intersections, Dual Mesh Building outputs quads by connecting vertex representatives from the four cells sharing an intersected grid edge [S2]. It differs from [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md) by relying on the asymptotic decider for topological consistency rather than explicit ambiguous configuration correction, and from [Parallel Dual Marching Cube](2131-parallel-dual-marching-cube.md) by focusing on the general algorithmic construction of the dual topology and geometry [S2].

## How it works
**Inputs:** A 3D scalar volume grid $V$ of dimensions $D_x \times D_y \times D_z$, an isovalue $\iota_0$, and a boolean flag `generateManifold` (optional, default false) [S1, unofficial][S2].
**Outputs:** A list of vertices $P$, a list of quadrilateral indices $Q$, and optionally a halfedge data structure [S1, unofficial][S2].

**Data Structures:**
*   `Vertex`: Struct with fields `x`, `y`, `z` of type `float` [S1, unofficial].
*   `Quad`: Struct with fields `i0`, `i1`, `i2`, `i3` of type `int32_t` [S1, unofficial].
*   `DualPointKey`: Struct with `linearizedCellID` (`int32_t`) and `pointCode` (`int`) [S1, unofficial].
*   `pointToIndex`: Hash map mapping `DualPointKey` to `QuadIndexType` (shared vertex index) [S1, unofficial].
*   `hashTable`: Maps edge index (key) to an array of four vertex indices (value) [S2].

**Formulas:**
The trilinear interpolant $T$ on a unit reference cell is:
$$ F(u, v, w) = (1 - w)[f_0(1 - u)(1 - v) + f_1u(1 - v) + f_2(1 - u)v + f_3uv] + w[f_4(1 - u)(1 - v) + f_5u(1 - v) + f_6(1 - u)v + f_7uv] $$
where $(u, v, w) \in [0, 1]^3$ are local coordinates and $f_i$ are scalar values at cell vertices $v_i$ [S2].

Given sampled parameters $u, v$, the third parameter $w$ is computed as:
$$ w = \frac{t_0 - g_1(u, v)}{g_2(u, v) - g_1(u, v)} $$
where $t_0 = \iota_0$, $g_1(u, v) = f_0(1 - u)(1 - v) + f_1u(1 - v) + f_2(1 - u)v + f_3uv$, and $g_2(u, v) = f_4(1 - u)(1 - v) + f_5u(1 - v) + f_6(1 - u)v + f_7uv$ [S2].

Linearized cell index:
$$ g_A(x, y, z) = x + D_x(y + D_z z) $$ [S1, unofficial].

**Steps:**

1.  **Initialize Buffers:** Create empty lists for vertices and quads. Initialize a hash table for edge-to-vertex mapping with capacity twice the expected number of intersected edges [G]. Initialize an atomic counter for global vertex indexing [S2].
2.  **Cell-wise Processing (Kernel 1):** For each cell $(cx, cy, cz)$:
    a.  **Check Intersection:** Compute the 8-bit cell code (in-out mask) based on corner values relative to $\iota_0$. If no intersection, skip to the next cell [S1, unofficial][S2].
    b.  **Compute MC Polygons:** Determine the intersected edges and form closed curves (MC polygons) using the asymptotic decider to resolve ambiguities [S2]. Encode polygon size and intersected edge indices in a 64-bit integer [S2].
    c.  **Select Vertex Representatives:** For each MC polygon branch:
        i.  Determine the sampling plane $(u, v)$, $(u, w)$, or $(v, w)$ where the polygon has the largest projection [S2].
        ii. Sample the plane (e.g., $7 \times 7$ grid). Compute the third parameter using the formula above. Discard samples outside $[0, 1]$ or within the bounding box of other branches [S2].
        iii. If no valid sample, retry with $25 \times 25$ grid. If still none, use the center of gravity of the MC polygon [S2].
        iv. Select the sample closest to the center of gravity as the vertex representative [S2].
    d.  **Back Projection:** Reposition the representative to lie exactly on the isosurface $T(x,y,z) = \iota_0$ using Newton-Raphson iteration (default 5 iterations) along the gradient direction [S4, unofficial]. [G] If Newton-Raphson fails, fallback to the sampled position.
    e.  **Compute Normal:** Estimate gradient at cell vertices via central difference. Interpolate gradient trilinearly at the representative position and normalize [S2].
    f.  **Assign Global Index:** Use `atomicAdd` to assign a unique global index to the vertex representative. Store this index in `pointToIndex` using `DualPointKey` (linearized cell ID + point code) to ensure shared vertices for the same dual point [S1, unofficial].
    g.  **Store in Hash Table:** For each intersected edge, store the vertex index in the hash table at the position corresponding to the cell's orientation relative to the edge (see Quadrilateral Construction below) [S2].
3.  **Quadrilateral Construction (Kernel 2):**
    a.  Iterate through the hash table. For each edge, retrieve the four vertex indices stored by the incident cells [S2].
    b.  If all four indices are valid (not boundary), form a quad `{B, A, D, C}` as defined by the edge orientation logic [S2].
    c.  Append the quad to the output list [S2].
4.  **Manifold Correction (Optional):** If `generateManifold` is true, apply Wenger's correction using the `problematicConfigs` table to resolve non-manifold edges caused by ambiguous configurations (C16, C19) [S1, unofficial].
5.  **Halfedge Data Structure (Optional):** Construct connectivity by grouping halfedges by incident vertex pairs using a hash table, handling non-manifold edges (shared by four faces) if present [S1, unofficial][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dual_mesh_building(volume, dims, iso, generate_manifold=False):
    vertices = []
    quads = []
    point_to_index = {}
    edge_hash = {}
    
    for z in range(dims[2]):
        for y in range(dims[1]):
            for x in range(dims[0]):
                cell_idx = x + dims[0] * (y + dims[1] * z)
                cell_code = get_cell_code(volume, dims, x, y, z, iso)
                
                if cell_code == 0 or cell_code == 255:
                    continue

                intersected_edges = get_intersected_edges(cell_code)
                branches = group_edges_into_branches(intersected_edges)
                
                for branch_edges in branches:
                    rep_pos = find_vertex_representative(branch_edges, volume, dims, x, y, z, iso)
                    rep_pos = newton_raphson_projection(rep_pos, volume, dims, x, y, z, iso)
                    grad = compute_gradient(rep_pos, volume, dims, x, y, z)
                    normal = normalize(grad)
                    
                    point_code = hash(tuple(sorted(branch_edges)))
                    key = (cell_idx, point_code)
                    
                    if key not in point_to_index:
                        vertices.append(Vertex(rep_pos.x, rep_pos.y, rep_pos.z, normal.x, normal.y, normal.z))
                        point_to_index[key] = len(vertices) - 1
                    v_idx = point_to_index[key]
                    
                    for edge_idx in branch_edges:
                        pos = get_quad_position_for_edge(edge_idx, x, y, z)
                        if edge_idx not in edge_hash:
                            edge_hash[edge_idx] = [None, None, None, None]
                        edge_hash[edge_idx][pos] = v_idx

    for edge_idx, quad_indices in edge_hash.items():
        if all(v is not None for v in quad_indices):
            quads.append(Quad(quad_indices[0], quad_indices[1], quad_indices[2], quad_indices[3]))

    if generate_manifold:
        # apply_wenger_correction(quads, volume, dims, iso) # Uses problematicConfigs[256] table [S1, unofficial]
        pass

    return vertices, quads

def get_cell_code(volume, dims, cx, cy, cz, iso):
    mask = 0
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                idx = (cx + dx) + dims[0] * ((cy + dy) + dims[1] * (cz + dz))
                if volume[idx] >= iso:
                    mask |= (1 << (dx + 2*dy + 4*dz))
    return mask

def get_intersected_edges(cell_code):
    edges = []
    for i in range(12):
        if cell_code & (1 << i):
            edges.append(i)
    return edges

def group_edges_into_branches(edges):
    return [edges]

def find_vertex_representative(branch_edges, volume, dims, cx, cy, cz, iso):
    for grid_size in [7, 25]:
        samples = []
        for u in range(grid_size):
            for v in range(grid_size):
                su, sv = u / (grid_size - 1), v / (grid_size - 1)
                w = compute_w(su, sv, volume, dims, cx, cy, cz, iso)
                if 0.0 <= w <= 1.0:
                    samples.append(Point(su, sv, w))
        if samples:
            return min(samples, key=lambda p: distance_to_centroid(p, branch_edges))
    return Point(0.5, 0.5, 0.5)

def compute_w(u, v, volume, dims, cx, cy, cz, iso):
    f = get_cell_vertex_values(volume, dims, cx, cy, cz)
    g1 = f[0]*(1-u)*(1-v) + f[1]*u*(1-v) + f[2]*(1-u)*v + f[3]*u*v
    g2 = f[4]*(1-u)*(1-v) + f[5]*u*(1-v) + f[6]*(1-u)*v + f[7]*u*v
    if g2 == g1: return 0.5
    return (iso - g1) / (g2 - g1)

def newton_raphson_projection(point, volume, dims, cx, cy, cz, iso):
    p = Point(point.x, point.y, point.z)
    for _ in range(5):
        grad = compute_gradient(p, volume, dims, cx, cy, cz)
        val = trilinear_interpolate(p, volume, dims, cx, cy, cz)
        if abs(val - iso) < 1e-6: break
        p = Point(p.x - grad.x, p.y - grad.y, p.z - grad.z)
    return p

def compute_gradient(point, volume, dims, cx, cy, cz):
    h = 1e-4
    dx = (trilinear_interpolate(Point(point.x+h, point.y, point.z), volume, dims, cx, cy, cz) - 
          trilinear_interpolate(Point(point.x-h, point.y, point.z), volume, dims, cx, cy, cz)) / (2*h)
    dy = (trilinear_interpolate(Point(point.x, point.y+h, point.z), volume, dims, cx, cy, cz) - 
          trilinear_interpolate(Point(point.x, point.y-h, point.z), volume, dims, cx, cy, cz)) / (2*h)
    dz = (trilinear_interpolate(Point(point.x, point.y, point.z+h), volume, dims, cx, cy, cz) - 
          trilinear_interpolate(Point(point.x, point.y, point.z-h), volume, dims, cx, cy, cz)) / (2*h)
    return Vector(dx, dy, dz)

def get_quad_position_for_edge(edge_idx, cx, cy, cz):
    return edge_idx % 4

def trilinear_interpolate(point, volume, dims, cx, cy, cz):
    u, v, w = point.x, point.y, point.z
    f = get_cell_vertex_values(volume, dims, cx, cy, cz)
    return (1-w)*(f[0]*(1-u)*(1-v) + f[1]*u*(1-v) + f[2]*(1-u)*v + f[3]*u*v) + \
           w*(f[4]*(1-u)*(1-v) + f[5]*u*(1-v) + f[6]*(1-u)*v + f[7]*u*v)

def get_cell_vertex_values(volume, dims, cx, cy, cz):
    vals = []
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                idx = (cx + dx) + dims[0] * ((cy + dy) + dims[1] * (cz + dz))
                vals.append(volume[idx])
    return vals

def normalize(v):
    mag = (v.x**2 + v.y**2 + v.z**2)**0.5
    return Vector(v.x/mag, v.y/mag, v.z/mag) if mag > 0 else Vector(0,0,0)

def distance_to_centroid(p, branch_edges):
    return 0.0

class Point:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
class Vector:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
class Vertex:
    def __init__(self, x, y, z, nx, ny, nz): self.x, self.y, self.z, self.nx, self.ny, self.nz = x, y, z, nx, ny, nz
class Quad:
    def __init__(self, i0, i1, i2, i3): self.i0, self.i1, self.i2, self.i3 = i0, i1, i2, i3
```

## Subtopics

### [Dual Marching Cubes Table Generation](2069-dual-marching-cubes-table-generation.md)
Offline algorithm constructing lookup tables for DualMC, mapping voxel corner configurations to dual point edge masks for efficient GPU parallel construction.

### [Manifold Dual Marching Cubes](2135-manifold-dual-marching-cubes.md)
Variant resolving ambiguous configurations (C16, C19) to guarantee a watertight, manifold mesh by checking neighboring cells and inverting cell codes.

### [Mesh simplification via face coloring](2141-mesh-simplification-via-face-coloring.md)
Parallel method reducing DMC mesh density by identifying and removing quads based on vertex valence patterns (3-X–3-Y, 3-3-3-3) using a 5-color face scheme.

### [Halfedge Data Structure Construction](2310-halfedge-data-structure-construction.md)
Constructs global connectivity for the DMC quad mesh using hash tables to group halfedges by incident vertex pairs, handling non-manifold edges.

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming constant-time hash table operations. The two-kernel parallel approach on GPU achieves high throughput [S2].
**Memory Complexity:** $O(E)$ where $E$ is the number of intersected edges, for the hash table and vertex storage [S2].
**Performance:** On NVIDIA GeForce RTX 4090, DMC achieves ~1.0 billion faces/s on sparse Fandisk CAD at $1024^3$ resolution (3.44 ms latency for 3.43 M faces) [S4, unofficial]. It is 4.3x faster than Dual Contouring and 4.8x faster than Marching Cubes Asymptotic (MCA) on the same data [S4, unofficial].
**Strengths:** Generates high-quality quad-only meshes with fewer vertices and better-shaped elements than standard MC. Manifold and watertight by construction (with asymptotic decider). Parallelizable on GPU [S2].
**Limitations:** Cannot reconstruct tunnels of sub-voxel size. May generate non-manifold edges if iso-surface forms a tunnel across two cells (requires special halfedge handling) [S2]. Newton-Raphson projection may blur sharp corners compared to Dual Contouring [S4, unofficial]. Sparse grids require boundary padding (`pad=1`) to ensure all incident voxels are present [S4, unofficial].

## Background
Dual Mesh Building is based on the work of Gregory M. Nielson, who observed that faces and vertices of standard Marching Cubes correspond to vertices and faces in the dual algorithm [S1, unofficial]. The original implementation was provided by Dominik Wodniok in 2009 [S1, unofficial]. Rephael Wenger later proposed the Manifold Dual Marching Cubes algorithm in his book "Isosurfaces: Geometry, Topology, and Algorithms" (Chapter 3.3.5) to resolve non-manifold issues [S1, unofficial]. A parallel GPU implementation without lookup tables was presented by Roberto Grosso and Daniel Zint in 2021, introducing vertex representative positioning and halfedge data structure construction [S2].

## Sources
[S1] dominikwodniok/dualmc, https://github.com/dominikwodniok/dualmc, fetched 2026-10-26 (unofficial)
[S2] A parallel dual marching cubes approach to quad only surface reconstruction, https://doi.org/10.1007/s00371-021-02139-w, fetched 2026-10-26
[S3] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S4] Implementation of Dual Marching Cubes, https://github.com/KhoiDOO/conquer3d/issues/3, fetched 2026-10-27 (unofficial)
[S5] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S6] isosurface, https://github.com/Lin20/isosurface, fetched 2026-10-29 (unofficial)
