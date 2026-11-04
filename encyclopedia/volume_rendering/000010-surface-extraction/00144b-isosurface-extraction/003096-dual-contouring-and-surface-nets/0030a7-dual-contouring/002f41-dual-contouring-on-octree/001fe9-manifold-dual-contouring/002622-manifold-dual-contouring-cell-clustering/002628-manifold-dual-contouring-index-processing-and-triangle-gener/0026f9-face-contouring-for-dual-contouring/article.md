# Face Contouring for Dual Contouring

Also called: ContourFaceProc

## Overview
Face Contouring (also called `ContourFaceProc`) is a recursive procedure that processes the faces of an octree to generate triangle indices during the mesh generation phase of Manifold Dual Contouring [S1, unofficial]. It operates on pairs of octree nodes, recursing into children for internal nodes and invoking the sibling Edge Contouring method for leaf nodes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `faceNodes`: An array of two `OctreeNode` pointers representing the two octree nodes sharing the face being processed [S1, unofficial].
- `dir`: The face direction index (0, 1, or 2), corresponding to the axis perpendicular to the face [S1, unofficial].
- `indexBuffer`: A reference to the index buffer where generated triangle indices are appended [S1, unofficial].

**Data Structures:**
The method relies on precomputed lookup tables provided by the implementation [S1, unofficial]:
1. `faceProcFaceMask[3][4][3]`: Maps face direction `dir`, face index `i`, and components. Components 0 and 1 are child indices for the two nodes [S1, unofficial]. Component 2 is the sub-direction for recursion [S1, unofficial].
2. `faceProcEdgeMask[3][4][6]`: Maps face direction `dir`, face index `i`, and components. Component 0 is an order index (0 or 1) [S1, unofficial]. Components 1-4 are child indices for the four edge nodes [S1, unofficial]. Component 5 is the sub-direction for the edge contouring call [S1, unofficial].
3. `orders[2][4]`: A static array defining node access order: `{{0,0,1,1}, {0,1,0,1}}` [S1, unofficial].

**Algorithm Steps:**
1. **Base Case:** If either node in `faceNodes` is null, return immediately [S1, unofficial].
2. **Internal Node Handling:** If either node is of type `Node_Internal`, iterate `i` from 0 to 3 [S1, unofficial]. Extract child indices `c_face[0] = faceProcFaceMask[dir][i][0]` and `c_face[1] = faceProcFaceMask[dir][i][1]` [S1, unofficial]. For `j` in {0, 1}, resolve the node pointer: if `faceNodes[j]` is not internal, use it; otherwise, use `faceNodes[j]->children[c_face[j]]` [S1, unofficial]. Recurse with `ContourFaceProc` using the resolved nodes, sub-direction `faceProcFaceMask[dir][i][2]`, and `indexBuffer` [S1, unofficial].
3. **Edge Processing:** Define `orders[2][4] = {{0,0,1,1}, {0,1,0,1}}` [S1, unofficial]. Iterate `i` from 0 to 3 [S1, unofficial]. Get order index `orderIdx = faceProcEdgeMask[dir][i][0]` and child indices `c_edge[0..3]` from `faceProcEdgeMask[dir][i][1..4]` [S1, unofficial]. Get `order = orders[orderIdx]` [S1, unofficial]. For `j` from 0 to 3, resolve edge nodes: if `faceNodes[order[j]]` is a leaf or pseudo-node, use it; otherwise, use `faceNodes[order[j]]->children[c_edge[j]]` [S1, unofficial]. Recurse with `ContourEdgeProc` using the four resolved edge nodes, sub-direction `faceProcEdgeMask[dir][i][5]`, and `indexBuffer` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Type definitions for self-containment
Node_Internal = 1
Node_Leaf = 0
Node_Pseudo = 2

# Precomputed lookup tables provided by the implementation [S1, unofficial]
FACE_PROC_FACE_MASK = [[[0, 0, 0] for _ in range(4)] for _ in range(3)]
FACE_PROC_EDGE_MASK = [[[0, 0, 0, 0, 0, 0] for _ in range(4)] for _ in range(3)]

def ContourFaceProc(faceNodes: list, dir: int, indexBuffer: list):
    # Base case: null check
    if faceNodes[0] is None or faceNodes[1] is None:
        return

    # Internal case: recurse into children faces
    if faceNodes[0].type == Node_Internal or faceNodes[1].type == Node_Internal:
        for i in range(4):
            c_face0 = FACE_PROC_FACE_MASK[dir][i][0]
            c_face1 = FACE_PROC_FACE_MASK[dir][i][1]
            sub_dir = FACE_PROC_FACE_MASK[dir][i][2]

            # Resolve node pointers safely
            node0 = faceNodes[0] if faceNodes[0].type != Node_Internal else faceNodes[0].children[c_face0]
            node1 = faceNodes[1] if faceNodes[1].type != Node_Internal else faceNodes[1].children[c_face1]

            ContourFaceProc([node0, node1], sub_dir, indexBuffer)

    # Edge processing
    orders = [[0, 0, 1, 1], [0, 1, 0, 1]]

    for i in range(4):
        order_idx = FACE_PROC_EDGE_MASK[dir][i][0]
        child_indices = FACE_PROC_EDGE_MASK[dir][i][1:5]
        sub_dir = FACE_PROC_EDGE_MASK[dir][i][5]
        order = orders[order_idx]

        edge_nodes = [None] * 4
        for j in range(4):
            node_idx = order[j]
            child_idx = child_indices[j]

            # Resolve edge node: use leaf/pseudo directly, else dereference children
            if faceNodes[node_idx].type != Node_Internal:
                edge_nodes[j] = faceNodes[node_idx]
            else:
                edge_nodes[j] = faceNodes[node_idx].children[child_idx]

        # Sibling method: generates indices for the edge contour [G]
        # ContourEdgeProc(edge_nodes, sub_dir, indexBuffer)
```

**Parameters and Edge Cases:**
- `faceProcFaceMask` and `faceProcEdgeMask` are precomputed lookup tables provided by the implementation; their exact values are implementation-specific [S1, unofficial].
- Pitfall: Accessing `children` on a non-internal node will cause a crash; the type check `Node_Internal` is mandatory before dereferencing `children` [S1, unofficial].
- Pitfall: The `order` array indices must map correctly to the `faceNodes` array indices (0 or 1) to ensure the correct geometric relationship between the two nodes is maintained during edge resolution [S1, unofficial].

## Operating characteristics
- Time and memory complexity: Not given in the sources [S1, unofficial].
- Measured performance, scaling, strengths, limitations, and usage recommendations: Not given in the sources [S1, unofficial].

## Background
This method is part of the Manifold Dual Contouring implementation by Tao Ju, as reproduced in the `DualContouringSample` repository [S1, unofficial]. It is a critical component of the index generation phase, bridging the gap between vertex generation (QEF solving) and final triangle assembly [S1, unofficial].

## Sources
[S1] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
