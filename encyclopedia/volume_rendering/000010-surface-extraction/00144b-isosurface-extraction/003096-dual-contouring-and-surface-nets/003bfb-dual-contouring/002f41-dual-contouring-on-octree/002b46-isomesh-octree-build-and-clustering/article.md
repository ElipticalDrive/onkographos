# Isomesh Octree Build and Clustering

Also called: Isomesh

## Overview
Isomesh is a hierarchical mesh generation method that constructs an octree over a uniform grid to extract a surface mesh while preserving material properties and minimizing reconstruction error [S1, unofficial]. It differs from standard dual contouring by recursively clustering vertices at every node level (leaves, edges, faces, and cells) using Disjoint Set Union (DSU) and Quadratic Error Function (QEF) solvers, ensuring topological consistency across the hierarchy [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- A scalar field function $f(\mathbf{p})$ and its gradient $\nabla f(\mathbf{p})$.
- A grid size $N$ (must be a power of two, $2 \le N \le 1024$) [S1, unofficial].
- A global scale factor $S_{global}$ [S1, unofficial].

**Outputs:**
- A `Mesh` object containing vertices, normals, materials, and triangle indices.

### Data Structures

**UniformGrid:**
- `m_size`: Integer grid dimension $N$.
- `m_halfSize`: $N / 2$.
- `m_gridStep`: Physical size of one voxel.
- `m_mat`: Raw array of `Material` of size $(N+1)^3$.
- `values`: Temporary array of floats of size $(N+1)^3$ storing scalar field values.
- Indexing is Z-major: `idx = ((y + m_halfSize) * (N + 1) + (x + m_halfSize)) * (N + 1) + (z + m_halfSize)` [S1, unofficial].

**Octree Node:**
- `m_vertices`: List of vertex structures.
- `m_isLeaf`: Boolean flag.
- `m_child[8]`: Pointers to child nodes (if not leaf).
- `m_corners`: Array of 8 `Material` values for the cell corners.
- `m_vertexMask`: 8-bit integer, bit $i$ set if `m_corners[i] != Material::Empty` [S1, unofficial].

**Vertex Structure:**
- `m_position`: `glm::vec3` position.
- `m_normal`: `glm::vec3` normal.
- `m_material`: `Material` enum.
- `m_error`: Float QEF error value.
- `m_qef`: QEF state object.
- `m_surfaceIdx`: Index into the storage structure (used during clustering).
- `m_parent`: Pointer to the parent node's vertex (used during upward clustering) [S1, unofficial].

**Edge Storage:**
- Three separate storage structures (`m_edgeX`, `m_edgeY`, `m_edgeZ`) for edges aligned with X, Y, and Z axes respectively.
- Each stores `surfacePoint()`, `surfaceNormal()`, `solidEndpointMaterial()`, `isLesserEndpointSolid()`, and geometric coordinates [S1, unofficial].

### Constants and Tables
- `kCellCornerOffset[8]`: Offsets for the 8 corners of a cube.
- `kCellEdgeDirection[12]`: Direction vectors for the 12 edges.
- `kCellEdgeEndpoint[12][2]`: Indices of the two endpoints for each edge.
- `kMdcEdgeSetIndex[256][12]`: Lookup table mapping a vertex mask and edge index to a solver index (or -1 if the edge is not intersected) [S1, unofficial].
- `kEdgeProcChildTable[3][4]` and `kFaceProcChildTable[3][4]`: Tables defining which children of a node are involved in edge/face processing for dimensions D=0,1,2 [S1, unofficial].
- `kBadIndex`: Sentinel value for invalid indices [S1, unofficial].

### Algorithm Steps

#### 1. Grid Filling
Populate the `UniformGrid` with scalar field values and materials.
1. Iterate $y, x, z$ from $-m\_halfSize$ to $m\_halfSize$.
2. Compute global position $\mathbf{p} = \text{localToGlobal}(x, y, z)$.
3. Compute $v = f(\mathbf{p})$.
4. Store $v$ in `values[idx]`.
5. If $v > 0$, set `m_mat[idx] = Material::Empty`.
6. Else, set `m_mat[idx] = f.material(\mathbf{p}, v)` [S1, unofficial].

#### 2. Edge Intersection Detection
Detect isosurface intersections on grid edges and store them in edge storage structures.

**X-Axis Edges:**
1. Iterate $y$ from $-m\_halfSize$ to $m\_halfSize$, $z$ from $-m\_halfSize$ to $m\_halfSize$, $x$ from $-m\_halfSize$ to $m\_halfSize - 1$.
2. Get indices `idx1` for $(x, y, z)$ and `idx2` for $(x+1, y, z)$.
3. Compute signs $s_1 = (values[idx1] \le 0.0)$ and $s_2 = (values[idx2] \le 0.0)$.
4. If $s_1 \neq s_2$:
   - Compute global position $\mathbf{p}_0 = \text{localToGlobal}(x, y, z)$.
   - Solve for intersection $p_x$ along the X-axis using linear interpolation:
     $$ p_x = x_0 + \frac{0 - v_1}{v_2 - v_1} (x_1 - x_0) $$
     where $x_0, x_1$ are global coordinates of the edge endpoints, and $v_1, v_2$ are the scalar values.
   - Compute intersection point $\mathbf{p}_{int} = (p_x, \mathbf{p}_0.y, \mathbf{p}_0.z)$.
   - Compute gradient $\mathbf{g} = \nabla f(\mathbf{p}_{int})$.
   - Compute offset $o = (p_x - x_0) / m\_gridStep$.
   - Determine material from the solid endpoint.
   - Call `m_edgeX.addEdge(x, y, z, \mathbf{g}, o, 0, s_1, mat)` [S1, unofficial].

**Y and Z-Axis Edges:**
- Repeat similar logic for Y and Z axes, iterating appropriate ranges ($x < m\_halfSize$ for X, $y < m\_halfSize$ for Y, $z < m\_halfSize$ for Z) and using `findAlongY`/`findAlongZ` and respective edge storages [S1, unofficial].

#### 3. Octree Build Phase
Recursively build the octree and process leaves.

**`buildNode(node, min_corner, size)`:**
1. If `size == 1`:
   - Call `buildLeaf(node, min_corner)` [S1, unofficial].
2. Else:
   - Call `node->subdivide()`.
   - Compute `child_size = size / 2`.
   - For $i$ from 0 to 7:
     - Compute `child_min_corner = min_corner + child_size * kCellCornerOffset[i]`.
     - Recursively call `buildNode(node->child(i), child_min_corner, child_size)` [S1, unofficial].

**`buildLeaf(node, min_corner)`:**
1. Get `corners = args.grid.materialsOfCell(args.grid.pointToIndex(min_corner))`.
2. Compute `vertex_mask`: set bit $i$ if `corners[i] != Material::Empty`.
3. Call `node->setCorners(corners)` and `node->setVertexMask(vertex_mask)`.
4. Initialize `solver[4]`, `filter[4]`, `avg_normal[4]`.
5. Initialize `corner_used[8]` to false.
6. **Leaf Edge Processing:**
   - For $i$ from 0 to 11 (12 edges):
     - Get `idx = kMdcEdgeSetIndex[vertex_mask][i]`.
     - If `idx < 0`, continue.
     - Get `storage` from `storages[kCellEdgeDirection[i]]`.
     - Calculate `edge_pos = min_corner + kCellCornerOffset[kCellEdgeEndpoint[i][0]]`.
     - Find edge iterator `iter = storage.findEdge(edge_pos.x, edge_pos.y, edge_pos.z)`.
     - Get `point = iter->surfacePoint()` and `normal = iter->surfaceNormal()`.
     - Call `solver[idx].addPlane(point, normal)`.
     - Add `normal` to `avg_normal[idx]`.
     - Determine `mat = iter->solidEndpointMaterial()`.
     - Determine `corner_idx`: if `iter->isLesserEndpointSolid()` is true, `corner_idx = kCellEdgeEndpoint[i][0]`, else `corner_idx = kCellEdgeEndpoint[i][1]`.
     - If `!corner_used[corner_idx]`:
       - Call `filter[idx].add(mat)`.
       - Set `corner_used[corner_idx] = true`.
7. **Leaf Vertex Solving:**
   - Determine `surface_cnt` (max `idx + 1` from the loop).
   - Resize `node->m_vertices` to `surface_cnt`.
   - Define `lower_bound = min_corner` and `upper_bound = min_corner + size`.
   - For $i$ from 0 to `surface_cnt - 1`:
     - Solve `point = solver[i].solve(lower_bound, upper_bound)`.
     - Set `node->m_vertices[i].m_position = point`.
     - Set `m_normal = glm::normalize(avg_normal[i])`.
     - Set `m_material = filter[i].select()`.
     - Set `m_error = solver[i].eval(point)`.
     - Set `m_qef = solver[i].state()` [S1, unofficial].

#### 4. Clustering Phase
Merge vertices from children into parents to ensure topological consistency.

**`clusterCell(node, min_corner, size)`:**
1. If `node->isLeaf()`, return.
2. Compute `sub_size = size / 2`.
3. For $i$ from 0 to 7:
   - Compute `sub_pos = min_corner + sub_size * kCellCornerOffset[i]`.
   - Recursively call `clusterCell(node->child(i), sub_pos, sub_size)`.
4. Initialize `DSU dsu` and `VertexArray vertices`.
5. Call `callSubClusterFace(0)`, `callSubClusterFace(1)`, `callSubClusterFace(2)`.
6. Call `callSubClusterEdge(0)`, `callSubClusterEdge(1)`, `callSubClusterEdge(2)` [S1, unofficial].

**`callSubClusterFace(D)` and `callSubClusterEdge(D)`:**
1. Iterate children using `kFaceProcChildTable[D]` or `kEdgeProcChildTable[D]`.
2. For each child node `n`:
   - If `n` is subdivided, get specific child `sub[i] = n->child(k...[i][1])` and set `all_leaves = false`.
   - Else `sub[i] = n`.
3. If `all_leaves` is true:
   - Call `clusterFaceLeaves<D>(nodes, dsu, vertices)` or `clusterEdgeLeaves<D>(nodes, dsu, vertices)`.
   - Return.
4. Otherwise, recursively call `callSubClusterFace(D)` or `callSubClusterEdge(D)` on the children [S1, unofficial].

**Vertex Merging in Cell (Post-Clustering):**
1. Map DSU sets to new indices.
2. Initialize `set_idx` vector of size `dsu.size()` with `kBadIndex`.
3. For $i$ from 0 to `dsu.size()-1`:
   - Get `leader = dsu.getSetLeader(i)`.
   - If `set_idx[leader] == kBadIndex`:
     - Set `set_idx[leader] = new_cnt`.
     - Increment `new_cnt`.
   - Set `set_idx[i] = set_idx[leader]`.
4. Resize `node->m_vertices` to `new_cnt` [S1, unofficial].

**Solving Clustered Vertices:**
1. Initialize `solver` vector of size `new_cnt` with copies of `orig_solver`.
2. Initialize `filter` and `avg_normal` vectors of size `new_cnt`.
3. For each vertex `v` in `vertices`:
   - Get `idx = set_idx[v->m_surfaceIdx]`.
   - Set `v->m_surfaceIdx = kBadIndex`.
   - Call `solver[idx].merge(v->m_qef)`.
   - Add `v->m_normal` to `avg_normal[idx]`.
   - Call `filter[idx].add(v->m_material)`.
   - Set `v->m_parent = &node->m_vertices[idx]` [S1, unofficial].

**Final Vertex Assignment:**
1. For $i$ from 0 to `new_cnt - 1`:
   - Solve `point = solver[i].solve(lower_bound, upper_bound)`.
   - Set `node->m_vertices[i].m_position = point`.
   - Set `m_normal = glm::normalize(avg_normal[i])`.
   - Set `m_material = filter[i].select()`.
   - Set `m_error = solver[i].eval(point)`.
   - Set `m_qef = solver[i].state()` [S1, unofficial].

#### 5. Contour Generation
1. Scale `epsilon` by $1 / (S_{global} \times S_{global})$ to get `scaled_epsilon`.
2. Create `Mesh mesh`.
3. Call `addVerticesToMesh(&m_root, mesh, scaled_epsilon)`.
4. Call `cellProc(&m_root, mesh)` to generate triangles.
5. Set global position and scale on mesh.
6. Return mesh [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The octree build is $O(N^3)$ where $N$ is the grid size. Clustering involves traversing the tree and merging vertices, which is generally efficient due to the DSU structure, but depends on the number of intersected edges/faces.
- **Memory Complexity:** Dominated by the uniform grid storage $(N+1)^3$ and the octree nodes. The edge storage structures also consume memory proportional to the number of intersected edges.
- **Strengths:** Preserves material properties at vertices; minimizes reconstruction error via QEF; handles complex topology through hierarchical clustering.
- **Limitations:** Grid size is constrained to powers of two up to 1024; requires significant memory for large grids; implementation complexity is high due to the hierarchical clustering logic [S1, unofficial].

## Background
Isomesh is an open-source project by Svenny, hosted on GitHub [S1, unofficial]. It implements a dual contouring approach on an octree structure, focusing on material-aware mesh generation. The method was developed as an alternative to traditional marching cubes or standard dual contouring, aiming to provide better material handling and topological consistency in volumetric data.

## Sources
[S1] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
