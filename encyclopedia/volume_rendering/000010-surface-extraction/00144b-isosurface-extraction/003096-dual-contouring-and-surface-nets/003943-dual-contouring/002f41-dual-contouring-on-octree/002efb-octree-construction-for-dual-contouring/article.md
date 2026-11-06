# Octree Construction for Dual Contouring
Also called: Recursive Octree Construction for Dual Contouring

## Overview
This method constructs the spatial hierarchy required for Dual Contouring by recursively subdividing a volume into an octree, pruning branches that do not intersect the isosurface [S1, unofficial]. It differs from sibling methods like [Mesh Extraction from K-D Tree](4607-mesh-extraction-from-k-d-tree.md) by using a fixed-depth hierarchical grid structure rather than a binary spatial partition, and from [Isomesh Octree Build and Clustering](4984-isomesh-octree-build-and-clustering.md) by focusing on single-vertex-per-leaf extraction logic rather than clustering [G]. It solves the problem of efficiently identifying which voxels contain surface geometry to minimize memory usage and processing time for sparse volumes [G].

## How it works
### Inputs and Outputs
**Inputs:**
- A root `OctreeNode` representing the entire volume bounding box [S1, unofficial].
- A scalar field function `Density_Func(p)` returning the density at position `p` [S1, unofficial].
- Constants: `MATERIAL_SOLID` (density < 0), `MATERIAL_AIR` (density >= 0), `MAX_CROSSINGS` (6), `H` (0.001f), `steps` (8) [S1, unofficial].

**Outputs:**
- A pruned octree where leaf nodes (`Node_Leaf`) contain `OctreeDrawInfo` with vertex position, normal, and QEF data [S1, unofficial]. Internal nodes (`Node_Internal`) contain pointers to children [S1, unofficial]. Nodes with no surface intersection are deleted (`nullptr`) [S1, unofficial].

### Data Structures
**OctreeNode**
- `type` (`enum`): `Node_None`, `Node_Internal`, `Node_Psuedo`, `Node_Leaf` [S1, unofficial].
- `min` (`ivec3`): Minimum corner of the node's bounding box in voxel coordinates [S1, unofficial].
- `size` (`int`): Side length of the cube in voxels [S1, unofficial].
- `children` (`OctreeNode* [8]`): Pointers to 8 child nodes [S1, unofficial].
- `drawInfo` (`OctreeDrawInfo*`): Pointer to drawing data; `nullptr` for internal nodes [S1, unofficial].

**OctreeDrawInfo**
- `index` (`int`): Vertex buffer index [S1, unofficial].
- `corners` (`int`): Bitmask of corner states (bit `i` is 1 if solid, 0 if air) [S1, unofficial].
- `position` (`vec3`): Calculated vertex position [S1, unofficial].
- `averageNormal` (`vec3`): Normalized sum of edge normals [S1, unofficial].
- `qef` (`QEF`): Quadric Error Function solver state [S1, unofficial].

**Constants**
- `CHILD_MIN_OFFSETS` (`ivec3 [8]`): Offsets for child bounding box minimums relative to parent. Indices 0-7 map to `(0,0,0)` through `(1,1,1)` [S1, unofficial].
- `edgevmap` (`int [12][2]`): Maps each of the 12 cube edges to its two corner indices [G].

### Algorithm Steps
1. **Recursive Construction**: Call `ConstructOctreeNodes` on the root [S1, unofficial].
2. **Leaf Evaluation**: If `node->size == 1`, call `ConstructLeaf` [S1, unofficial].
3. **Child Creation**: If `node->size > 1`, create 8 children [S1, unofficial]. For child `i`:
   - `child->size = node->size / 2` [S1, unofficial].
   - `child->min = node->min + (CHILD_MIN_OFFSETS[i] * child->size)` [S1, unofficial].
   - `child->type = Node_Internal` [S1, unofficial].
   - Recursively call `ConstructOctreeNodes(child)` [S1, unofficial].
4. **Pruning**: If all 8 children return `nullptr`, delete the current node and return `nullptr`. Otherwise, return the node [S1, unofficial].
5. **Corner Sampling**: In `ConstructLeaf`, sample `Density_Func` at 8 corners [S1, unofficial]. Corner `i` position is `leaf->min + CHILD_MIN_OFFSETS[i]` [S1, unofficial]. Set bit `i` of `corners` to 1 if density < 0, else 0 [S1, unofficial].
6. **Discard Check**: If `corners == 0` (all air) or `corners == 255` (all solid), delete leaf and return `nullptr` [S1, unofficial].
7. **Edge Intersection**: Iterate 12 edges defined by `edgevmap[i][0]` and `edgevmap[i][1]` (corner indices) [S1, unofficial].
   - Check material states `m1` and `m2` from `corners` [S1, unofficial].
   - If `m1 == m2`, skip edge [S1, unofficial].
   - Otherwise, calculate zero crossing position `p` and normal `n` for the edge [S1, unofficial].
   - Store `p` and `n` in `positions[edgeCount]` and `normals[edgeCount]`. Increment `edgeCount`. Stop if `edgeCount == MAX_CROSSINGS` [S1, unofficial].
8. **Zero Crossing Approximation**: For edge endpoints `p0` and `p1`:
   - Sample 8 points along the segment. `t` ranges from 0 to 1 in steps of `1/8` [S1, unofficial].
   - `p = p0 + (p1 - p0) * t` [S1, unofficial].
   - Find `t` minimizing `abs(Density_Func(p))` [S1, unofficial].
   - Return `p0 + (p1 - p0) * t` [S1, unofficial].
9. **Normal Calculation**: For intersection point `p`:
   - `dx = Density_Func(p + (H,0,0)) - Density_Func(p - (H,0,0))` [S1, unofficial].
   - `dy = Density_Func(p + (0,H,0)) - Density_Func(p - (0,H,0))` [S1, unofficial].
   - `dz = Density_Func(p + (0,0,H)) - Density_Func(p - (0,0,H))` [S1, unofficial].
   - `n = normalize(vec3(dx, dy, dz))` [S1, unofficial].
10. **QEF Solve**: Initialize QEF with `edgeCount`, `normals`, and `positions`. Solve for `position` [S1, unofficial].
11. **Clamping**: If `position` is outside `leaf`'s bounding box (`min` to `min + size`), clamp `position` to `qef.masspoint` (average of input positions) [S1, unofficial].
12. **Finalize**: Sum `normals`, normalize to get `averageNormal`. Set `leaf->type = Node_Leaf`. Assign `drawInfo` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MATERIAL_SOLID = -1
MATERIAL_AIR = 0
MAX_CROSSINGS = 6
H = 0.001
STEPS = 8

# Standard vector/math operations assumed available
# vec3, ivec3, normalize, abs are standard

# Child bounding box offset mapping (0,0,0) to (1,1,1)
CHILD_MIN_OFFSETS = [
    (0,0,0), (0,0,1), (0,1,0), (0,1,1),
    (1,0,0), (1,0,1), (1,1,0), (1,1,1)
]

# Edge to corner index mapping for a cube
edgevmap = [
    (0,1), (1,3), (3,2), (2,0),  # bottom face
    (4,5), (5,7), (7,6), (6,4),  # top face
    (0,4), (1,5), (2,6), (3,7)   # vertical edges
]

class QEF:
    """Standard Quadric Error Function solver [G]."""
    def __init__(self):
        self.normals = []
        self.positions = []
        self.masspoint = (0.0, 0.0, 0.0)
    
    def initialize(self, count, normals, positions):
        self.normals = normals[:count]
        self.positions = positions[:count]
        # Masspoint is average of input positions [S1, unofficial]
        self.masspoint = tuple(sum(p[i] for p in self.positions) / count for i in range(3))
    
    def solve(self):
        # Standard least-squares solution for QEF [G]
        # Returns a vec3 minimizing sum((n_i . (x - p_i))^2)
        return self.masspoint  # Placeholder for actual linear algebra solve [G]

class OctreeNode:
    def __init__(self):
        self.type = 0  # Node_None
        self.min = (0, 0, 0)
        self.size = 0
        self.children = [None] * 8
        self.draw_info = None

class OctreeDrawInfo:
    def __init__(self):
        self.index = -1
        self.corners = 0
        self.position = (0.0, 0.0, 0.0)
        self.average_normal = (0.0, 0.0, 0.0)
        self.qef = QEF()

def calculate_surface_normal(p, density_func):
    dx = density_func((p[0]+H, p[1], p[2])) - density_func((p[0]-H, p[1], p[2]))
    dy = density_func((p[0], p[1]+H, p[2])) - density_func((p[0], p[1]-H, p[2]))
    dz = density_func((p[0], p[1], p[2]+H)) - density_func((p[0], p[1], p[2]-H))
    return normalize((dx, dy, dz))

def approximate_zero_crossing_position(p0, p1, density_func):
    min_val = 100000.0
    best_t = 0.0
    increment = 1.0 / STEPS
    current_t = 0.0
    while current_t <= 1.0:
        p = (p0[0] + (p1[0]-p0[0])*current_t,
             p0[1] + (p1[1]-p0[1])*current_t,
             p0[2] + (p1[2]-p0[2])*current_t)
        density = abs(density_func(p))
        if density < min_val:
            min_val = density
            best_t = current_t
        current_t += increment
    return (p0[0] + (p1[0]-p0[0])*best_t,
            p0[1] + (p1[1]-p0[1])*best_t,
            p0[2] + (p1[2]-p0[2])*best_t)

def construct_octree_nodes(node):
    if node is None:
        return None

    if node.size == 1:
        return construct_leaf(node)

    child_size = node.size // 2
    has_children = False

    for i in range(8):
        child = OctreeNode()
        child.size = child_size
        child.min = tuple(node.min[j] + CHILD_MIN_OFFSETS[i][j] * child_size for j in range(3))
        child.type = 1  # Node_Internal
        
        child_node = construct_octree_nodes(child)
        node.children[i] = child_node
        has_children = has_children or (child_node is not None)

    if not has_children:
        del node
        return None

    return node

def construct_leaf(leaf):
    corners = 0
    for i in range(8):
        corner_pos = tuple(leaf.min[j] + CHILD_MIN_OFFSETS[i][j] for j in range(3))
        density = density_func(corner_pos)
        if density < 0:
            corners |= (1 << i)
    
    if corners == 0 or corners == 255:
        del leaf
        return None

    positions = [(0.0, 0.0, 0.0)] * MAX_CROSSINGS
    normals = [(0.0, 0.0, 0.0)] * MAX_CROSSINGS
    edge_count = 0

    for i in range(12):
        if edge_count >= MAX_CROSSINGS:
            break
            
        c1 = edgevmap[i][0]
        c2 = edgevmap[i][1]
        
        m1 = (corners >> c1) & 1
        m2 = (corners >> c2) & 1
        
        if m1 == m2:
            continue
        
        p1 = tuple(leaf.min[j] + CHILD_MIN_OFFSETS[c1][j] for j in range(3))
        p2 = tuple(leaf.min[j] + CHILD_MIN_OFFSETS[c2][j] for j in range(3))
        
        p = approximate_zero_crossing_position(p1, p2, density_func)
        n = calculate_surface_normal(p, density_func)
        
        positions[edge_count] = p
        normals[edge_count] = n
        edge_count += 1

    draw_info = OctreeDrawInfo()
    draw_info.qef.initialize(edge_count, normals, positions)
    draw_info.position = draw_info.qef.solve()
    
    min_v = tuple(leaf.min[j] for j in range(3))
    max_v = tuple(leaf.min[j] + leaf.size for j in range(3))
    
    if (draw_info.position[0] < min_v[0] or draw_info.position[0] > max_v[0] or
        draw_info.position[1] < min_v[1] or draw_info.position[1] > max_v[1] or
        draw_info.position[2] < min_v[2] or draw_info.position[2] > max_v[2]):
        draw_info.position = draw_info.qef.masspoint

    avg_normal = (0.0, 0.0, 0.0)
    for i in range(edge_count):
        avg_normal = tuple(avg_normal[j] + normals[i][j] for j in range(3))
    draw_info.average_normal = normalize(avg_normal)
    
    draw_info.corners = corners
    leaf.type = 3  # Node_Leaf
    leaf.draw_info = draw_info
    
    return leaf
```

### Edge Cases and Pitfalls
- **Precision**: The zero-crossing approximation uses only 8 samples, which generally finds a position that is not quite the zero crossing, introducing positional error determined by the sample count [S1, unofficial].
- **QEF Solution Outside Box**: The QEF solver may calculate a position outside the leaf's bounding box, requiring clamping to the masspoint (average of input positions) [S1, unofficial].
- **Memory**: The algorithm constructs all internal nodes before pruning leaves, which is acknowledged as wasteful for empty branches [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Simple to implement; hierarchical structure allows for efficient culling of empty space [G].
- **Limitations**: Inefficient memory usage due to pre-construction of internal nodes; fixed sampling rate for zero-crossing may lack accuracy for complex surfaces [S1, unofficial].

## Background
This method is described in an unofficial blog post by Nigel Gildea, which provides a simple implementation of Dual Contouring [S1, unofficial]. It diverges from the reference implementation's class hierarchy by using a single `OctreeNode` class with a type enum [S1, unofficial]. The approach is designed for simplicity rather than optimal performance, explicitly acknowledging the wastefulness of constructing parent nodes before evaluating leaves [S1, unofficial].

## Sources
[S1] Implementing Dual Contouring, http://ngildea.blogspot.com.br/2014/11/implementing-dual-contouring.html, fetched 2026-10-30 (unofficial)
