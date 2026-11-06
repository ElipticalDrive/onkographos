# Solid Voxelization based on Octree

Also called: RFVoxelOctreeBuilder

## Overview
Solid Voxelization based on Octree is a method for converting triangular surface meshes into a solid voxel representation using an adaptive octree structure [S1, unofficial]. It determines the solid or empty status of leaf nodes by combining ray casting for sign determination with closest-point distance calculations [S1, unofficial]. Unlike fixed-grid voxelization, this method adapts resolution to the geometry, potentially reducing node count for sparse scenes, while its reliance on ray casting for sign determination distinguishes it from exact geometric intersection methods like [Octree-based 3D Voxelization with SAT Pruning](2588-octree-based-3d-voxelization-with-sat-pruning.md) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh in Wavefront OBJ format (vertices ordered in triangles $v_0, v_1, v_2$) [S1, unofficial].
*   **Output:** An octree structure where each leaf node contains a signed distance to the surface [S1, unofficial].

**Data Structures:**
*   `RFVoxelOctreeNode`:
    *   `m_kWorldAabb`: `RFAABB3d` (double-precision axis-aligned bounding box in world space).
    *   `m_kBuildAabb`: `RFAABB3d` (integer min/max coordinates in build space).
    *   `m_pkChildren`: Array of 8 pointers to child nodes (null for leaves).
    *   `m_eNodeType`: Enum (`NT_EMPTY_LEAF`, `NT_LEAF`, `NT_INTERNAL`).
    *   `m_fDistanceToSurface`: Double-precision signed distance.
*   `RFVoxelOctree`:
    *   `m_uiOctreeDim`: Unsigned integer, the dimension of the voxelization grid (next power of two).
    *   `m_fVoxelSize`: Double, size of a single voxel.
    *   `m_kOctreeOrigin`: `RFVector3d`, world-space origin of the octree.
    *   `m_kDataDimension`: `RFVector3i`, dimensions of the data grid.
    *   `m_pkRoot`: Pointer to the root node.
    *   `m_uiNumberOfLeafs`, `m_uiNumberOfEmptyLeafs`, `m_uiNumberOfInternalNodes`: Counts [S1, unofficial].

**Numbered Steps:**

1.  **Calculate Octree Dimension:**
    Compute the next power of two for the requested dimension $D$. If $D=0$, result is 2. Otherwise:
    $$ D' = D - 1 $$
    $$ D' = D' \ | \ (D' \gg 16) \ | \ (D' \gg 8) \ | \ (D' \gg 4) \ | \ (D' \gg 2) \ | \ (D' \gg 1) $$
    $$ m\_uiOctreeDim = D' + 1 $$
    [S1, unofficial]

2.  **Determine Voxel Size and Origin:**
    *   Compute mesh AABB: $kMin, kMax$.
    *   Find longest axis: $kDim = kMax - kMin$.
    *   Set $m\_fVoxelSize = \max(kDim) / m\_uiOctreeDim$.
    *   Set $fOctreeWorldSize = m\_uiOctreeDim \times m\_fVoxelSize$.
    *   Set $m\_kOctreeOrigin = \text{Center}(kMeshAABB) - \text{RFVector3d}(fOctreeWorldSize / 2.0)$.
    [S1, unofficial]

3.  **Initialize Root Node:**
    *   Create `RFVoxelOctreeNode` for root.
    *   Set `m_kWorldAabb` from $m\_kOctreeOrigin$ to $m\_kOctreeOrigin + \text{RFVector3d}(fOctreeWorldSize)$.
    *   Set `m_kBuildAabb.kMin` to $(0,0,0)$ and `kMax` to $(m\_uiOctreeDim, m\_uiOctreeDim, m\_uiOctreeDim)$.
    *   Set `m_kDataDimension` to $\text{RFVector3i}(m\_uiOctreeDim)$.
    [S1, unofficial]

4.  **Recursive Subdivision:**
    For a parent node with build-space dimension $iAabbDim$:
    *   Create 8 children. For each child $(x,y,z)$ where $x,y,z \in \{0,1\}$:
        *   `child.kMin` = `parent.kMin` + $(iAabbDim \cdot x, iAabbDim \cdot y, iAabbDim \cdot z)$.
        *   `child.kMax` = `child.kMin` + $iAabbDim$.
        *   Calculate `child.m_kWorldAabb` using $m\_kOctreeOrigin$ and $m\_fVoxelSize$.
    *   **Assign Triangles:** For each triangle in the parent's list, if `TriangleAabbIntersects(triangle_verts, child.m_kWorldAabb)` is true, add the triangle index to the child's list.
    *   **Classify Child:**
        *   If child triangle list is empty: Set type to `NT_EMPTY_LEAF`.
        *   If $iAabbDim == 1$: Set type to `NT_LEAF` and call `UpdateLeafNodeDistanceValue`.
        *   Otherwise: Set type to `NT_INTERNAL` and recursively call Subdivide.
    [S1, unofficial]

5.  **Update Leaf Distance (Solid Leaves):**
    For a leaf node ($iAabbDim=1$):
    *   Initialize $fMinDist = \infty$.
    *   For each triangle in the leaf's list:
        *   Compute normal: $kNormal = \text{Cross}(v_2-v_0, v_1-v_0)$.
        *   Find closest point on triangle to leaf center: $kClosestPoint = \text{ClosestPointOnTriangle}(verts, leaf\_center)$.
        *   Compute delta: $kDelta = kClosestPoint - leaf\_center$.
        *   Compute sign: $fSign = (\text{Dot}(kDelta, kNormal) < 0.0) ? -1.0 : 1.0$.
        *   Compute distance: $fDist = \text{Length}(kDelta)$.
        *   If $fDist < |fMinDist|$, set $fMinDist = fDist \times fSign$.
    *   Store $fMinDist$ in `m_fDistanceToSurface`.
    [S1, unofficial]

6.  **Update Internal Empty Leaf Signs:**
    For `NT_INTERNAL` nodes, recurse to children. For `NT_EMPTY_LEAF` nodes:
    *   Cast rays from node center using 200 precomputed directions ($m\_kSampleDirections$).
    *   Count hits ($uiCoverage$) and misses ($uiMisses$).
    *   Stop if $uiMisses \ge size/4$ (where $size$ is total samples, 200).
    *   If $fCoverage = uiCoverage / size \ge fMinCoverage$, set `m_fDistanceToSurface` = -1.0 (solid).
    *   Else, set `m_fDistanceToSurface` = 1.0 (empty).
    [S1, unofficial]

7.  **Generate Sample Directions:**
    In constructor, for $i$ from 0 to 199:
    *   $kRand = \text{Hammersley2D}(i, 200)$.
    *   $direction = \text{UniformSphericalSampling}(kRand.tX, kRand.tY)$.
    *   Store in $m\_kSampleDirections$.
    [S1, unofficial]

**Formulas:**

*   **Hammersley2D:**
    $$ x = i/N $$
    $$ y = \text{RadicalInverseVdC}(i) $$
    Where `RadicalInverseVdC` performs bit reversal on `uiBits`: swap 16-bit halves, swap 8-bit halves, swap 4-bit nibbles, swap 2-bit pairs, swap 1-bit pairs, then multiply by $2.3283064365386963 \times 10^{-10}$.
    [S1, unofficial]

*   **UniformSphericalSampling:**
    $$ \phi = v \cdot 2\pi $$
    $$ \cos\theta = 1 - u $$
    $$ \sin\theta = \sqrt{1 - \cos^2\theta} $$
    Result: $(\cos\phi \sin\theta, \sin\phi \sin\theta, \cos\theta)$.
    [S1, unofficial]

**Parameters:**
*   `kOctreeDim`: Requested dimension of the voxelization grid. Default: 64 [S1, unofficial].
*   `fMinCoverage`: Threshold for ray coverage to classify an internal empty leaf as solid. Header default: 0.9; Implementation call default: 0.8 [S1, unofficial].
*   `fBorderSize`: Passed to `CreateOctree`, default 0.0, not used in logic [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Empty leaf classification uses a stopping heuristic: if misses exceed 25% of total samples (size/4), the loop breaks early. This may lead to inaccurate coverage calculation for nodes near the surface [S1, unofficial].
*   The method only supports OBJ files with vertices ordered in triangles [v0,v1,v2] [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def next_power_of_two(n):
    if n == 0:
        return 2
    n -= 1
    n |= n >> 16
    n |= n >> 8
    n |= n >> 4
    n |= n >> 2
    n |= n >> 1
    return n + 1

def radical_inverse_vdc(bits):
    bits = ((bits * 0x0202020202) & 0x010884422) * 0x101010101 >> 32
    return bits * 2.3283064365386963e-10

def hammersley_2d(i, n):
    x = i / n
    y = radical_inverse_vdc(i)
    return (x, y)

def uniform_spherical_sampling(u, v):
    phi = v * 2 * math.pi
    cos_theta = 1 - u
    sin_theta = math.sqrt(1 - cos_theta * cos_theta)
    return (math.cos(phi) * sin_theta, math.sin(phi) * sin_theta, cos_theta)

def closest_point_on_triangle(v0, v1, v2, p):
    # Standard closest point on triangle algorithm [G]
    e1 = (v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2])
    e2 = (v2[0]-v0[0], v2[1]-v0[1], v2[2]-v0[2])
    d = (p[0]-v0[0], p[1]-v0[1], p[2]-v0[2])
    b11 = e1[0]*e1[0] + e1[1]*e1[1] + e1[2]*e1[2]
    b12 = e1[0]*e2[0] + e1[1]*e2[1] + e1[2]*e2[2]
    b22 = e2[0]*e2[0] + e2[1]*e2[1] + e2[2]*e2[2]
    b1d = e1[0]*d[0] + e1[1]*d[1] + e1[2]*d[2]
    b2d = e2[0]*d[0] + e2[1]*d[1] + e2[2]*d[2]
    det = b11 * b22 - b12 * b12
    if det == 0.0: return v0 # Degenerate triangle [G]
    u = (b22 * b1d - b12 * b2d) / det
    v = (b11 * b2d - b12 * b1d) / det
    if u < 0.0:
        u = 0.0
        v = max(0.0, -b1d / b11)
    elif v < 0.0:
        v = 0.0
        u = max(0.0, -b2d / b22)
    elif u + v > 1.0:
        t = (b11 + b22 - 2*b12)
        if t > 0.0:
            t = (b11 + b2d - b12) / t
        else:
            t = 0.5
        u = 1.0 - t
        v = 1.0 - u
    return (v0[0] + u*e1[0] + v*e2[0], v0[1] + u*e1[1] + v*e2[1], v0[2] + u*e1[2] + v*e2[2])

def triangle_aabb_intersects(v0, v1, v2, aabb_min, aabb_max):
    min_x = min(v0[0], v1[0], v2[0])
    max_x = max(v0[0], v1[0], v2[0])
    min_y = min(v0[1], v1[1], v2[1])
    max_y = max(v0[1], v1[1], v2[1])
    min_z = min(v0[2], v1[2], v2[2])
    max_z = max(v0[2], v1[2], v2[2])
    if max_x < aabb_min[0] or min_x > aabb_max[0]: return False
    if max_y < aabb_min[1] or min_y > aabb_max[1]: return False
    if max_z < aabb_min[2] or min_z > aabb_max[2]: return False
    return True

def ray_triangle_intersect(origin, direction, v0, v1, v2):
    # Möller–Trumbore algorithm [G]
    e1 = (v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2])
    e2 = (v2[0]-v0[0], v2[1]-v0[1], v2[2]-v0[2])
    h = (direction[1]*e2[2]-direction[2]*e2[1], direction[2]*e2[0]-direction[0]*e2[2], direction[0]*e2[1]-direction[1]*e2[0])
    a = e1[0]*h[0] + e1[1]*h[1] + e1[2]*h[2]
    if a < 0.0000001: return False
    f = 1.0 / a
    d = (origin[0]-v0[0], origin[1]-v0[1], origin[2]-v0[2])
    u = f * (d[0]*h[0] + d[1]*h[1] + d[2]*h[2])
    if u < 0.0 or u > 1.0: return False
    q = (d[1]*e2[2]-d[2]*e2[1], d[2]*e2[0]-d[0]*e2[2], d[0]*e2[1]-d[1]*e2[0])
    v = f * (direction[0]*q[0] + direction[1]*q[1] + direction[2]*q[2])
    if v < 0.0 or u + v > 1.0: return False
    t = f * (e2[0]*q[0] + e2[1]*q[1] + e2[2]*q[2])
    return t > 0.0000001

class VoxelOctreeNode:
    def __init__(self):
        self.world_aabb_min = None
        self.world_aabb_max = None
        self.build_aabb_min = None
        self.build_aabb_max = None
        self.children = [None] * 8
        self.node_type = None
        self.distance_to_surface = 0.0
        self.triangle_indices = []

def create_octree(mesh_vertices, mesh_faces, requested_dim):
    octree_dim = next_power_of_two(requested_dim)
    
    min_x = min(v[0] for v in mesh_vertices)
    max_x = max(v[0] for v in mesh_vertices)
    min_y = min(v[1] for v in mesh_vertices)
    max_y = max(v[1] for v in mesh_vertices)
    min_z = min(v[2] for v in mesh_vertices)
    max_z = max(v[2] for v in mesh_vertices)
    
    k_min = (min_x, min_y, min_z)
    k_max = (max_x, max_y, max_z)
    
    dim_x = max_x - min_x
    dim_y = max_y - min_y
    dim_z = max_z - min_z
    max_dim = max(dim_x, dim_y, dim_z)
    
    voxel_size = max_dim / octree_dim
    world_size = octree_dim * voxel_size
    
    center_x = (min_x + max_x) / 2
    center_y = (min_y + max_y) / 2
    center_z = (min_z + max_z) / 2
    
    origin_x = center_x - world_size / 2
    origin_y = center_y - world_size / 2
    origin_z = center_z - world_size / 2
    
    root = VoxelOctreeNode()
    root.world_aabb_min = (origin_x, origin_y, origin_z)
    root.world_aabb_max = (origin_x + world_size, origin_y + world_size, origin_z + world_size)
    root.build_aabb_min = (0, 0, 0)
    root.build_aabb_max = (octree_dim, octree_dim, octree_dim)
    
    # Precompute sample directions [S1, unofficial]
    sample_directions = []
    for i in range(200):
        u, v = hammersley_2d(i, 200)
        sample_directions.append(uniform_spherical_sampling(u, v))
    
    def subdivide(node, build_min, build_max, triangles):
        build_dim = build_max[0] - build_min[0]
        
        if not triangles:
            node.node_type = "NT_EMPTY_LEAF"
            node.triangle_indices = []
            return
        
        if build_dim == 1:
            node.node_type = "NT_LEAF"
            node.triangle_indices = triangles
            update_leaf_distance(node, mesh_vertices, mesh_faces, origin_x, origin_y, origin_z, voxel_size)
            return
        
        node.node_type = "NT_INTERNAL"
        node.triangle_indices = []
        
        for x in range(2):
            for y in range(2):
                for z in range(2):
                    child_min = (build_min[0] + build_dim * x, build_min[1] + build_dim * y, build_min[2] + build_dim * z)
                    child_max = (child_min[0] + build_dim, child_min[1] + build_dim, child_min[2] + build_dim)
                    
                    child = VoxelOctreeNode()
                    child.build_aabb_min = child_min
                    child.build_aabb_max = child_max
                    
                    child.world_aabb_min = (
                        origin_x + child_min[0] * voxel_size,
                        origin_y + child_min[1] * voxel_size,
                        origin_z + child_min[2] * voxel_size
                    )
                    child.world_aabb_max = (
                        origin_x + child_max[0] * voxel_size,
                        origin_y + child_max[1] * voxel_size,
                        origin_z + child_max[2] * voxel_size
                    )
                    
                    child_triangles = []
                    for idx in triangles:
                        v0 = mesh_vertices[mesh_faces[idx][0]]
                        v1 = mesh_vertices[mesh_faces[idx][1]]
                        v2 = mesh_vertices[mesh_faces[idx][2]]
                        if triangle_aabb_intersects(v0, v1, v2, child.world_aabb_min, child.world_aabb_max):
                            child_triangles.append(idx)
                    
                    child.triangle_indices = child_triangles
                    subdivide(child, child_min, child_max, child_triangles)
                    
                    child_idx = x * 4 + y * 2 + z
                    node.children[child_idx] = child

    all_triangles = list(range(len(mesh_faces)))
    subdivide(root, (0,0,0), (octree_dim, octree_dim, octree_dim), all_triangles)
    
    def update_empty_leaves(node):
        if node.node_type == "NT_INTERNAL":
            for child in node.children:
                if child:
                    update_empty_leaves(child)
        elif node.node_type == "NT_EMPTY_LEAF":
            coverage = 0
            misses = 0
            center_world = (
                origin_x + (node.build_aabb_min[0] + node.build_aabb_max[0]) / 2 * voxel_size,
                origin_y + (node.build_aabb_min[1] + node.build_aabb_max[1]) / 2 * voxel_size,
                origin_z + (node.build_aabb_min[2] + node.build_aabb_max[2]) / 2 * voxel_size
            )
            for dir in sample_directions:
                hit = False
                for idx in range(len(mesh_faces)):
                    v0 = mesh_vertices[mesh_faces[idx][0]]
                    v1 = mesh_vertices[mesh_faces[idx][1]]
                    v2 = mesh_vertices[mesh_faces[idx][2]]
                    if ray_triangle_intersect(center_world, dir, v0, v1, v2):
                        hit = True
                        break
                if hit:
                    coverage += 1
                else:
                    misses += 1
                if misses >= 50: # size/4 = 200/4 = 50 [S1, unofficial]
                    break
            
            if coverage / 200 >= 0.8: # fMinCoverage default 0.8 [S1, unofficial]
                node.distance_to_surface = -1.0
            else:
                node.distance_to_surface = 1.0

    update_empty_leaves(root)
    return root

def update_leaf_distance(node, mesh_vertices, mesh_faces, origin_x, origin_y, origin_z, voxel_size):
    build_min = node.build_aabb_min
    build_max = node.build_aabb_max
    center_build = (
        (build_min[0] + build_max[0]) / 2,
        (build_min[1] + build_max[1]) / 2,
        (build_min[2] + build_max[2]) / 2
    )
    center_world = (
        origin_x + center_build[0] * voxel_size,
        origin_y + center_build[1] * voxel_size,
        origin_z + center_build[2] * voxel_size
    )
    
    min_dist = float('inf')
    
    for idx in node.triangle_indices:
        v0 = mesh_vertices[mesh_faces[idx][0]]
        v1 = mesh_vertices[mesh_faces[idx][1]]
        v2 = mesh_vertices[mesh_faces[idx][2]]
        
        v0v1 = (v1[0]-v0[0], v1[1]-v0[1], v1[2]-v0[2])
        v0v2 = (v2[0]-v0[0], v2[1]-v0[1], v2[2]-v0[2])
        normal = (v0v1[1]*v0v2[2]-v0v1[2]*v0v2[1], v0v1[2]*v0v2[0]-v0v1[0]*v0v2[2], v0v1[0]*v0v2[1]-v0v1[1]*v0v2[0])
        
        closest = closest_point_on_triangle(v0, v1, v2, center_world)
        
        delta = (closest[0]-center_world[0], closest[1]-center_world[1], closest[2]-center_world[2])
        
        dot = delta[0]*normal[0] + delta[1]*normal[1] + delta[2]*normal[2]
        sign = -1.0 if dot < 0.0 else 1.0
        
        dist = (delta[0]**2 + delta[1]**2 + delta[2]**2) ** 0.5
        
        if dist < abs(min_dist):
            min_dist = dist * sign
            
    node.distance_to_surface = min_dist
```

## Subtopics

### [Octree-based 3D Voxelization with SAT Pruning](2588-octree-based-3d-voxelization-with-sat-pruning.md)
Uses Separating Axis Theorem for exact geometric intersection tests instead of ray casting, offering robustness and performance for arbitrary meshes. Choose when exact intersection is preferred over probabilistic ray coverage.

### [Robust Inside and Outside Solid Voxelization](3376-robust-inside-and-outside-solid-voxelization.md)
Uses cube map rendering and visibility propagation to handle non-watertight meshes and holes. Choose when the input mesh is not watertight or contains intersecting geometry.

## Operating characteristics
*   **Time Complexity:** Depends on octree depth and triangle distribution. Recursive subdivision and ray casting for empty leaves add overhead. Benchmark on Armadillo.obj (M1 Pro): BuildOctree ~484ms (before optimization), ~77ms (after optimization) for output [S1, unofficial].
*   **Memory Complexity:** Adaptive octree structure. Fewer nodes for sparse scenes compared to uniform grids, but requires storing triangle lists and child pointers [S1, unofficial].
*   **Strengths:** Adaptive resolution, handles sparse scenes efficiently, outputs solid voxel representation with signed distance [S1, unofficial].
*   **Limitations:** Only supports OBJ files with vertices ordered in triangles [S1, unofficial]. Ray casting for sign determination may be inaccurate for nodes near the surface due to early stopping heuristic (misses >= 25% of samples) [S1, unofficial].
*   **When to use:** When adaptive resolution is needed and input is a watertight OBJ mesh.
*   **When not to use:** For non-OBJ formats, non-watertight meshes, or when exact geometric intersection is required.

## Background
The method originates from work by Peter Trier Mikkelsen at Alexandra [S1, unofficial]. It is implemented in the `itsyouyi/Voxelizer` repository [S1, unofficial]. The video tutorial is available on YouTube [S1, unofficial].

## Sources
[S1] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
