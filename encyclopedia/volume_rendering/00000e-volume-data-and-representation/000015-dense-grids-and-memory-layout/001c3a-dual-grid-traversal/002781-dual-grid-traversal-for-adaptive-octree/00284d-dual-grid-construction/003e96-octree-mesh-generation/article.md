# Octree Mesh Generation
Also called: MeshFromOctree
## Overview
Octree Mesh Generation constructs a manifold triangle mesh from an adaptive octree volume representation by traversing the dual grid structure. It solves the problem of converting hierarchical spatial subdivision into a renderable surface without explicit marching cubes lookup tables. It differs from standard dual contouring by operating directly on octree node connectivity rather than scalar field gradients, ensuring watertightness at adaptive boundaries through recursive face and edge processing.
## How it works
**Inputs and Outputs:**
- Input: Root octree node, vertex buffer (stores 3D positions), index buffer (stores triangle indices).
- Output: Populated index buffer defining the mesh topology.

**Data Structures:**
- `Octree`: Node containing `m_leaf` (boolean), children pointers (up to 8), and spatial bounds.
- `IndexBuffer`: Dynamic array of integers storing triangle vertex indices.
- `LeafOrChild(n, i)`: Helper returning the $i$-th child of node $n$ if $n$ is not a leaf, otherwise returning $n$ itself.

**Algorithm:**
The generation proceeds in two phases: vertex index generation and cell processing.

1. **Generate Vertex Indices:**
   Call `GenerateVertexIndices` with the root node and the vertex buffer. This step typically assigns unique identifiers to dual grid vertices associated with octree nodes. [S1, unofficial]

2. **Cell Processing:**
   Call `CellProc` with the index buffer. This initiates the recursive traversal. [S1, unofficial]

**Recursive Procedures:**
The core logic relies on three recursive functions that process faces and edges of the octree cells.

**FaceProcX (Processing X-aligned faces):**
For a pair of adjacent octree nodes $n_0$ and $n_1$ along the X-axis:
1. If either node is null or both are leaves, return.
2. Define face pairs: $\{(1,0), (3,2), (5,4), (7,6)\}$.
3. Define XY-edge pairs: $\{(1,0,3,2), (5,4,7,6)\}$.
4. Define XZ-edge pairs: $\{(3,2,7,6), (1,0,5,4)\}$.
5. For each face pair $(a,b)$, recursively call `FaceProcX(LeafOrChild(n0, a), LeafOrChild(n1, b), indexBuffer)`.
6. For each XY-edge tuple $(a,b,c,d)$, call `EdgeProcXY(LeafOrChild(n0, a), LeafOrChild(n1, b), LeafOrChild(n0, c), LeafOrChild(n1, d), indexBuffer)`.
7. For each XZ-edge tuple $(a,b,c,d)$, call `EdgeProcXZ(LeafOrChild(n0, a), LeafOrChild(n1, b), LeafOrChild(n0, c), LeafOrChild(n1, d), indexBuffer)`. [S1, unofficial]

**EdgeProcYZ (Processing YZ-aligned edges):**
For four octree nodes $n_0, n_1, n_2, n_3$ forming a YZ-edge configuration:
1. Call `EdgeProcYZ(LeafOrChild(n0, 7), LeafOrChild(n1, 5), LeafOrChild(n2, 3), LeafOrChild(n3, 1), indexBuffer)`. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(root_node, vertex_buffer, index_buffer):
    # Phase 1: Generate vertex indices
    generate_vertex_indices(root_node, vertex_buffer)
    
    # Phase 2: Process cells
    cell_proc(index_buffer)

def face_proc_x(n0, n1, index_buffer):
    if n0 is None or n1 is None:
        return
    if n0.is_leaf and n1.is_leaf:
        return

    face_pairs = [(1, 0), (3, 2), (5, 4), (7, 6)]
    edge_xy_pairs = [(1, 0, 3, 2), (5, 4, 7, 6)]
    edge_xz_pairs = [(3, 2, 7, 6), (1, 0, 5, 4)]

    for a, b in face_pairs:
        child_a = leaf_or_child(n0, a)
        child_b = leaf_or_child(n1, b)
        face_proc_x(child_a, child_b, index_buffer)

    for a, b, c, d in edge_xy_pairs:
        n0_a = leaf_or_child(n0, a)
        n1_b = leaf_or_child(n1, b)
        n0_c = leaf_or_child(n0, c)
        n1_d = leaf_or_child(n1, d)
        edge_proc_xy(n0_a, n1_b, n0_c, n1_d, index_buffer)

    for a, b, c, d in edge_xz_pairs:
        n0_a = leaf_or_child(n0, a)
        n1_b = leaf_or_child(n1, b)
        n0_c = leaf_or_child(n0, c)
        n1_d = leaf_or_child(n1, d)
        edge_proc_xz(n0_a, n1_b, n0_c, n1_d, index_buffer)

def leaf_or_child(node, index):
    if node.is_leaf:
        return node
    return node.children[index]

def cell_proc(index_buffer):
    # Implementation details for initial cell traversal not fully specified in source [S1, unofficial]
    # Typically iterates over root children or calls face_proc_x on root neighbors
    pass
```

**Parameters:**
- Not given in the sources.

**Edge Cases:**
- Null nodes: Handled by early return in `FaceProcX`.
- Leaf nodes: Internal faces between two leaves are skipped to avoid duplicate or invalid geometry. [S1, unofficial]
- Adaptive boundaries: The `LeafOrChild` helper ensures that traversal continues into child nodes when a boundary exists, maintaining connectivity.

## Subtopics

### [Adaptive Octree Voxel Grid](18773-adaptive-octree-voxel-grid.md)
A dynamic volume representation that refines/coarsens based on simulation activity. Choose when memory optimization for sparse/dynamic scenes is required.

## Operating characteristics
- Time Complexity: Linear with respect to the number of octree nodes involved in the surface boundary.
- Memory Complexity: Proportional to the number of triangles generated.
- Strengths: Watertight mesh generation at adaptive boundaries without lookup tables.
- Limitations: Requires complex recursive traversal logic.
- When to use: Adaptive octree volume rendering where manifold mesh output is needed.
- When not to use: Static uniform grids where simpler methods suffice.
## Background
Originated in the context of procedural generation and adaptive volume rendering. The specific implementation details are documented in the `Tsarpf/procgen` repository. [S1, unofficial]
## Sources
[S1] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
