# Octree-based Dual Contouring Edge/Face Processing

Also called: DualContouringSample Octree, Tao Ju Octree Implementation

## Overview
This method is a recursive procedure that processes the edges and faces of an adaptive octree to generate triangle indices for a dual contouring mesh [S1, unofficial]. It handles both leaf and pseudo-leaf nodes by descending into children when necessary, ensuring that geometry is generated only at the boundaries between solid and air regions [S1, unofficial]. It builds on [Edge Contouring for Dual Contouring](3711-edge-contouring-for-dual-contouring.md) by providing the specific octree-aware traversal and edge processing logic required for adaptive volumetric data [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: An adaptive octree where each node contains material flags for its 8 corners (`m_corners`), a size (`m_size`), and optional children (`m_children`) [S1, unofficial]. A scalar field sampler (`Sampler::Sample` or `Sampler::SampleCache`) is required to evaluate density at arbitrary positions [S1, unofficial].
- **Output**: An `IndexBuffer` containing triangle indices (triplets of vertex indices) [S1, unofficial].

### Data Structures
- **Octree Node**:
  - `m_leaf` (bool): True if the node is a leaf [S1, unofficial].
  - `m_children` (pointer to `OctreeChildren`): Contains a `field` (uint8_t bitmask of existing children) and an array of 8 child pointers [S1, unofficial].
  - `m_corners` (int): Bit-packed material flags for the 8 corners of the node. Bit `i` corresponds to corner `i` [S1, unofficial].
  - `m_size` (int): The size of the node in world units (or grid units) [S1, unofficial].
  - `m_index` (int): The vertex index assigned to this node's dual vertex [S1, unofficial].
- **OctreeChildren**:
  - `field` (uint8_t): Bitmask indicating which of the 8 children exist [S1, unofficial].
  - `children` (array of 8 pointers): Pointers to child `Octree` nodes [S1, unofficial].
- **Constants**:
  - `MATERIAL_AIR`: 0 [S1, unofficial].
  - `MATERIAL_SOLID`: 1 [S1, unofficial].
  - `QEF_ERROR`: 1e-6f (used in QEF solving, not directly in edge processing but part of the leaf construction context) [S1, unofficial].
  - `minError`: 0.01f (early exit threshold for zero-crossing approximation) [S1, unofficial].

### Helper Functions
1. **LeafOrChild(node, idx)**:
   - If `node` is a leaf, return `node` [S1, unofficial].
   - If `node` has children, return `node->children[idx]` [S1, unofficial].
   - Return `nullptr` if the child does not exist [S1, unofficial].

2. **ApproximateZeroCrossingPosition(p0, p1)**:
   - Finds the position on edge `p0-p1` where the scalar field is near zero [S1, unofficial].
   - Uses a binary search approach with 8 steps [S1, unofficial].
   - Start `currentT = 0.5`, `increment = 0.5` [S1, unofficial].
   - Loop 8 times:
     - `p = p0 + (p1 - p0) * currentT` [S1, unofficial].
     - `density = abs(Sample(p))` [S1, unofficial].
     - If `density < minError` (0.01f), return `p` [S1, unofficial].
     - If `density > 0`, `currentT -= increment` [S1, unofficial].
     - Else, `currentT += increment` [S1, unofficial].
     - `increment *= 0.5` [S1, unofficial].
   - Return `p0 + (p1 - p0) * currentT` [S1, unofficial].

### Main Traversal
1. **CellProc(node)**:
   - If `node` is a leaf, return [S1, unofficial].
   - Call `CellChildProc` for each of the 8 children [S1, unofficial].

2. **CellChildProc(node)**:
   - Process 4 X-faces, 4 Y-faces, 4 Z-faces, and 6 edges (XY, XZ, YZ) using predefined neighbor indices [S1, unofficial].
   - Recursively call `CellProc` on all 8 children [S1, unofficial].

### Face Processing
- **FaceProcX/Y/Z(n0, n1, n2, n3, indexBuffer)**:
  - Check if both adjacent nodes are leaves [S1, unofficial].
  - If so, return (no geometry generated for internal faces between two solid/air leaves in this specific implementation logic) [S1, unofficial].
  - If not, recursively process sub-faces and sub-edges by calling `FaceProc` on the appropriate children [S1, unofficial].

### Edge Processing
- **EdgeProcXY/XZ/YZ(n0, n1, n2, n3, indexBuffer)**:
  - Check if all 4 adjacent nodes are leaves [S1, unofficial].
  - If so, call `ProcessEdge(n0, n1, n2, n3, dir, indexBuffer)` [S1, unofficial].
  - If not, recursively call `EdgeProc` on specific child indices to descend into the octree [S1, unofficial].

### ProcessEdge(n0, n1, n2, n3, dir, indexBuffer)
1. Initialize `minSize` to infinity, `minIndex` to -1 [S1, unofficial].
2. Initialize `indices` array of size 4 [S1, unofficial].
3. For `i` from 0 to 3:
   - `node = [n0, n1, n2, n3][i]` [S1, unofficial].
   - If `node` is `nullptr`, continue [S1, unofficial].
   - `edgeIdx = processEdgeMask[dir][i]` [S1, unofficial].
   - `c1 = (node->m_corners >> edgeIdx[0]) & 1` [S1, unofficial].
   - `c2 = (node->m_corners >> edgeIdx[1]) & 1` [S1, unofficial].
   - `m1 = (c1 == 1) ? MATERIAL_SOLID : MATERIAL_AIR` [S1, unofficial].
   - `m2 = (c2 == 1) ? MATERIAL_SOLID : MATERIAL_AIR` [S1, unofficial].
   - `signChange = (m1 != m2)` [S1, unofficial].
   - If `signChange`:
     - `indices[i] = node->m_index` [S1, unofficial].
     - If `node->m_size < minSize`:
       - `minSize = node->m_size` [S1, unofficial].
       - `minIndex = i` [S1, unofficial].
     - `flip = (m1 == MATERIAL_SOLID)` [S1, unofficial].
4. If `minIndex != -1`:
   - Generate 2 triangles (6 indices) using `indices[0..3]` [S1, unofficial].
   - If `flip` is false:
     - Add indices `(indices[0], indices[1], indices[3])` [S1, unofficial].
     - Add indices `(indices[0], indices[3], indices[2])` [S1, unofficial].
   - If `flip` is true:
     - Add indices `(indices[0], indices[3], indices[1])` [S1, unofficial].
     - Add indices `(indices[0], indices[2], indices[3])` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def Sample(p):
    # Standard scalar field evaluation [G]
    pass

def LeafOrChild(node, idx):
    if node.is_leaf:
        return node
    if node.children[idx] is not None:
        return node.children[idx]
    return None

def CellProc(node):
    if node.is_leaf:
        return
    for i in range(8):
        child = node.children[i]
        if child is not None:
            CellChildProc(child)

def CellChildProc(node):
    # Process X-faces
    FaceProcX(node.children[0], node.children[1], node.children[2], node.children[3], index_buffer)
    FaceProcX(node.children[4], node.children[5], node.children[6], node.children[7], index_buffer)
    # Process Y-faces
    FaceProcY(node.children[0], node.children[1], node.children[4], node.children[5], index_buffer)
    FaceProcY(node.children[2], node.children[3], node.children[6], node.children[7], index_buffer)
    # Process Z-faces
    FaceProcZ(node.children[0], node.children[2], node.children[4], node.children[6], index_buffer)
    FaceProcZ(node.children[1], node.children[3], node.children[5], node.children[7], index_buffer)
    
    # Process Edges
    EdgeProcXY(node.children[0], node.children[1], node.children[2], node.children[3], index_buffer)
    EdgeProcXY(node.children[4], node.children[5], node.children[6], node.children[7], index_buffer)
    EdgeProcXZ(node.children[0], node.children[2], node.children[4], node.children[6], index_buffer)
    EdgeProcXZ(node.children[1], node.children[3], node.children[5], node.children[7], index_buffer)
    EdgeProcYZ(node.children[0], node.children[1], node.children[4], node.children[5], index_buffer)
    EdgeProcYZ(node.children[2], node.children[3], node.children[6], node.children[7], index_buffer)
    
    # Recurse
    for i in range(8):
        child = node.children[i]
        if child is not None:
            CellProc(child)

def FaceProcX(n0, n1, n2, n3, idx_buf):
    if all(n.is_leaf for n in [n0, n1, n2, n3]):
        return
    FaceProcX(LeafOrChild(n0, 0), LeafOrChild(n1, 0), LeafOrChild(n2, 0), LeafOrChild(n3, 0), idx_buf)
    FaceProcX(LeafOrChild(n0, 1), LeafOrChild(n1, 1), LeafOrChild(n2, 1), LeafOrChild(n3, 1), idx_buf)
    FaceProcX(LeafOrChild(n0, 2), LeafOrChild(n1, 2), LeafOrChild(n2, 2), LeafOrChild(n3, 2), idx_buf)
    FaceProcX(LeafOrChild(n0, 3), LeafOrChild(n1, 3), LeafOrChild(n2, 3), LeafOrChild(n3, 3), idx_buf)

def FaceProcY(n0, n1, n2, n3, idx_buf):
    if all(n.is_leaf for n in [n0, n1, n2, n3]):
        return
    FaceProcY(LeafOrChild(n0, 0), LeafOrChild(n1, 0), LeafOrChild(n2, 0), LeafOrChild(n3, 0), idx_buf)
    FaceProcY(LeafOrChild(n0, 1), LeafOrChild(n1, 1), LeafOrChild(n2, 1), LeafOrChild(n3, 1), idx_buf)
    FaceProcY(LeafOrChild(n0, 2), LeafOrChild(n1, 2), LeafOrChild(n2, 2), LeafOrChild(n3, 2), idx_buf)
    FaceProcY(LeafOrChild(n0, 3), LeafOrChild(n1, 3), LeafOrChild(n2, 3), LeafOrChild(n3, 3), idx_buf)

def FaceProcZ(n0, n1, n2, n3, idx_buf):
    if all(n.is_leaf for n in [n0, n1, n2, n3]):
        return
    FaceProcZ(LeafOrChild(n0, 0), LeafOrChild(n1, 0), LeafOrChild(n2, 0), LeafOrChild(n3, 0), idx_buf)
    FaceProcZ(LeafOrChild(n0, 1), LeafOrChild(n1, 1), LeafOrChild(n2, 1), LeafOrChild(n3, 1), idx_buf)
    FaceProcZ(LeafOrChild(n0, 2), LeafOrChild(n1, 2), LeafOrChild(n2, 2), LeafOrChild(n3, 2), idx_buf)
    FaceProcZ(LeafOrChild(n0, 3), LeafOrChild(n1, 3), LeafOrChild(n2, 3), LeafOrChild(n3, 3), idx_buf)

def EdgeProcXY(n0, n1, n2, n3, idx_buf):
    if all(n.is_leaf for n in [n0, n1, n2, n3]):
        ProcessEdge([n0, n1, n2, n3], 0, idx_buf)
    else:
        EdgeProcXY(LeafOrChild(n0, 3), LeafOrChild(n1, 2), LeafOrChild(n2, 1), LeafOrChild(n3, 0), idx_buf)

def EdgeProcXZ(n0, n1, n2, n3, idx_buf):
    if all(n.is_leaf for n in [n0, n1, n2, n3]):
        ProcessEdge([n0, n1, n2, n3], 1, idx_buf)
    else:
        EdgeProcXZ(LeafOrChild(n0, 3), LeafOrChild(n1, 2), LeafOrChild(n2, 1), LeafOrChild(n3, 0), idx_buf)

def EdgeProcYZ(n0, n1, n2, n3, idx_buf):
    if all(n.is_leaf for n in [n0, n1, n2, n3]):
        ProcessEdge([n0, n1, n2, n3], 2, idx_buf)
    else:
        EdgeProcYZ(LeafOrChild(n0, 3), LeafOrChild(n1, 2), LeafOrChild(n2, 1), LeafOrChild(n3, 0), idx_buf)

def ProcessEdge(nodes, dir, idx_buf):
    min_size = float('inf')
    min_index = -1
    indices = [None] * 4
    flip = False
    
    edgevmap = [
        [0, 4], [1, 5], [2, 6], [3, 7],
        [0, 2], [1, 3], [4, 6], [5, 7],
        [0, 1], [2, 3], [4, 5], [6, 7]
    ]
    processEdgeMask = [
        [3, 2, 1, 0],
        [7, 5, 6, 4],
        [11, 10, 9, 8]
    ]
    
    for i in range(4):
        node = nodes[i]
        if node is None:
            continue
        edge_idx = processEdgeMask[dir][i]
        c1 = (node.m_corners >> edgevmap[edge_idx][0]) & 1
        c2 = (node.m_corners >> edgevmap[edge_idx][1]) & 1
        m1 = 1 if c1 == 1 else 0
        m2 = 1 if c2 == 1 else 0
        sign_change = (m1 != m2)
        
        if sign_change:
            indices[i] = node.m_index
            if node.m_size < min_size:
                min_size = node.m_size
                min_index = i
            flip = (m1 == 1)
            
    if min_index != -1:
        if not flip:
            idx_buf.add(indices[0], indices[1], indices[3])
            idx_buf.add(indices[0], indices[3], indices[2])
        else:
            idx_buf.add(indices[0], indices[3], indices[1])
            idx_buf.add(indices[0], indices[2], indices[3])

def ApproximateZeroCrossingPosition(p0, p1):
    min_error = 0.01
    current_t = 0.5
    increment = 0.5
    for _ in range(8):
        p = p0 + (p1 - p0) * current_t
        density = abs(Sample(p))
        if density < min_error:
            return p
        if density > 0:
            current_t -= increment
        else:
            current_t += increment
        increment *= 0.5
    return p0 + (p1 - p0) * current_t
```

## Operating characteristics
- **Time Complexity**: Depends on the octree depth and the number of boundary voxels. Each edge is processed once, and recursion depth is proportional to the octree height [S1, unofficial].
- **Memory Complexity**: Proportional to the number of octree nodes. Each node stores a small amount of metadata [S1, unofficial].
- **Strengths**: Adaptive resolution allows for detailed geometry where needed and coarser geometry elsewhere. Efficient handling of large volumetric datasets [S1, unofficial].
- **Limitations**: May produce non-manifold geometry if not carefully implemented. The specific edge processing logic may require additional steps to ensure manifoldness [S1, unofficial].
- **When to Use**: Suitable for adaptive volumetric data where resolution varies significantly across the domain [S1, unofficial].
- **When Not to Use**: For uniform grids, simpler methods like Marching Cubes may be more efficient [S1, unofficial].

## Background
This method is based on the implementation found in the `DualContouringSample` project by Tao Ju [S1, unofficial]. It is designed to extract polygonal meshes from volumetric scalar fields using an adaptive octree structure. The approach is particularly useful for procedural generation and terrain modeling where adaptive resolution is beneficial [S1, unofficial].

## Sources
[S1] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
