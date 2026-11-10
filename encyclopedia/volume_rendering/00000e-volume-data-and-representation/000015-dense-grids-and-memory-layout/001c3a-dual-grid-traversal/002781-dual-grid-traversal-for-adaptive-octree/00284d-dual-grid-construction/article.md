# Dual Grid Construction

Also called: Dual Cell Generation, Adaptive Octree Dual Grid

## Overview
Dual Grid Construction generates a topological structure (the dual grid) from an adaptive octree, where each vertex in the dual grid corresponds to a junction point in the octree hierarchy. It solves the problem of creating a manifold mesh or traversal structure from non-uniform voxel data by explicitly handling boundaries between nodes of different sizes. It differs from uniform grid duals by using a set of 2–8 surrounding octree nodes to define each dual vertex, allowing it to bridge gaps where large nodes abut smaller ones [S1, unofficial].

## How it works
**Inputs:** An adaptive octree root node.
**Outputs:** A collection of `DualCell` objects (or indices into vertex/face buffers).

**Data Structures:**
*   `OctreeNode`: Contains `Bounds` (AABB), `Depth`, and `Children` (array of 8 `OctreeNode` or `null` if leaf).
*   `DualCell`: A struct containing 8 references to `OctreeNode` objects (`n0` through `n7`). These references are ordered identically to the standard octree child ordering (0: -x,-y,-z; 1: +x,-y,-z; etc.). The array may contain duplicate references if adjacent nodes have different sizes [S1, unofficial].
*   `Vertex` property of `DualCell`: Defined as `n0.Bounds.max` [S1, unofficial]. Note: In a consistent implementation, this vertex position should be geometrically consistent across all cells sharing it, often derived from the bounding box intersection of the participating nodes.

**Algorithm Steps:**
1.  **Recursive Traversal:** Start at the root. If the current node is a leaf (`Children == null`), yield a break or return [S1, unofficial].
2.  **Child Traversal:** Recursively traverse all 8 children. Collect results from these recursive calls [S1, unofficial].
3.  **Face Processing:** For each pair of adjacent children sharing an X, Y, or Z face, generate a dual cell representing that interface.
    *   **X-faces:** Pairs `(0,1)`, `(3,2)`, `(4,5)`, `(7,6)` [S1, unofficial].
    *   **Y-faces:** Pairs `(0,4)`, `(1,5)`, `(3,7)`, `(2,6)` [S1, unofficial].
    *   **Z-faces:** Pairs `(0,3)`, `(1,2)`, `(4,7)`, `(5,6)` [S1, unofficial].
4.  **Edge Processing:** For each quartet of children sharing an edge, generate a dual cell representing that edge junction.
    *   **X-edges:** Quartets `(0,3,7,4)`, `(1,2,6,5)` [S1, unofficial].
    *   **Y-edges:** Quartets `(0,1,2,3)`, `(4,5,6,7)` [S1, unofficial].
    *   **Z-edges:** Quartets `(3,2,6,7)`, `(1,0,4,5)` (pattern inferred from standard octree indexing) [S1, unofficial].
5.  **Vertex Processing:** (Implicit in some implementations, explicit in others) Generate dual cells for the internal vertices of the node, involving all 8 children.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_dual_grid(node: OctreeNode) -> List[DualCell]:
    cells = []
    
    # Base case: Leaf node
    if node.children is None:
        return cells

    # Recursive step: Process children first
    for child in node.children:
        cells.extend(build_dual_grid(child))

    # Process interfaces within the current node's 8 children
    children = node.children
    
    # 1. Process Faces
    # X-faces (left/right pairs)
    face_pairs_x = [(0, 1), (3, 2), (4, 5), (7, 6)]
    for i, j in face_pairs_x:
        cells.append(create_dual_cell(children[i], children[j], type='face', axis='x'))
        
    # Y-faces (bottom/top pairs)
    face_pairs_y = [(0, 4), (1, 5), (3, 7), (2, 6)]
    for i, j in face_pairs_y:
        cells.append(create_dual_cell(children[i], children[j], type='face', axis='y'))
        
    # Z-faces (front/back pairs)
    face_pairs_z = [(0, 3), (1, 2), (4, 7), (5, 6)]
    for i, j in face_pairs_z:
        cells.append(create_dual_cell(children[i], children[j], type='face', axis='z'))

    # 2. Process Edges
    # X-edges (quartets sharing an X-aligned edge)
    edge_quartets_x = [(0, 3, 7, 4), (1, 2, 6, 5)]
    for q in edge_quartets_x:
        cells.append(create_dual_cell([children[i] for i in q], type='edge', axis='x'))
        
    # Y-edges (quartets sharing a Y-aligned edge)
    edge_quartets_y = [(0, 1, 2, 3), (4, 5, 6, 7)]
    for q in edge_quartets_y:
        cells.append(create_dual_cell([children[i] for i in q], type='edge', axis='y'))
        
    # Z-edges (quartets sharing a Z-aligned edge)
    edge_quartets_z = [(3, 2, 6, 7), (1, 0, 4, 5)] # Inferred pattern
    for q in edge_quartets_z:
        cells.append(create_dual_cell([children[i] for i in q], type='edge', axis='z'))

    return cells

def create_dual_cell(nodes: List[OctreeNode], type: str, axis: str) -> DualCell:
    """
    Creates a DualCell from a list of nodes.
    The DualCell stores references to the nodes.
    Duplicates are allowed if nodes are of different sizes [S1, unofficial].
    """
    # Pad to 8 nodes if necessary, or use the specific subset depending on implementation
    # Standard DualCell expects 8 references. 
    # For faces/edges, some references might be null or duplicates of existing ones 
    # to maintain the 8-slot structure, or the implementation uses a variable-length list.
    # Based on [S1, unofficial], DualCell contains 8 OctreeNode references.
    
    # Filling missing nodes with duplicates of existing ones or nulls depends on specific 
    # traversal logic. [S1, unofficial] implies handling duplicates. 
    # A robust implementation fills the 8 slots based on the geometric position 
    # relative to the dual vertex.
    
    # Simplified: Return a cell with the provided nodes, padded to 8 if needed.
    # In practice, 'n0' is often the node at the minimum corner of the dual vertex.
    
    cell = DualCell()
    # Map nodes to n0..n7 based on their position in the octree
    # This mapping requires knowing the local index of each node within the parent.
    # For face/edge processing, the caller usually passes the specific children involved.
    
    # Note: Exact mapping logic for partial sets (faces/edges) to 8-slot DualCell 
    # is not fully detailed in sources, but [S1, unofficial] states n0.Bounds.max is the vertex.
    # We assume the input list corresponds to the relevant corners.
    
    # Placeholder for exact 8-slot mapping logic which is implementation-specific
    # but must preserve the adjacency relationships.
    return cell
```

**Parameters:**
*   Recursion depth: Determined by octree construction.
*   Thresholds: Not part of construction itself, but the octree structure depends on value thresholds used during octree generation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Duplicate References:** Large nodes adjacent to smaller nodes cause the same `OctreeNode` reference to appear multiple times in a `DualCell`'s 8-node array. The algorithm explicitly allows these duplicates [S1, unofficial].
*   **Boundary Conditions:** Nodes at the boundary of the octree root may have missing children (conceptually null or outside bounds). These must be handled, often by treating them as leaves or skipping interface generation for missing sides [S1, S2, unofficial].
*   **Vertex Consistency:** The `Vertex` property is defined as `n0.Bounds.max` [S1, unofficial]. For the dual mesh to be manifold, all `DualCell` objects sharing a geometric vertex must compute the same position. This requires careful handling of the `n0` selection or a global vertex lookup.

## Subtopics

### Octree Mesh Generation
This variant focuses on generating the final triangle mesh indices rather than just the dual cells. It calls `GenerateVertexIndices` with the root node and a vertex buffer to populate vertex positions, then calls `CellProc` (or similar face/edge processing functions) to populate the index buffer with triangle primitives [S2, unofficial]. It relies on the dual grid construction to identify the topology but adds the step of converting that topology into renderable geometry.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of nodes in the octree. Each node is visited once, and a constant number of dual cells (faces, edges, vertices) are generated per node.
*   **Memory Complexity:** $O(N)$ for storing the dual cells. Each dual cell stores 8 node references.
*   **Strengths:** Handles arbitrary adaptive levels of detail without requiring a uniform grid. Efficiently generates topology for marching cubes or similar algorithms on sparse data.
*   **Limitations:** The dual grid can be non-manifold if not carefully constructed (e.g., T-junctions). Duplicate node references in `DualCell` require special handling during mesh generation to avoid degenerate triangles.
*   **When to use:** When rendering adaptive octree data (e.g., signed distance fields, noise volumes) where uniform grid resolution is too expensive or memory-intensive.

## Background
The concept of dual grids in the context of adaptive octrees originates from the need to apply marching cubes-like algorithms to non-uniform voxel data. The specific implementation details described here are derived from open-source implementations of Dual Marching Cubes for adaptive octrees [S1, S2, unofficial]. The approach allows for the generation of smooth surfaces from hierarchical data structures commonly used in procedural generation and scientific visualization.

## Sources
[S1] heyx3/DualMarchingCubes, https://github.com/heyx3/DualMarchingCubes, fetched 2026-10-27 (unofficial)
[S2] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
