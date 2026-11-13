# Solid Voxelization based on Octree

Also called: Polygon-to-Octree Rasterization, RFVoxelOctreeBuilder, voxelization via bounding box overlap

## Overview
Solid Voxelization based on Octree converts a polygon mesh (specifically OBJ format) into a hierarchical octree representation to determine the solid, empty, and distance-to-surface status of leaf nodes [S1, S2, unofficial]. It differs from fixed-grid voxelization by using adaptive spatial subdivision, allowing for efficient representation of sparse scenes or high-resolution details where a uniform grid would be prohibitively large [S2, unofficial]. Unlike [Solid Voxelization via Ray Casting](6829-solid-voxelization-via-ray-casting.md), which casts rays per pixel to find intervals, this method recursively partitions space and assigns triangles to nodes, using ray sampling only for ambiguous internal empty leaves [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh in OBJ format (vertices, faces). Note: Only supports OBJ files with vertices ordered in triangles [v0,v1,v2] [S2, unofficial].
*   **Output:** An octree structure where each leaf node contains a distance to the surface (negative for inside, positive for outside) and type (Empty, Internal, Leaf) [S1, S2, unofficial].

**Data Structures:**

1.  **Octree Node (`RFVoxelOctreeNode`)** [S2, unofficial]:
    *   `m_kWorldAabb`: Axis-aligned bounding box in world space (`RFAABB3d`).
    *   `m_kBuildAabb`: Axis-aligned bounding box in build/grid space (`RFAABB3d` with integer min/max).
    *   `m_pkChildren`: Array of 8 pointers to child nodes.
    *   `m_eNodeType`: Enum indicating `NT_EMPTY_LEAF`, `NT_LEAF`, or `NT_INTERNAL`.
    *   `m_fDistanceToSurface`: Double precision distance to the closest triangle surface.
    *   *Alternative Layout (from [S1, unofficial]):* `vx_node` uses `uint32_t child_pointers[8]`, `uint8_t valid_masks`, `uint8_t leaf_mask`, `uint12_t padding`.

2.  **Octree Container (`RFVoxelOctree`)** [S2, unofficial]:
    *   `m_uiOctreeDim`: Dimension of the voxelization grid (next power of two).
    *   `m_fVoxelSize`: Size of a single voxel.
    *   `m_kOctreeOrigin`: World space origin of the octree.
    *   `m_kDataDimension`: Integer dimensions (`RFVector3i`).
    *   `m_pkRoot`: Pointer to the root node.
    *   `m_uiNumberOfLeafs`, `m_uiNumberOfEmptyLeafs`, `m_uiNumberOfInternalNodes`: Counters.

3.  **Voxel Data (`vx_voxel`)** [S1, unofficial]:
    *   `int8_t normals[3]`: Normal vector multiplied by 128.
    *   `uint8_t colors[4]`: Color (typically white 255,255,255,255).
    *   Padding fields.

**Parameters:**
*   `kOctreeDim` (or `detail`): The requested dimension of the voxelization grid. Default is 64 [S2, unofficial]. In [S1, unofficial], `detail` is the logarithmic resolution; typical value is 8.
*   `fMinCoverage`: Threshold for ray coverage to classify an internal empty leaf as solid. Default is 0.9 (header) or 0.8 (implementation) [S2, unofficial].
*   `fBorderSize`: Default 0.0, not used in provided logic [S2, unofficial].

**Algorithm Steps:**

1.  **Parse Mesh:** Read the OBJ file. Count vertices, UVs, normals, and faces. Track the bounding box min/max coordinates. Load vertices and faces into memory arrays. Decrement face indices by 1 to convert from 1-based OBJ indices to 0-based array indices [S1, S2, unofficial].

2.  **Calculate Grid Dimensions:**
    *   Find the longest axis of the mesh AABB: $kDim = kMax - kMin$.
    *   Calculate the next power of two for the octree dimension ($m\_uiOctreeDim$). If input is 0, return 2. Otherwise, decrement, then bitwise OR with right shifts of 16, 8, 4, 2, and 1 bits, then increment [S2, unofficial].
    *   Calculate voxel size: $m\_fVoxelSize = fMaxDim / m\_uiOctreeDim$.
    *   Calculate world size: $fOctreeWorldSize = m\_uiOctreeDim \times m\_fVoxelSize$.
    *   Set origin: $m\_kOctreeOrigin = kMeshAabb.GetCenter() - RFVector3d(fOctreeWorldSize / 2.0f)$ [S2, unofficial].
    *   *Alternative Normalization ([S1, unofficial]):* Calculate offset and scale to normalize model into unit cube $[0,1]^3$: $offset = -min$, $scale = 1.0 / max(width, height, depth)$. Grid step size: $step = 1.0 / (1 \ll detail)$ [S1, unofficial].

3.  **Initialize Root Node:**
    *   Create root `RFVoxelOctreeNode`.
    *   Set `m_kWorldAabb` from `m_kOctreeOrigin` to `m_kOctreeOrigin + RFVector3d(fOctreeWorldSize)`.
    *   Set `m_kBuildAabb.kMin` to $(0,0,0)$ and `kMax` to $(m\_uiOctreeDim, m\_uiOctreeDim, m\_uiOctreeDim)$.
    *   Set `m_kDataDimension` to `RFVector3i(m_uiOctreeDim)` [S2, unofficial].
    *   *Alternative ([S1, unofficial]):* Initialize with allocated size 4096, size 1, leaf_mask and valid_masks to 0.

4.  **Recursive Subdivision and Triangle Assignment:**
    For a parent node, create 8 children (octants $x,y,z \in \{0,1\}$).
    *   **Child AABB:**
        *   Build AABB: $kMin = parent.kMin + (iAabbDim \cdot x, iAabbDim \cdot y, iAabbDim \cdot z)$, $kMax = kMin + iAabbDim$.
        *   World AABB: Calculate using `m_kOctreeOrigin` and `m_fVoxelSize` [S2, unofficial].
    *   **Triangle Assignment:**
        *   For each triangle in the parent's list, check if `TriangleAabbIntersects(kTriangleVerts, kWorldAabb)`. If true, add the triangle index to the child's list [S2, unofficial].
        *   *Alternative ([S1, unofficial]):* Expand triangle bounding box by one step size. Iterate through grid cells within the expanded box. Check overlap using `triBoxOverlap`. If overlapping, retrieve or create the voxel.

5.  **Classify Child Node:**
    *   If child triangle list is empty: Set type to `NT_EMPTY_LEAF` [S2, unofficial].
    *   If `iAabbDim == 1` (leaf level): Set type to `NT_LEAF` and call `UpdateLeafNodeDistanceValue` [S2, unofficial].
    *   Otherwise: Set type to `NT_INTERNAL` and recursively call `Subdivide` [S2, unofficial].

6.  **Update Leaf Distance Value (`NT_LEAF`):**
    *   Initialize $fMinDist$ to max double.
    *   For each triangle in the leaf's list:
        *   Compute normal: $kNormal = VectorCross(kTriangleVerts[2] - kTriangleVerts[0], kTriangleVerts[1] - kTriangleVerts[0])$.
        *   Find closest point on triangle to leaf center: $kClosestPoint = ClosestPointOnTriangle(kTriangleVerts, leaf\_center)$.
        *   Compute delta: $kDelta = kClosestPoint - leaf\_center$.
        *   Compute sign: $fSign = (VectorDot(kDelta, kNormal) < 0.0f) ? -1.0f : 1.0f$.
        *   Compute distance: $fDist = VectorLength(kDelta)$.
        *   If $fDist < |fMinDist|$, set $fMinDist = fDist \times fSign$.
    *   Store $fMinDist$ in `m_fDistanceToSurface` [S2, unofficial].

7.  **Update Internal Empty Leaf Signs (`NT_EMPTY_LEAF`):**
    *   For `NT_INTERNAL` nodes, recurse to children.
    *   For `NT_EMPTY_LEAF` nodes:
        *   Cast rays from node center using precomputed sample directions (200 directions generated via Hammersley2D and UniformSphericalSampling) [S2, unofficial].
        *   Count hits (`uiCoverage`) and misses (`uiMisses`).
        *   **Stopping Heuristic:** Stop if `uiMisses >= size/4` (25% of total samples) [S2, unofficial].
        *   Calculate coverage: $fCoverage = uiCoverage / size$.
        *   If $fCoverage \ge fMinCoverage$, set `m_fDistanceToSurface = -1.0f` (solid/inside).
        *   Else, set `m_fDistanceToSurface = 1.0f` (empty/outside) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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

def radical_inverse_vdC(n):
    # Bit reversal for 32-bit integer
    n = ((n & 0x55555555) << 1) | ((n & 0xAAAAAAAA) >> 1)
    n = ((n & 0x33333333) << 2) | ((n & 0xCCCCCCCC) >> 2)
    n = ((n & 0x0F0F0F0F) << 4) | ((n & 0xF0F0F0F0) >> 4)
    n = ((n & 0x00FF00FF) << 8) | ((n & 0xFF00FF00) >> 8)
    n = ((n & 0x0000FFFF) << 16) | ((n & 0xFFFF0000) >> 16)
    return n * 2.3283064365386963e-10

def hammersley_2d(i, N):
    x = i / N
    y = radical_inverse_vdC(i)
    return (x, y)

def uniform_spherical_sampling(u, v):
    phi = v * 2 * 3.14159265358979323846
    cos_theta = 1 - u
    sin_theta = (1 - cos_theta * cos_theta) ** 0.5
    return (cos(phi) * sin_theta, sin(phi) * sin_theta, cos_theta)

def closest_point_on_triangle(p, v0, v1, v2):
    # Implementation of closest point on triangle to point p
    # Returns the closest point on the triangle surface
    pass

def solid_voxelization(mesh_vertices, mesh_faces, octree_dim):
    # 1. Parse and Normalize
    # Calculate AABB of mesh
    min_coord = min(mesh_vertices)
    max_coord = max(mesh_vertices)
    width = max_coord[0] - min_coord[0]
    height = max_coord[1] - min_coord[1]
    depth = max_coord[2] - min_coord[2]
    max_dim = max(width, height, depth)
    
    # Calculate dimensions
    dim = next_power_of_two(octree_dim)
    voxel_size = max_dim / dim
    world_size = dim * voxel_size
    origin = (min_coord + max_coord) / 2 - world_size / 2
    
    # 2. Initialize Root
    root = Node()
    root.world_aabb = AABB(origin, origin + world_size)
    root.build_aabb = AABB((0,0,0), (dim, dim, dim))
    root.triangles = mesh_faces # Indices
    
    # 3. Subdivide
    subdivide(root, dim)
    
    return root

def subdivide(node, current_dim):
    if current_dim == 1:
        update_leaf_distance(node)
        node.type = NT_LEAF
        return

    # Create 8 children
    children = []
    for x in range(2):
        for y in range(2):
            for z in range(2):
                child = Node()
                child.parent = node
                child.dim = current_dim // 2
                
                # Calculate AABBs
                build_min = node.build_aabb.min + (child.dim * x, child.dim * y, child.dim * z)
                build_max = build_min + (child.dim, child.dim, child.dim)
                child.build_aabb = AABB(build_min, build_max)
                
                # World AABB calculation would use origin and voxel_size
                # child.world_aabb = ...
                
                # Assign triangles
                for tri_idx in node.triangles:
                    if triangle_aabb_intersects(mesh_faces[tri_idx], child.world_aabb):
                        child.triangles.append(tri_idx)
                
                children.append(child)

    # Classify children
    for child in children:
        if not child.triangles:
            child.type = NT_EMPTY_LEAF
            update_empty_leaf_sign(child)
        else:
            child.type = NT_INTERNAL
            subdivide(child, child.dim)
        
        node.children.append(child)

def update_leaf_distance(node):
    leaf_center = node.build_aabb.center() # In world coords
    min_dist = float('inf')
    
    for tri_idx in node.triangles:
        v0, v1, v2 = mesh_faces[tri_idx]
        normal = cross(v2 - v0, v1 - v0)
        closest = closest_point_on_triangle(leaf_center, v0, v1, v2)
        delta = closest - leaf_center
        sign = -1.0 if dot(delta, normal) < 0 else 1.0
        dist = length(delta)
        
        if dist < abs(min_dist):
            min_dist = dist * sign
            
    node.distance_to_surface = min_dist

def update_empty_leaf_sign(node):
    # Precomputed sample directions (200)
    samples = []
    for i in range(200):
        u, v = hammersley_2d(i, 200)
        samples.append(uniform_spherical_sampling(u, v))
        
    center = node.build_aabb.center()
    coverage = 0
    misses = 0
    
    for direction in samples:
        # Cast ray from center in direction
        # Check intersection with any triangle in parent's list? 
        # Or just check if ray hits mesh?
        # The source implies checking coverage against the mesh geometry
        if ray_hits_mesh(center, direction, mesh_faces):
            coverage += 1
        else:
            misses += 1
            
        if misses >= 200 / 4: # Heuristic stop
            break
            
    f_coverage = coverage / 200
    if f_coverage >= 0.8: # fMinCoverage
        node.distance_to_surface = -1.0
    else:
        node.distance_to_surface = 1.0
```

## Subtopics

### [Octree-based 3D Voxelization with SAT Pruning](2588-octree-based-3d-voxelization-with-sat-pruning.md)
Uses Separating Axis Theorem for pruning empty bounding volumes instead of simple AABB intersection, offering robustness and performance improvements.

### [Robust Inside and Outside Solid Voxelization](3376-robust-inside-and-outside-solid-voxelization.md)
Handles non-watertight meshes using cube map rendering and visibility propagation, replacing the ray-casting sign test with a GPU-accelerated heuristic.

### [Solid Voxelization via Ray Casting](6829-solid-voxelization-via-ray-casting.md)
Computes solid intervals per pixel via ray-mesh intersection, differing by explicitly finding intervals rather than relying on spatial partitioning pruning.

### [Hierarchical Inside/Outside Propagation](9772-hierarchical-inside-outside-propagation.md)
Post-processing step that corrects inside/outside status by propagating boundary crossings along the +x direction, ensuring consistency without per-voxel ray queries.

### [Solid Voxelization based on Quadtree](13750-solid-voxelization-based-on-quadtree.md)
Uses 2D quadtree structures on projected planes to store intersection distances, reducing memory usage for huge models or high resolutions.

## Operating characteristics
*   **Time Complexity:** Depends on the number of triangles and octree depth. Subdivision is recursive, and triangle assignment requires AABB intersection tests for each triangle at each level. Leaf distance calculation requires closest-point tests for all triangles in the leaf.
*   **Memory Complexity:** Adaptive. Uses an octree structure, which is more memory-efficient than a fixed grid for sparse scenes but requires overhead for node pointers and metadata [S2, unofficial].
*   **Performance:** On an Apple M1 Pro, building the octree for `Armadillo.obj` at dimension 64 took ~484ms before optimization and ~77ms after output optimization (total build time not explicitly separated after optimization, but output was 77ms) [S2, unofficial].
*   **Strengths:** Adaptive resolution, handles large bounding boxes efficiently, supports OBJ format directly.
*   **Limitations:** Only supports OBJ files with triangle-ordered vertices. Empty leaf classification uses a heuristic that may be inaccurate for nodes near the surface (early stop if misses > 25%) [S2, unofficial]. Bounding box expansion by 'step' is required to avoid precision issues at boundaries [S1, unofficial].

## Background
The method originates from work by Peter Trier Mikkelsen, with tutorials available via Alexandra [S2, unofficial]. It is implemented in repositories such as `itsyouyi/Voxelizer` and `rools/voxl` [S1, S2, unofficial]. It builds on [Mesh Voxelization](6514-mesh-voxelization.md) by providing a specific octree-based implementation for solid voxelization.

## Sources
[S1] rools/voxl, https://github.com/rools/voxl, fetched 2026-10-26 (unofficial)
[S2] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
