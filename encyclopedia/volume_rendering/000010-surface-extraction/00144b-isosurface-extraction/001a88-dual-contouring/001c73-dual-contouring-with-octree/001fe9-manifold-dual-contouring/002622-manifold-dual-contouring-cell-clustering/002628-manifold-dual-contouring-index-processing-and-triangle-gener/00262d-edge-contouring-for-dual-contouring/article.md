# Edge Contouring for Dual Contouring

Also called: ContourEdgeProc, ContourProcessEdge

## Overview
A recursive procedure that processes edges of the octree to generate triangle indices for the dual contouring mesh, handling both leaf/pseudo-leaf nodes and internal nodes by recursing into children [S1, unofficial]. It serves as the core traversal mechanism in Manifold Dual Contouring implementations to ensure manifold connectivity by processing edges in a specific order and resolving internal node intersections via child recursion [S1, unofficial].

## How it works
**Inputs:**
- `nodes`: An array of 4 `OctreeNode` pointers representing the endpoints and intermediate nodes of an octree edge.
- `dir`: The edge direction index (0, 1, or 2).
- `indexBuffer`: A buffer to which generated triangle indices are appended.

**Data Structures:**
- `edgeProcEdgeMask`: A 3D array with layout `[3][2][5]`. It maps edge direction `dir`, sub-edge index `i`, and mask components. Components 0-3 are child indices, and component 4 is the sub-direction [S1, unofficial].
- `Node` types: `Node_Internal`, `Node_Leaf`, `Node_Psuedo` (note: source uses "Psuedo" spelling), and `null` [S1, unofficial].

**Steps:**
1. **Base Case Check:** If any of the 4 input nodes is `null`, return immediately [S1, unofficial].
2. **Leaf/Pseudo Case:** If all 4 nodes are not `Node_Internal` (i.e., they are `Node_Leaf`, `Node_Psuedo`, or `null`), call `ContourProcessEdge(node, dir, indexBuffer)` [S1, unofficial].
3. **Internal Case:** If any node is `Node_Internal`:
   - Loop `i` from 0 to 1.
   - Extract child indices `c[0..3]` from `edgeProcEdgeMask[dir][i][0..3]` [S1, unofficial].
   - For each `j` from 0 to 3:
     - If `node[j]` is a leaf or pseudo-leaf, use `node[j]` as the child node.
     - Else, use `node[j]->children[c[j]]` as the child node [S1, unofficial].
   - Recurse with `ContourEdgeProc(edgeNodes, edgeProcEdgeMask[dir][i][4], indexBuffer)`, where `edgeProcEdgeMask[dir][i][4]` is the sub-direction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ContourEdgeProc(nodes: List[Optional[OctreeNode]], dir: int, indexBuffer: IndexBuffer):
    # Step 1: Base case check
    for node in nodes:
        if node is None:
            return

    # Step 2: Leaf/Pseudo case
    all_non_internal = all(node.type != Node_Internal for node in nodes)
    if all_non_internal:
        ContourProcessEdge(nodes, dir, indexBuffer)
        return

    # Step 3: Internal case
    for i in range(2):
        # Extract child indices from mask
        child_indices = edgeProcEdgeMask[dir][i][0:4]
        sub_dir = edgeProcEdgeMask[dir][i][4]
        
        edge_nodes = []
        for j in range(4):
            node = nodes[j]
            if node.type in (Node_Leaf, Node_Psuedo):
                edge_nodes.append(node)
            else:
                # node is Node_Internal
                child_idx = child_indices[j]
                edge_nodes.append(node.children[child_idx])
        
        # Recurse
        ContourEdgeProc(edge_nodes, sub_dir, indexBuffer)
```

**Parameters:**
- `edgeProcEdgeMask`: A 3D array `[3][2][5]` mapping edge direction `dir`, sub-edge index `i`, and mask components. Components 0-3 are child indices, component 4 is the sub-direction [S1, unofficial].

**Edge Cases and Pitfalls:**
- Null nodes are handled by early return [S1, unofficial].
- The term "pseudo-leaf" is used in the source code, spelled "Psuedo" [S1, unofficial].
- The mask array structure is critical for correct child selection and sub-direction propagation [S1, unofficial].

## Subtopics

### Manifold Dual Contouring - Edge Clustering
If all four nodes in the nodes array are either null or Node_Leaf (not Node_Internal), call `ClusterIndexes(nodes, direction, intHolder, collected_vertices)` [S2, unofficial]. Else, loop `i` from 0 to 1. Create `edge_nodes` array size 4. Loop `j` from 0 to 3: if `nodes[j]` is null, continue. If `nodes[j].Type == Node_Leaf`, `edge_nodes[j] = nodes[j]`. Else, `edge_nodes[j] = nodes[j].children[Utilities.TEdgeProcEdgeMask[direction][i][j]]` [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Origin: This method is part of the Dual Contouring implementation by Tao Ju, as seen in the `DualContouringSample` repository [S1, unofficial]. It is specifically used in the context of Manifold Dual Contouring to handle octree edge processing [S1, unofficial].

## Sources
[S1] DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S2] Manifold-dual-contouring-Java, https://github.com/proton2/Manifold-dual-contouring-Java, fetched 2026-10-26 (unofficial)
