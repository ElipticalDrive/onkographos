# Edge Contouring for Dual Contouring

Also called: ContourEdgeProc, ContourProcessEdge

## Overview
Edge Contouring is a recursive procedure that processes the edges of an adaptive octree to generate triangle indices for a dual contouring mesh [S1, unofficial]. It builds on [Manifold Dual Contouring - Index Processing and Triangle Generation](3699-manifold-dual-contouring-index-processing-and-triangle-gener.md) by providing the specific octree-aware traversal logic required for adaptive volumetric data [G].

## How it works
**Inputs:**
- `nodes`: An array of 4 `OctreeNode` pointers or `null` [S1, unofficial].
- `dir`: An integer representing the edge direction (0, 1, or 2) [S1, unofficial].
- `indexBuffer`: A reference to an index buffer or list where generated triangle indices are appended [G].

**Outputs:**
- Appends triangle indices to `indexBuffer` [G].

**Data Structures:**
- `edgeProcEdgeMask`: A 3D array of integers with layout `[3][2][5]` that maps edge direction `dir`, sub-edge index `i`, and mask components [S1, unofficial].
  - Components 0-3: Child indices for the four nodes in the current level [S1, unofficial].
  - Component 4: The sub-direction for the recursive call [S1, unofficial].
- `Node` types: `Node_Leaf`, `Node_Pseudo` (spelled `Node_Psuedo` in the source [S1, unofficial]), and `Node_Internal` [S1, unofficial].

**Steps:**
1. **Base Case:** If any of the 4 input nodes in `nodes` is `null`, return immediately [S1, unofficial].
2. **Leaf/Pseudo Case:** If all 4 nodes are not `Node_Internal` (i.e., they are `Node_Leaf` or `Node_Pseudo`), call `ContourProcessEdge(nodes, dir, indexBuffer)` [S1, unofficial].
3. **Internal Case:** If any node is `Node_Internal`, iterate `i` from 0 to 1:
   a. Extract child indices `c[0..3]` from `edgeProcEdgeMask[dir][i][0..3]` [S1, unofficial].
   b. Create a new array `edgeNodes` of size 4 [S1, unofficial].
   c. For each `j` from 0 to 3:
      - If `nodes[j]` is `null`, `edgeNodes[j]` remains `null` [S1, unofficial].
      - If `nodes[j]` is `Node_Leaf` or `Node_Pseudo`, set `edgeNodes[j] = nodes[j]` [S1, unofficial].
      - If `nodes[j]` is `Node_Internal`, set `edgeNodes[j] = nodes[j].children[c[j]]` [S1, unofficial].
   d. Recurse: Call `ContourEdgeProc(edgeNodes, edgeProcEdgeMask[dir][i][4], indexBuffer)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Node type constants
Node_Leaf = 0
Node_Pseudo = 1
Node_Internal = 2

# edgeProcEdgeMask is a 3D array [3][2][5] provided by the implementation
# edgeProcEdgeMask[dir][i][0:4] stores child indices
# edgeProcEdgeMask[dir][i][4] stores the sub-direction

def ContourProcessEdge(nodes, dir, indexBuffer):
    # The sources do not give the implementation details for this helper function.
    pass

def ContourEdgeProc(nodes, dir, indexBuffer):
    # nodes: list of 4 OctreeNode or None
    # dir: int (0, 1, 2)
    # indexBuffer: list of ints

    # Base case: if any node is null, return
    for node in nodes:
        if node is None:
            return

    # Leaf/Pseudo case: if all nodes are not Internal
    all_leaf_or_pseudo = True
    for node in nodes:
        if node.type == Node_Internal:
            all_leaf_or_pseudo = False
            break

    if all_leaf_or_pseudo:
        ContourProcessEdge(nodes, dir, indexBuffer)
        return

    # Internal case: recurse into children
    for i in range(2):
        # Extract child indices from mask
        child_indices = edgeProcEdgeMask[dir][i][0:4]
        sub_dir = edgeProcEdgeMask[dir][i][4]

        edge_nodes = [None] * 4
        for j in range(4):
            if nodes[j] is None:
                edge_nodes[j] = None
            elif nodes[j].type in (Node_Leaf, Node_Pseudo):
                edge_nodes[j] = nodes[j]
            else:
                # Node is Internal, get child
                child_idx = child_indices[j]
                edge_nodes[j] = nodes[j].children[child_idx]

        # Recurse
        ContourEdgeProc(edge_nodes, sub_dir, indexBuffer)
```

**Parameters:**
- `edgeProcEdgeMask`: A 3D array `[3][2][5]` mapping edge direction `dir`, sub-edge index `i`, and mask components. Components 0-3 are child indices, component 4 is the sub-direction [S1, unofficial].

**Edge Cases and Pitfalls:**
- Null nodes: The algorithm explicitly checks for null nodes at the start and handles them by skipping recursion for that branch [S1, unofficial].
- Pseudo-leaf nodes: Nodes converted from internal to pseudo-leaf (e.g., due to QEF mass point projection) are treated as leaves for contouring purposes [S1, unofficial].

## Subtopics

### [Octree-based Dual Contouring Edge/Face Processing](4297-octree-based-dual-contouring-edge-face-processing.md)
This method is a recursive procedure that processes the edges and faces of an adaptive octree to generate triangle indices for a dual contouring mesh [G]. It handles both leaf and pseudo-leaf nodes by descending into children when necessary [G]. It builds on [Edge Contouring for Dual Contouring](3711-edge-contouring-for-dual-contouring.md) by providing the specific octree-aware traversal and edge processing logic required for adaptive volumetric data [G].

### Manifold Dual Contouring - Edge Clustering
If all four nodes in the `nodes` array are either `null` or `Node_Leaf` (not `Node_Internal`), call `ClusterIndexes(nodes, direction, intHolder, collected_vertices)` [S2, unofficial]. Else, loop `i` from 0 to 1. Create `edge_nodes` array size 4. Loop `j` from 0 to 3: if `nodes[j]` is `null`, continue. If `nodes[j].Type == Node_Leaf`, `edge_nodes[j] = nodes[j]`. Else, `edge_nodes[j] = nodes[j].children[Utilities.TEdgeProcEdgeMask[direction][i][j]]` [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method is part of the Dual Contouring implementation in the `DualContouringSample` project by Tao Ju, as distributed by nickgildea [S1, unofficial]. The code is licensed under the GNU Lesser General Public License (LGPL) version 2.1 or later [S1, unofficial].

## Sources
[S1] DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S2] Manifold-dual-contouring-Java, https://github.com/proton2/Manifold-dual-contouring-Java, fetched 2026-10-26 (unofficial)
