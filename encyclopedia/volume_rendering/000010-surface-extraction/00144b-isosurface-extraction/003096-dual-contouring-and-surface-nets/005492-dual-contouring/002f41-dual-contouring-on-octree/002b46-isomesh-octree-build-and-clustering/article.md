# Isomesh Octree Build and Clustering

Also called: Voxel Level of Detail with Dual Contouring

## Overview
Isomesh Octree Build and Clustering is a hierarchical mesh generation method that extracts an isotopic triangular mesh from an implicit function defined over an octree structure [S1, S2, unofficial]. It solves the problem of surface extraction in sparse or hierarchical domains by recursively building an octree, clustering vertices at each node level (leaves, edges, faces, and cells) using Disjoint Set Union (DSU) and Quadratic Error Function (QEF) solvers, and merging geometric primitives while preserving material properties and minimizing reconstruction error [S1, S2, unofficial]. Unlike uniform grid methods, it adapts resolution locally, allowing for efficient Level of Detail (LOD) management where coarser branches store vertices that minimize the sum of their children's Q functions, enabling recursive vertex calculation for all levels of detail [S2, unofficial]. It sets itself apart from siblings like [Manifold Dual Contouring](2138-manifold-dual-contouring.md) by explicitly handling hierarchical LOD stitching via a "skirt" optimization and using DSU for topological correctness across irregular octree subdivisions [S1, S2, unofficial].

## How it works
The method operates in three main phases: Grid Filling, Octree Build, and Clustering/Mesh Generation.

### Inputs and Outputs
**Inputs:**
- An implicit function $f(\mathbf{p})$ and its gradient $\nabla f(\mathbf{p})$.
- A grid size $N$ (must be a power of two, $2 \le N \le 1024$) [S1, unofficial].
- A global scale factor $S_{global}$ and a local epsilon $\epsilon$ for contouring [S1, unofficial].

**Outputs:**
- A `Mesh` object containing vertices (position, normal, material, error, QEF state) and triangle indices.

### Data Structures

**UniformGrid:**
- `m_mat`: Raw array `Material[(N + 1)^3]`. Stores material types at grid vertices.
- `m_size`: Integer $N$.
- `m_halfSize`: Integer $N/2$.
- Indexing is Z-major: `idx = ((y + m_halfSize) * (N + 1) + (x + m_halfSize)) * (N + 1) + (z + m_halfSize)` [S1, unofficial].

**OctreeNode:**
- `m_vertices`: Array of vertex structures.
- `m_isLeaf`: Boolean.
- `m_children`: Array of 8 pointers to child nodes (if not leaf).
- `m_corners`: Array of 8 `Material` values (for leaves).
- `m_vertexMask`: 8-bit integer bitmask indicating which corners are non-empty [S1, unofficial].
- `m_surfaceIdx`: Index into the edge storage for edge vertices [S1, unofficial].

**Vertex Structure:**
- `m_position`: `glm::vec3` (or equivalent 3D vector).
- `m_normal`: `glm::vec3` (normalized).
- `m_material`: Enum/ID.
- `m_error`: Float (QEF evaluation at position).
- `m_qef`: QEF state object (for merging).
- `m_parent`: Pointer to the parent node's vertex (for clustering hierarchy) [S1, unofficial].

**Edge Storage:**
- Hash maps or spatial structures keyed by edge coordinates `(x, y, z)` storing `surfacePoint` and `surfaceNormal` for Hermite data extracted from the grid [S1, unofficial].

### Step-by-Step Method

#### 1. Grid Filling
Compute the implicit function values for all $(N+1)^3$ grid vertices.
1. Initialize `values` array of size $(N+1)^3$.
2. Iterate $y, x, z$ from $-m\_halfSize$ to $m\_halfSize$.
3. Compute global position $\mathbf{p} = \text{localToGlobal}(x, y, z)$.
4. Compute $v = f(\mathbf{p})$. Set `values[idx] = v`.
5. If $v > 0$, set `m_mat[idx] = Material::Empty`. Else, set `m_mat[idx] = f.material(\mathbf{p}, v)` [S1, unofficial].

#### 2. Edge Intersection Detection
Detect where the isosurface intersects grid edges to generate Hermite data (point + normal).
1. **X-axis edges:** Iterate $y, z$ from $-m\_halfSize$ to $m\_halfSize$, and $x$ from $-m\_halfSize$ to $m\_halfSize - 1$.
   - Let $idx1$ correspond to $(x, y, z)$ and $idx2$ to $(x+1, y, z)$.
   - If $\text{sign}(values[idx1]) \neq \text{sign}(values[idx2])$:
     - Compute intersection $p_x$ along the edge using linear interpolation or root finding: $p_x = \text{solver.findAlongX}(x_0, p_y, p_z, x_1, values[idx1], values[idx2], f)$.
     - Compute gradient $\mathbf{g} = \nabla f(\mathbf{p})$.
     - Compute offset $o = (p_x - x_0) / m\_gridStep$.
     - Determine material and store edge data in `m_edgeX` at $(x, y, z)$ [S1, unofficial].
2. **Y-axis and Z-axis edges:** Repeat similarly using `findAlongY`/`findAlongZ` and respective edge storages, iterating appropriate ranges ($x < m\_halfSize$ for X, etc.) [S1, unofficial].

#### 3. Octree Build
Recursively build the octree structure.
1. Call `buildNode(root, min_corner, size)`.
2. If `size == 1`:
   - Call `buildLeaf(node, min_corner)` [S1, unofficial].
3. Else:
   - `node->subdivide()`.
   - `child_size = size / 2`.
   - For $i$ from 0 to 7:
     - `child_min_corner = min_corner + child_size * kCellCornerOffset[i]`.
     - `buildNode(node->child(i), child_min_corner, child_size)` [S1, unofficial].

**Leaf Construction (`buildLeaf`):**
1. Get `corners` from `args.grid.materialsOfCell(args.grid.pointToIndex(min_corner))` [S1, unofficial].
2. Compute `vertex_mask`: bit $i$ is 1 if `corners[i] != Material::Empty` [S1, unofficial].
3. `node->setCorners(corners)`, `node->setVertexMask(vertex_mask)` [S1, unofficial].
4. Initialize `solver[4]`, `filter[4]`, `avg_normal[4]`.
5. Initialize `corner_used[8]` to false [S1, unofficial].

**Leaf Edge Processing & QEF Accumulation:**
1. Iterate $i$ from 0 to 11 (12 edges of a cube).
2. Get `idx = kMdcEdgeSetIndex[vertex_mask][i]`. If `idx < 0`, continue [S1, unofficial].
3. Get `storage` from `storages[kCellEdgeDirection[i]]`.
4. Calculate `edge_pos = min_corner + kCellCornerOffset[kCellEdgeEndpoint[i][0]]`.
5. Find edge iterator `iter = storage.findEdge(edge_pos.x, edge_pos.y, edge_pos.z)`.
6. Get `point = iter->surfacePoint()` and `normal = iter->surfaceNormal()` [S1, unofficial].
7. `solver[idx].addPlane(point, normal)` [S1, unofficial].
8. `avg_normal[idx] += normal` [S1, unofficial].
9. Determine `mat = iter->solidEndpointMaterial()` [S1, unofficial].
10. Determine `corner_idx`: if `iter->isLesserEndpointSolid()`, `corner_idx = kCellEdgeEndpoint[i][0]`, else `corner_idx = kCellEdgeEndpoint[i][1]` [S1, unofficial].
11. If `!corner_used[corner_idx]`:
    - `filter[idx].add(mat)` [S1, unofficial].
    - `corner_used[corner_idx] = true` [S1, unofficial].

**Leaf Vertex Solving:**
1. `surface_cnt = max(idx + 1)` from the edge loop [S1, unofficial].
2. `node->m_vertices.resize(surface_cnt)` [S1, unofficial].
3. `lower_bound = min_corner`, `upper_bound = min_corner + size` [S1, unofficial].
4. For $i$ from 0 to `surface_cnt - 1`:
   - `point = solver[i].solve(lower_bound, upper_bound)` [S1, unofficial].
   - `node->m_vertices[i].m_position = point` [S1, unofficial].
   - `node->m_vertices[i].m_normal = glm::normalize(avg_normal[i])` [S1, unofficial].
   - `node->m_vertices[i].m_material = filter[i].select()` [S1, unofficial].
   - `node->m_vertices[i].m_error = solver[i].eval(point)` [S1, unofficial].
   - `node->m_vertices[i].m_qef = solver[i].state()` [S1, unofficial].

#### 4. Clustering and Mesh Generation
Process the octree to merge vertices and generate the final mesh.

**Cell Clustering (`clusterCell`):**
1. If `node->isLeaf()`, return [S1, unofficial].
2. `sub_size = size / 2`.
3. For $i$ from 0 to 7:
   - `sub_pos = min_corner + sub_size * kCellCornerOffset[i]`.
   - `clusterCell(node->child(i), sub_pos, sub_size)` [S1, unofficial].
4. Initialize `DSU dsu` and `VertexArray vertices` [S1, unofficial].
5. Call `callSubClusterFace(0)`, `callSubClusterFace(1)`, `callSubClusterFace(2)` [S1, unofficial].
6. Call `callSubClusterEdge(0)`, `callSubClusterEdge(1)`, `callSubClusterEdge(2)` [S1, unofficial].

**Edge/Face Clustering Logic:**
1. In `clusterEdge` or `clusterFace`, iterate children using `kEdgeProcChildTable[D]` or `kFaceProcChildTable[D]`.
2. For each child node $n$:
   - If $n$ is subdivided, get specific child `sub[i] = n->child(k...[i][1])` and set `all_leaves = false`.
   - Else `sub[i] = n`.
3. If `all_leaves`:
   - Call `clusterEdgeLeaves<D>(nodes, dsu, vertices)` (or face equivalent) and return [S1, unofficial].
4. Otherwise, call `callSubClusterEdge(D)` or `callSubClusterFace(D)` [S1, unofficial].

**Vertex Merging in Cell:**
1. Map DSU sets to new indices.
2. Initialize `set_idx` vector of size `dsu.size()` with `kBadIndex`.
3. For $i$ from 0 to `dsu.size() - 1`:
   - `leader = dsu.getSetLeader(i)` [S1, unofficial].
   - If `set_idx[leader] == kBadIndex`:
     - `set_idx[leader] = new_cnt` [S1, unofficial].
     - `new_cnt++` [S1, unofficial].
   - `set_idx[i] = set_idx[leader]` [S1, unofficial].
4. `node->m_vertices.resize(new_cnt)` [S1, unofficial].

**Solving Clustered Vertices:**
1. Initialize `solver` vector of size `new_cnt` with copies of `orig_solver` [S1, unofficial].
2. Initialize `filter` and `avg_normal` vectors of size `new_cnt` [S1, unofficial].
3. For each vertex $v$ in `vertices`:
   - `idx = set_idx[v->m_surfaceIdx]` [S1, unofficial].
   - `v->m_surfaceIdx = kBadIndex` [S1, unofficial].
   - `solver[idx].merge(v->m_qef)` [S1, unofficial].
   - `avg_normal[idx] += v->m_normal` [S1, unofficial].
   - `filter[idx].add(v->m_material)` [S1, unofficial].
   - `v->m_parent = &node->m_vertices[idx]` [S1, unofficial].

**Final Vertex Assignment:**
1. For $i$ from 0 to `new_cnt - 1`:
   - `point = solver[i].solve(lower_bound, upper_bound)` [S1, unofficial].
   - `node->m_vertices[i].m_position = point` [S1, unofficial].
   - `node->m_vertices[i].m_normal = glm::normalize(avg_normal[i])` [S1, unofficial].
   - `node->m_vertices[i].m_material = filter[i].select()` [S1, unofficial].
   - `node->m_vertices[i].m_error = solver[i].eval(point)` [S1, unofficial].
   - `node->m_vertices[i].m_qef = solver[i].state()` [S1, unofficial].

**Contour Generation:**
1. Scale $\epsilon$: `scaled_epsilon = epsilon / (S_{global} * S_{global})` [S1, unofficial].
2. Create `Mesh mesh` [S1, unofficial].
3. Call `addVerticesToMesh(&m_root, mesh, scaled_epsilon)` [S1, unofficial].
4. Call `cellProc(&m_root, mesh)` [S1, unofficial].
5. Set global position and scale on mesh [S1, unofficial].
6. Return mesh [S1, unofficial].

### Parameters
- **Grid Size ($N$):** Must be a power of two, $2 \le N \le 1024$. Throws `std::invalid_argument` if not power of two or $< 2$, `std::length_error` if $> 1024$ [S1, unofficial].
- **Epsilon ($\epsilon$):** Scaling factor for contouring, adjusted by global scale [S1, unofficial].

### Edge Cases and Pitfalls
- **LOD Stitching:** Stitching polygons between different levels of detail is difficult and requires handling specific cases with extra triangles to ensure mesh continuity [S2, unofficial].
- **Skirt Recalculation:** Changing the LOD on one chunk can cause tiny changes in the edges of up to six other meshes. To avoid full recalculation, the main mesh is stored separately from the "skirt" (border), which is recalculated only when the chunk or its six relevant neighbors change [S2, unofficial].
- **Frustum Culling:** False positives are unacceptable in frustum culling because they cause meshes to disappear from the player's view, whereas false negatives are acceptable [S2, unofficial].
- **Precision:** The QEF minimization $Q(x,y,z)$ involves summing squared distances to tangent planes, requiring careful handling of numerical precision during plane merging [S2, unofficial].

## Subtopics

### [Octree Mesh Construction](10109-octree-mesh-construction.md)
Generates an isotopic triangular mesh from an implicit function defined over an octree structure. It solves the problem of surface extraction in sparse or hierarchical domains by placing vertices at edge midpoints and connecting them into closed chains per cell, ensuring topological correctness (isotopy) regardless of the octree's irregular subdivision. Unlike uniform grid methods, it adapts resolution locally, though it requires careful handling of nested loops to maintain manifold topology.

## Operating characteristics
- **Time Complexity:** The octree build is $O(N^3)$ for the initial grid and $O(1)$ per node for subdivision. Clustering involves DSU operations, which are nearly linear in the number of vertices per cell. LOD updates are optimized: recalculating only the skirt takes much less time than recalculating the whole mesh, minimizing LOD recalculation overhead [S2, unofficial].
- **Memory Complexity:** $O(N^3)$ for the initial grid storage. The octree structure scales with the number of nodes, which is proportional to the surface area in sparse regions. VBO indexing optimization reduces graphics card memory usage, as the index array does not change when Hermite data changes, allowing only part of the mesh generation to be recalculated for small terrain changes [S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** The method scales well with LOD, as coarser branches minimize the sum of their children's Q functions, allowing efficient rendering of large terrains [S2, unofficial].
- **Strengths:** Efficient LOD management, topological correctness via DSU, material preservation, and optimized updates via skirt separation [S1, S2, unofficial].
- **Limitations:** Complex implementation due to hierarchical clustering and LOD stitching; false positives in frustum culling must be avoided [S2, unofficial].
- **When to use:** Suitable for voxel-based games or applications requiring dynamic LOD and efficient terrain rendering [S2, unofficial].
- **When not to use:** Not suitable for non-hierarchical or uniform grid requirements where simpler dual contouring suffices.

## Background
The method originates from the Isomesh project [S1, unofficial] and the Voxel Level of Detail concept by Parker Lawrence [S2, unofficial]. It addresses the challenge of generating meshes from volumetric data at specified levels of detail using an octree structure, where branches store vertices and act as leaves at coarser levels, minimizing a quadratic function $Q(x,y,z)$ for vertex placement [S2, unofficial]. The technique is used in voxel-based games to ensure smooth performance by reducing triangle counts for distant objects while maintaining detail close to the player [S2, unofficial].

## Sources
[S1] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
[S2] Voxel Level of Detail | Parker Lawrence portfolio, https://legalian.github.io/voxel/2018/03/12/voxel-level-of-detail.html, fetched 2026-10-27 (unofficial)
