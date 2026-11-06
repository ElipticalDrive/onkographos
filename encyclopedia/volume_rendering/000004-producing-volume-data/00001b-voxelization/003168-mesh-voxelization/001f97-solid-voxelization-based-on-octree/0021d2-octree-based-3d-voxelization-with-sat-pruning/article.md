# Octree-based 3D Voxelization with SAT Pruning

Also called: Divide-and-Conquer Spatial Partitioning, Octree Divide-and-Conquer Spatial Partitioning, Recursive Octree Voxelizer

## Overview
This method converts arbitrary 3D triangular surface meshes into uniform voxel representations by recursively partitioning space into an octree and using the Separating Axis Theorem (SAT) to prune empty bounding volumes [S1, unofficial]. It builds on [Solid Voxelization based on Octree](2062-solid-voxelization-based-on-octree.md) by replacing ray casting with exact geometric intersection tests for robustness and performance [S1, unofficial].

## How it works
**Inputs:** A triangular mesh (vertices and faces, typically from a Wavefront `.obj` file) and an integer `maxDepth` $\ge 1$ [S1, unofficial].
**Outputs:** A list of voxel cubes (AABBs) or a mesh representation of the voxelized surface [S1, unofficial].

**Data Structures:**
*   `Vec3`: A 3D vector with components `x`, `y`, `z`.
*   `AABB`: Axis-Aligned Bounding Box defined by `min` (Vec3) and `max` (Vec3).
*   `Triangle`: Defined by three `Vec3` vertices `v0`, `v1`, `v2`.
*   `OctreeNode`:
    *   `bounds`: AABB
    *   `children`: Array of 8 pointers to `OctreeNode` (or null if pruned/leaf).
    *   `isLeaf`: Boolean.
    *   `hasGeometry`: Boolean.
    *   `depth`: Integer.
    *   Destructor recursively deletes children [S1, unofficial].
*   `OctreeStats`:
    *   `nodesPerDepth`: Map<int, int>
    *   `prunedPerDepth`: Map<int, int>
    *   `maxDepth`: Integer
    *   `totalVoxels`: Integer [S1, unofficial]

**Algorithm Steps:**

1.  **Root Bounding Box Calculation:**
    Compute the bounding box of the input mesh:
    $$ \mathbf{p}_{\min} = (\min x_i, \min y_i, \min z_i), \quad \mathbf{p}_{\max} = (\max x_i, \max y_i, \max z_i) $$
    where $i$ iterates over all mesh vertices [S1, unofficial].
    Calculate the side length of the largest dimension:
    $$ s_{\text{max}} = \max(\mathbf{p}_{\max}.x - \mathbf{p}_{\min}.x, \mathbf{p}_{\max}.y - \mathbf{p}_{\min}.y, \mathbf{p}_{\max}.z - \mathbf{p}_{\min}.z) $$
    Center the box at the mesh centroid $\mathbf{c} = \frac{\mathbf{p}_{\min} + \mathbf{p}_{\max}}{2}$ [S1, unofficial].
    Expand to a cube with a 1% margin:
    $$ s_{\text{root}} = s_{\text{max}} \times 1.01 $$
    The root AABB is defined by:
    $$ \text{root.min} = \mathbf{c} - \frac{s_{\text{root}}}{2}, \quad \text{root.max} = \mathbf{c} + \frac{s_{\text{root}}}{2} $$

2.  **Recursive Octree Construction (`buildOctree`):**
    Function signature: `buildOctree(node, triangles, maxDepth, currentDepth, stats)` [S1, unofficial].
    
    *   **Pruning Check:** Filter `triangles` to find those intersecting `node.bounds`. Use the 13-axis SAT test (see below). Let this list be `intersecting` [S1, unofficial].
    *   If `intersecting` is empty:
        *   Mark `node` as pruned.
        *   Increment `stats.prunedPerDepth[currentDepth]`.
        *   Return.
    *   If `currentDepth == maxDepth`:
        *   Mark `node.isLeaf = true`.
        *   Mark `node.hasGeometry = true`.
        *   Increment `stats.totalVoxels`.
        *   Return.
    *   **Subdivision:**
        *   Calculate child bounds for $i \in [0, 7]$ [S1, unofficial].
        *   Let $c = \text{node.bounds.center()}$ [S1, unofficial].
        *   For each axis $a \in \{x, y, z\}$:
            *   Bit $b$ corresponds to axis index (0 for x, 1 for y, 2 for z) [S1, unofficial].
            *   If $(i \gg b) \& 1$:
                *   $\text{child.min}.a = c.a$ [S1, unofficial].
            *   Else:
                *   $\text{child.min}.a = \text{node.bounds.min}.a$ [S1, unofficial].
            *   $\text{child.max}.a = \text{node.bounds.max}.a - (\text{child.min}.a - \text{node.bounds.min}.a)$ (or symmetrically derived from center) [S1, unofficial].
        *   For each child $i$:
            *   Test if any triangle in `intersecting` intersects `child.bounds` using SAT [S1, unofficial].
            *   If no intersection:
                *   Increment `stats.nodesPerDepth[currentDepth + 1]` and `stats.prunedPerDepth[currentDepth + 1]` [S1, unofficial].
                *   Continue to next child.
            *   If intersection exists:
                *   Create `newNode`.
                *   Set `newNode.bounds`, `newNode.depth = currentDepth + 1` [S1, unofficial].
                *   `node.children[i] = newNode`.
                *   Recursively call `buildOctree(newNode, intersecting, maxDepth, currentDepth + 1, stats)` [S1, unofficial].

3.  **Voxel Collection:**
    Traverse the octree. If a node is a leaf and `hasGeometry` is true, add its `bounds` to the output list. Otherwise, recursively collect from all 8 children [S1, unofficial].

4.  **Mesh Generation:**
    For each collected AABB (min, max):
    *   Generate 8 vertices at corners $(x_0, y_0, z_0)$ to $(x_1, y_1, z_1)$ [S1, unofficial].
    *   Generate 12 triangular faces (2 per side) using specific vertex indices to ensure outward-facing normals [S1, unofficial].

**SAT Collision Test:**
To determine if a triangle intersects an AABB, test overlap along 13 separating axes [S1, unofficial]:
1.  **3 Box Face Normals:** $\mathbf{u}_0 = (1, 0, 0), \mathbf{u}_1 = (0, 1, 0), \mathbf{u}_2 = (0, 0, 1)$ [S1, unofficial].
2.  **1 Triangle Face Normal:** $\mathbf{n} = (\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0)$ [S1, unofficial].
3.  **9 Edge Cross Products:** Cross products of each triangle edge with each box face normal [S1, unofficial].
    *   Triangle edges: $\mathbf{e}_0 = \mathbf{v}_1 - \mathbf{v}_0$, $\mathbf{e}_1 = \mathbf{v}_2 - \mathbf{v}_1$, $\mathbf{e}_2 = \mathbf{v}_0 - \mathbf{v}_2$ [S1, unofficial].
    *   Axes: $\mathbf{e}_k \times \mathbf{u}_j$ for $k \in \{0,1,2\}, j \in \{0,1,2\}$ [S1, unofficial].
    
    For each axis $\mathbf{L}$, project both the triangle vertices and the AABB corners onto $\mathbf{L}$. If the intervals do not overlap, the triangle and box do not intersect [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
    def __sub__(self, other):
        return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __add__(self, other):
        return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __mul__(self, scalar):
        return Vec3(self.x * scalar, self.y * scalar, self.z * scalar)
    def dot(self, other):
        return self.x * other.x + self.y * other.y + self.z * other.z
    def cross(self, other):
        return Vec3(
            self.y * other.z - self.z * other.y,
            self.z * other.x - self.x * other.z,
            self.x * other.y - self.y * other.x
        )

class AABB:
    def __init__(self, min, max):
        self.min = min
        self.max = max
    def center(self):
        return Vec3(
            (self.min.x + self.max.x) / 2.0,
            (self.min.y + self.max.y) / 2.0,
            (self.min.z + self.max.z) / 2.0
        )

class Triangle:
    def __init__(self, v0, v1, v2):
        self.v0 = v0
        self.v1 = v1
        self.v2 = v2

class OctreeNode:
    def __init__(self, bounds, depth):
        self.bounds = bounds
        self.depth = depth
        self.children = [None] * 8
        self.isLeaf = False
        self.hasGeometry = False

def project_interval(vertices, axis):
    min_p = float('inf')
    max_p = float('-inf')
    for v in vertices:
        p = v.dot(axis)
        if p < min_p: min_p = p
        if p > max_p: max_p = p
    return min_p, max_p

def sat_test(triangle, aabb):
    # 1. Box Face Normals
    box_axes = [Vec3(1,0,0), Vec3(0,1,0), Vec3(0,0,1)]
    
    # 2. Triangle Normal: n = (v1 - v0) x (v2 - v0)
    e0 = triangle.v1 - triangle.v0
    e1 = triangle.v2 - triangle.v0
    tri_normal = e0.cross(e1)
    
    # 3. Edge Cross Products: e_k x u_j
    tri_edges = [e0, triangle.v2 - triangle.v1, triangle.v0 - triangle.v2]
    axes = box_axes + [tri_normal] + [e.cross(b) for e in tri_edges for b in box_axes]
    
    # Normalize axes for projection
    normalized_axes = []
    for ax in axes:
        l = math.sqrt(ax.x**2 + ax.y**2 + ax.z**2)
        if l > 1e-9:
            normalized_axes.append(Vec3(ax.x/l, ax.y/l, ax.z/l))
        else:
            normalized_axes.append(ax)
            
    tri_verts = [triangle.v0, triangle.v1, triangle.v2]
    
    # AABB corners
    corners = [
        aabb.min,
        Vec3(aabb.max.x, aabb.min.y, aabb.min.z),
        Vec3(aabb.min.x, aabb.max.y, aabb.min.z),
        Vec3(aabb.max.x, aabb.max.y, aabb.min.z),
        Vec3(aabb.min.x, aabb.min.y, aabb.max.z),
        Vec3(aabb.max.x, aabb.min.y, aabb.max.z),
        Vec3(aabb.min.x, aabb.max.y, aabb.max.z),
        aabb.max
    ]
    
    for axis in normalized_axes:
        t_min, t_max = project_interval(tri_verts, axis)
        a_min, a_max = project_interval(corners, axis)
        
        if t_max < a_min or a_max < t_min:
            return False # Separating axis found
    return True # No separating axis found, intersection

def build_octree(node, triangles, max_depth, current_depth, stats):
    # Filter triangles
    intersecting = [t for t in triangles if sat_test(t, node.bounds)]
    
    if not intersecting:
        # Pruned
        stats['prunedPerDepth'][current_depth] = stats['prunedPerDepth'].get(current_depth, 0) + 1
        return
    
    if current_depth == max_depth:
        node.isLeaf = True
        node.hasGeometry = True
        stats['totalVoxels'] = stats.get('totalVoxels', 0) + 1
        return

    center = node.bounds.center()
    
    for i in range(8):
        # Calculate child bounds using bitwise operations on index
        # Bit 0 -> x, Bit 1 -> y, Bit 2 -> z
        child_min = Vec3(
            center.x if (i & 1) else node.bounds.min.x,
            center.y if (i & 2) else node.bounds.min.y,
            center.z if (i & 4) else node.bounds.min.z
        )
        child_max = Vec3(
            center.x if not (i & 1) else node.bounds.max.x,
            center.y if not (i & 2) else node.bounds.max.y,
            center.z if not (i & 4) else node.bounds.max.z
        )
        
        child_bounds = AABB(child_min, child_max)
        
        # Check intersection with any triangle in intersecting list
        has_geom = False
        for t in intersecting:
            if sat_test(t, child_bounds):
                has_geom = True
                break
        
        if has_geom:
            child_node = OctreeNode(child_bounds, current_depth + 1)
            node.children[i] = child_node
            build_octree(child_node, intersecting, max_depth, current_depth + 1, stats)
        else:
            stats['nodesPerDepth'][current_depth + 1] = stats['nodesPerDepth'].get(current_depth + 1, 0) + 1
            stats['prunedPerDepth'][current_depth + 1] = stats['prunedPerDepth'].get(current_depth + 1, 0) + 1

def collect_voxels(node, voxels):
    if node.isLeaf and node.hasGeometry:
        voxels.append(node.bounds)
        return
    for child in node.children:
        if child is not None:
            collect_voxels(child, voxels)

def generate_voxel_mesh(voxels):
    mesh_vertices = []
    mesh_faces = []
    
    for v in voxels:
        x0, y0, z0 = v.min.x, v.min.y, v.min.z
        x1, y1, z1 = v.max.x, v.max.y, v.max.z
        
        # 8 vertices
        verts = [
            Vec3(x0, y0, z0), Vec3(x1, y0, z0), Vec3(x1, y1, z0), Vec3(x0, y1, z0),
            Vec3(x0, y0, z1), Vec3(x1, y0, z1), Vec3(x1, y1, z1), Vec3(x0, y1, z1)
        ]
        base_idx = len(mesh_vertices)
        mesh_vertices.extend(verts)
        
        # 12 faces (2 per side) with outward-facing normals
        faces = [
            [0, 1, 2], [0, 2, 3], # Front (z0)
            [4, 6, 5], [4, 7, 6], # Back (z1)
            [0, 4, 7], [0, 7, 3], # Left (x0)
            [1, 5, 6], [1, 6, 2], # Right (x1)
            [3, 7, 6], [3, 6, 2], # Top (y1)
            [0, 5, 4], [0, 1, 5]  # Bottom (y0)
        ]
        
        for f in faces:
            mesh_faces.append([base_idx + f[0], base_idx + f[1], base_idx + f[2]])
            
    return mesh_vertices, mesh_faces

def voxelize(mesh_vertices, mesh_faces, max_depth):
    # 1. Compute Root AABB
    p_min = Vec3(float('inf'), float('inf'), float('inf'))
    p_max = Vec3(float('-inf'), float('-inf'), float('-inf'))
    
    for v in mesh_vertices:
        if v.x < p_min.x: p_min.x = v.x
        if v.y < p_min.y: p_min.y = v.y
        if v.z < p_min.z: p_min.z = v.z
        if v.x > p_max.x: p_max.x = v.x
        if v.y > p_max.y: p_max.y = v.y
        if v.z > p_max.z: p_max.z = v.z
        
    s_max = max(p_max.x - p_min.x, p_max.y - p_min.y, p_max.z - p_min.z)
    s_root = s_max * 1.01
    center = Vec3((p_min.x + p_max.x)/2, (p_min.y + p_max.y)/2, (p_min.z + p_max.z)/2)
    
    root_min = Vec3(center.x - s_root/2, center.y - s_root/2, center.z - s_root/2)
    root_max = Vec3(center.x + s_root/2, center.y + s_root/2, center.z + s_root/2)
    
    root_bounds = AABB(root_min, root_max)
    root_node = OctreeNode(root_bounds, 0)
    
    # Create triangles
    triangles = []
    for i in range(0, len(mesh_faces), 3):
        v0 = mesh_vertices[mesh_faces[i]]
        v1 = mesh_vertices[mesh_faces[i+1]]
        v2 = mesh_vertices[mesh_faces[i+2]]
        triangles.append(Triangle(v0, v1, v2))
        
    # 2. Build Octree
    stats = {'nodesPerDepth': {}, 'prunedPerDepth': {}, 'totalVoxels': 0}
    build_octree(root_node, triangles, max_depth, 0, stats)
    
    # 3. Collect Voxels
    voxels = []
    collect_voxels(root_node, voxels)
    
    # 4. Generate Mesh
    return generate_voxel_mesh(voxels)
```

## Operating characteristics
*   **Time Complexity:** Depends on the spatial distribution of geometry and `maxDepth`. In the worst case (dense geometry), it is proportional to the number of leaf nodes. Pruning significantly reduces this for sparse models [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of octree nodes created. Pruned nodes do not allocate child pointers [S1, unofficial].
*   **Performance:** On the 'cow' model (4583 vertices, 5804 faces) at depth 5, the method produced 1303 voxels, 10424 vertices, and 15636 faces, executing in 24.32 ms [S1, unofficial].
*   **Strengths:** Exact geometric intersection avoids rasterization gaps. Recursive structure adapts to geometry density [S1, unofficial].
*   **Limitations:** Requires `maxDepth` tuning for resolution. SAT test is computationally expensive per triangle per node compared to simple bounding box overlap [S1, unofficial].

## Background
This method is described in the project "3D-Object-Surface-Voxelizer" by Joji0 [S1, unofficial]. It utilizes a C++17 engine with SFML for visualization. The approach is categorized as "Divide-and-Conquer Spatial Partitioning" [S1, unofficial].

## Sources
[S1] Joji0/3D-Object-Surface-Voxelizer, https://github.com/Joji0/3D-Object-Surface-Voxelizer, fetched 2026-10-27 (unofficial)
