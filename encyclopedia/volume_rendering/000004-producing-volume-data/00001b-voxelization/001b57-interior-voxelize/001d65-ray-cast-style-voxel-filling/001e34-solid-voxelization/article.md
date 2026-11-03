# Solid Voxelization

Also called: BfsSolid, Bulk Voxelization, Two-Level Model Voxelization

## Overview
Solid Voxelization fills the interior of a mesh with voxels (solid/bulk) rather than just the surface, using a two-level grid model [S1, unofficial]. It builds on [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md) by determining interior voxels via a two-pass approach with bitsets and a final inversion step [S2, unofficial]. It differs from [Slice Filling (Simple)](1691-slice-filling-simple.md) by explicitly handling volume connectivity via flood-fill or axis-aligned projection strategies to ensure watertightness, even if the input mesh is not strictly watertight [S2, unofficial].

## How it works
**Inputs:** A triangle mesh (vertices, indices, bounding box), and `gridVoxelSize` (the physical size of one side of a voxel) [S1, unofficial].
**Outputs:** A 3D grid bitmask where set bits indicate solid voxels [S1, unofficial, S2, unofficial].

**Data Structures:**
1. `MeshDataDevice`: GPU memory layout containing `bbox[6]` (float array `[min_x, min_y, min_z, max_x, max_y, max_z]`), `vertices` (array of 3D vectors), `triangles` (array of `Triangle` structs with `_idx1`, `_idx2`, `_idx3`, `_bottom`, `_top`), `trianglesNo` (int*), `_bottom` (Vector3Df), and `_top` (Vector3Df) [S1, unofficial].
2. `Triangle`: Struct containing `_idx1`, `_idx2`, `_idx3` (indices into vertices), `_bottom` (Vector3Df), and `_top` (Vector3Df) [S1, unofficial].
3. `Grid`: Contains `vTable1` (bitmask array) and grid dimensions (`gridSizeX`, `gridSizeY`, `gridSizeZ`) [S1, unofficial].
4. `voxels_` / `voxels_buffer_`: Bitsets used for flood-fill state, chunked by `kBatchSize` [S2, unofficial].

**Algorithm Steps:**
1. **Preprocessing:** Compute the global translation vector `globalTranslation[0..2] = meshData->bbox[0..2]` (the minimum corner) [S1, unofficial]. Compute grid dimensions using `ops::computeTwoLevelSizes(meshData->bbox, gridVoxelSize)` [S1, unofficial]. Allocate GPU memory for `MeshDataDevice` and copy data from CPU to GPU [S1, unofficial].
2. **Surface Voxelization:** Launch `solidVoxelGpu` kernel [S1, unofficial]. Each thread processes one triangle. Compute the triangle's bounding box relative to the mesh minimum. Iterate over voxels in the bounding box. Perform planar tests to determine if the triangle intersects the voxel. If intersected, flip the corresponding bit in `grid->vTable1` [S1, unofficial].
3. **Interior Filling:**
   - **Pass 1 (Bulk Fill):** Launch `fillBulkVoxelGpu` kernel [S1, unofficial]. For each slice (fixed Y, Z), iterate along X. Identify occupied voxels (surface hits). Fill all voxels between pairs of occupied voxels [S1, unofficial, S2, unofficial].
   - **Pass 2 (Flood Fill):** `BfsSolid` initializes a queue with boundary voxels [S2, unofficial]. Pops a voxel, checks `voxels_` and `voxels_buffer_` [S2, unofficial]. If the bit is not set, sets it in `voxels_buffer_` [S2, unofficial]. Checks 6 neighbors. If a neighbor is within the mesh bounding box and not set, pushes it to the queue [S2, unofficial].
   - **Final Inversion:** `voxels_[i] = voxels_buffer_[i] ^ (~static_cast<VoxelIndex>(0))` [S2, unofficial].
4. **Meta Data Update:** If metadata is required, `UpdateSolidMeta` iterates through all voxels [S2, unofficial]. If `GETBIT` is true and `Surface()` is false, increments `solid_cnt` and calls `SetSolid()` [S2, unofficial].

**Formulas:**
* **Edge Normals (for planar tests):** Let $e_0 = v_1 - v_0$, $e_1 = v_2 - v_1$, $e_2 = v_0 - v_2$.
  For XY plane:
  $$n_{xy\_e0} = (-e_0.y, e_0.x), \quad n_{xy\_e1} = (-e_1.y, e_1.x), \quad n_{xy\_e2} = (-e_2.y, e_2.x)$$
  If $n.z < 0.0$, flip all $n_{xy}$ vectors by multiplying by $-1.0$ [S1, unofficial].
  For ZX plane:
  $$n_{zx\_e0} = (-e_0.x, e_0.z), \quad n_{zx\_e1} = (-e_1.x, e_1.z), \quad n_{zx\_e2} = (-e_2.x, e_2.z)$$
  If $n.y < 0.0$, flip all $n_{zx}$ vectors by multiplying by $-1.0$ [S1, unofficial].
* **Distance Calculations:**
  $$d_{xy\_e0} = -\text{dot2D}(n_{xy\_e0}, (v_0.x, v_0.y)) + \max(0.0, \Delta p.x \cdot n_{xy\_e0}.x) + \max(0.0, \Delta p.y \cdot n_{xy\_e0}.y)$$
  (Analogous formulas apply for other edges and planes) [S1, unofficial].
* **Point-in-Triangle Test (2D Projection):** For a point $p$ in the projected plane, it is inside if:
  $$\text{dot2D}(n_{e0}, p) + d_{e0} \ge 0.0$$
  $$\text{dot2D}(n_{e1}, p) + d_{e1} \ge 0.0$$
  $$\text{dot2D}(n_{e2}, p) + d_{e2} \ge 0.0$$
  *Edge Case:* If $\text{dot2D}(n_{e0}, p) + d_{e0} == 0.0$, inclusion depends on normal direction: $n_{e0}.x > 0.0$ or $(n_{e0}.x == 0.0 \land n_{e0}.y < 0.0)$ [S1, unofficial].
* **3D Containment Check:**
  $$n \cdot p = \text{dot}(n, p)$$
  If $((n \cdot p + d_1) \cdot (n \cdot p + d_2)) > 0.0$, the point is outside the slab defined by the triangle's depth bounds [S1, unofficial].
* **Grid Indexing:**
  $$\text{location} = x + (y \cdot \text{gridSizeY}) + (z \cdot \text{gridSizeY} \cdot \text{gridSizeZ})$$
  Bits are flipped using `ops::flipBit(grid->vTable1, location)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solid_voxelization(mesh, voxel_size):
    # 1. Preprocessing
    bbox_min = mesh.bbox.min
    bbox_max = mesh.bbox.max
    grid_size = compute_grid_size(bbox_min, bbox_max, voxel_size)
    local_vertices = [v - bbox_min for v in mesh.vertices]
    
    # 2. Surface Voxelization
    surface_grid = BitGrid(grid_size)
    
    for tri in mesh.triangles:
        tri_bbox = compute_triangle_bbox(tri, voxel_size)
        for z in range(tri_bbox.z_min, tri_bbox.z_max + 1):
            for y in range(tri_bbox.y_min, tri_bbox.y_max + 1):
                # 2D Projection Test (XY)
                if not is_triangle_in_xy_projection(tri, y, z, voxel_size):
                    continue
                
                for x in range(tri_bbox.x_min, tri_bbox.x_max + 1):
                    # 3D Containment Test
                    p = vec3(x * voxel_size, y * voxel_size, z * voxel_size)
                    if is_point_in_triangle(tri, p, voxel_size):
                        surface_grid.set(x, y, z)
                        
    # 3. Interior Filling
    voxels_ = BitGrid(grid_size)
    voxels_buffer_ = BitGrid(grid_size)
    
    # Pass 1: Axis-aligned filling (Bulk)
    for z in range(grid_size.z):
        for y in range(grid_size.y):
            in_solid = False
            for x in range(grid_size.x):
                if surface_grid.get(x, y, z):
                    in_solid = not in_solid
                if in_solid:
                    voxels_.set(x, y, z)
                    
    # Pass 2: Flood Fill for holes
    queue = Queue()
    for x, y, z in surface_grid.get_all_set():
        queue.push((x, y, z))
        
    while not queue.empty():
        cx, cy, cz = queue.pop()
        for dx, dy, dz in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]:
            nx, ny, nz = cx + dx, cy + dy, cz + dz
            if is_in_bounds(nx, ny, nz, grid_size):
                if not voxels_.get(nx, ny, nz) and not surface_grid.get(nx, ny, nz):
                    voxels_buffer_.set(nx, ny, nz)
                    queue.push((nx, ny, nz))
                    
    # Final Inversion
    for i in range(len(voxels_.data)):
        voxels_.data[i] = voxels_buffer_.data[i] ^ (~0)
        
    return voxels_

def is_triangle_in_xy_projection(tri, y, z, voxel_size):
    p_xy = vec2(y * voxel_size, z * voxel_size)
    # Sources do not provide the exact 2D overlap check implementation.
    return True 

def is_point_in_triangle(tri, p, voxel_size):
    n = cross(tri.v1 - tri.v0, tri.v2 - tri.v0)
    d = -dot(n, tri.v0)
    dist = dot(n, p) + d
    if abs(dist) > voxel_size / 2:
        return False
    
    e0 = tri.v1 - tri.v0
    e1 = tri.v2 - tri.v1
    e2 = tri.v0 - tri.v2
    
    n0 = cross(e0, n)
    n1 = cross(e1, n)
    n2 = cross(e2, n)
    
    d0 = -dot(n0, tri.v0)
    d1 = -dot(n1, tri.v1)
    d2 = -dot(n2, tri.v2)
    
    return (dot(n0, p) + d0 >= 0) and (dot(n1, p) + d1 >= 0) and (dot(n2, p) + d2 >= 0)

# Helper definitions
class vec3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
class vec2:
    def __init__(self, x, y): self.x, self.y = x, y
def dot(a, b): return a.x*b.x + a.y*b.y + a.z*b.z
def dot2d(a, b): return a.x*b.x + a.y*b.y
def cross(a, b): return vec3(a.y*b.z - a.z*b.y, a.z*b.x - a.x*b.z, a.x*b.y - a.y*b.x)
class BitGrid:
    def __init__(self, size): self.size = size; self.data = [0] * (size.x * size.y * size.z)
    def set(self, x, y, z): self.data[x + y*self.size.x + z*self.size.x*self.size.y] |= 1
    def get(self, x, y, z): return (self.data[x + y*self.size.x + z*self.size.x*self.size.y] >> 0) & 1
    def get_all_set(self): return [(i % self.size.x, (i // self.size.x) % self.size.y, i // (self.size.x * self.size.y)) for i, v in enumerate(self.data) if v]
class Queue:
    def __init__(self): self.q = []
    def push(self, item): self.q.append(item)
    def pop(self): return self.q.pop(0)
    def empty(self): return len(self.q) == 0
def compute_grid_size(bbox_min, bbox_max, voxel_size):
    return vec3((bbox_max.x - bbox_min.x) // voxel_size + 1, (bbox_max.y - bbox_min.y) // voxel_size + 1, (bbox_max.z - bbox_min.z) // voxel_size + 1)
def compute_triangle_bbox(tri, voxel_size):
    mins = [min(v.x, v.y, v.z) for v in [tri.v0, tri.v1, tri.v2]]
    maxs = [max(v.x, v.y, v.z) for v in [tri.v0, tri.v1, tri.v2]]
    return type('BBox', (), {'x_min': mins[0]//voxel_size, 'x_max': maxs[0]//voxel_size, 'y_min': mins[1]//voxel_size, 'y_max': maxs[1]//voxel_size, 'z_min': mins[2]//voxel_size, 'z_max': maxs[2]//voxel_size})()
def is_in_bounds(x, y, z, grid_size):
    return 0 <= x < grid_size.x and 0 <= y < grid_size.y and 0 <= z < grid_size.z
```

**Parameters:**
* `gridVoxelSize`: The physical size of one side of a voxel [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Planar Test Edge Cases:** When the dot product plus distance equals zero, a specific normal direction check determines inclusion: $n.x > 0.0$ or $(n.x == 0.0 \land n.y < 0.0)$ [S1, unofficial].
* **Normal Flipping:** If the triangle normal's z-component is negative, the XY plane edge normals are flipped. If the y-component is negative, the ZX plane edge normals are flipped [S1, unofficial].
* **Precision and Boundaries:** Care must be taken with floating-point comparisons near voxel boundaries to avoid gaps or overlaps in the final bitmask [G].

## Subtopics

### [Single-pass GPU Solid Voxelization](3323-single-pass-gpu-solid-voxelization.md)
A GPU-based algorithm that voxelize solid geometry in a single render pass, producing point-sampled or over/under-conservative results. It solves the problem of efficiently converting watertight meshes into volumetric data by exploiting the Jordan Curve Theorem: a point is interior if a ray from it intersects the surface an odd number of times. Unlike [8, 10], this method is accurate even if an arbitrary number of fragments fall in a single voxel and does not require front/back face separation. It builds on [Solid Voxelization](1695-solid-voxelization.md) but achieves significant speedups by extracting multiple layers per pass.

### Solid Voxelization via Filter-and-Refine
A technique to fill the interior of a mesh to create a solid voxel representation [S2, unofficial]. It uses a filter-and-refine strategy to optimize basic flood fill, avoiding the need for watertight meshes [S2, unofficial].
* **Filter Step:** Fill voxels along each axis [S2, unofficial]. For a fixed pair of coordinates (e.g., fix x, y), enumerate z in $[0, \text{maxz}]$ [S2, unofficial]. If a voxel $(x,y,z)$ is occupied (by surface voxelization), stop filling in that direction [S2, unofficial]. This marks all voxels visible along that axis [S2, unofficial].
* **Refine Step:** Identify 'holes'—voxels that are not marked as visible along any axis [S2, unofficial]. Perform a standard flood fill specifically on these remaining empty voxels to fill the interior [S2, unofficial].
* **Complexity:** Time complexity is $O(n)$, where $n$ is the total number of voxels in the bounding box of the whole mesh [S2, unofficial].
* **Comparison:** Advantage over Basic Flood Fill: The filter-and-refine strategy is much more efficient than basic flood fill because the filter step uses lightweight operations, and the subsequent flood fill is only applied to a small number of holes [S2, unofficial].

## Operating characteristics
* **Time Complexity:** $O(m \cdot c)$ for surface voxelization, where $m$ is the number of triangles and $c$ is a factor such as the voxel number in the bounding box or constant overhead of BFS [S2, unofficial]. The flood fill component is $O(n)$, where $n$ is the number of voxels in the bounding box [S2, unofficial].
* **Memory Complexity:** Proportional to the volume of the bounding box ($N_x \cdot N_y \cdot N_z$) for the bitmask grid [S2, unofficial].
* **Strengths:** Produces watertight volumes; handles non-watertight meshes via flood-fill/refine; efficient on GPU via parallel triangle processing [S1, unofficial, S2, unofficial].
* **Limitations:** Performance degrades with very large bounding boxes relative to mesh density; requires careful handling of edge cases in planar tests [S1, unofficial].
* **When to use:** When a solid interior representation is required for volume rendering or collision detection [S2, unofficial].
* **When not to use:** When only surface representation is needed (use Surface Voxelization); when memory is extremely constrained and the mesh is sparse [G].

## Background
Solid Voxelization is a technique to fill the interior of a mesh with voxels [S2, unofficial]. It implements the algorithm from doi:10.1145/1882261.1866201 [S1, unofficial]. It is often used in conjunction with surface voxelization to create a complete volumetric representation [S2, unofficial]. The method has been implemented in various GPU-based voxelizers [S1, unofficial] and optimized with filter-and-refine strategies [S2, unofficial].

## Sources
[S1] gpu-voxelizer, https://github.com/ivandrodri/gpu-voxelizer, fetched 2026-10-26
[S2] voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27
