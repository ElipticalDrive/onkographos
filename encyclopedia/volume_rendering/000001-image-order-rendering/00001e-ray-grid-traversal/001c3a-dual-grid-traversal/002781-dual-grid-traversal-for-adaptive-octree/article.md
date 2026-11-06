# Dual Grid Traversal for Adaptive Octree


## Overview
This method traverses an adaptive octree to generate a dual grid representation by identifying all dual cells (vertices) that straddle the boundaries between octree nodes of varying depths [S1, unofficial]. It handles cases where adjacent nodes have different depths by using parent nodes as stand-ins for missing children, ensuring the dual grid remains connected across level boundaries [S1, unofficial]. Unlike methods that process uniform grids, this algorithm recursively visits faces, edges, and vertices at refinement boundaries to maintain topological consistency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The root node of an adaptive octree [S1, unofficial].
- Output: An enumerable sequence of `DualCell` objects [S1, unofficial].

**Data Structures:**
- `OctreeNode`: Contains `Bounds` (axis-aligned bounding box), `Depth` (integer), and `Children` (array of 8 `OctreeNode` references or `null` if it is a leaf) [S1, unofficial].
- `DualCell`: A struct containing 8 fields named `n0` through `n7`, each of type `OctreeNode` [S1, unofficial]. These fields reference the octree nodes surrounding the dual vertex [S1, unofficial]. The order corresponds to the standard octree child ordering (0=lower-left-back, 7=upper-right-front) [G]. Some fields may reference the same node if the dual vertex lies on the boundary of a larger parent node [S1, unofficial].

**Algorithm Steps:**
The traversal is performed recursively via three main functions: `TraverseVertex`, `TraverseFace`, and `TraverseEdge` [S1, unofficial]. The process starts by calling these functions on the root node's children to cover all internal faces, edges, and the central vertex [S1, unofficial].

1.  **TraverseVertex(n0, n1, n2, n3, n4, n5, n6, n7)**:
    -   This function processes the dual vertex at the center of the 8 octree nodes `n0` through `n7` [S1, unofficial].
    -   If any of the 8 nodes has children (is not a leaf), recursively call `TraverseVertex` on the specific child indices that surround the central vertex [S1, unofficial]. The mapping from parent index to child index is fixed: `n0` → `Children[6]`, `n1` → `Children[7]`, `n2` → `Children[4]`, `n3` → `Children[5]`, `n4` → `Children[2]`, `n5` → `Children[3]`, `n6` → `Children[0]`, `n7` → `Children[1]` [S1, unofficial].
    -   If all 8 nodes are leaves, create a `DualCell` object, assign the 8 nodes to fields `n0` through `n7`, and yield the cell [S1, unofficial].

2.  **TraverseFace(n0, n1, axis)**:
    -   This function traverses the dual cells on the face shared between two nodes `n0` (min side) and `n1` (max side) perpendicular to `axis` (0=X, 1=Y, 2=Z) [S1, unofficial].
    -   If both `n0` and `n1` are leaves, yield break [S1, unofficial].
    -   Identify 4 child nodes on the 'min' side (`cA0`-`cA3`) and 4 on the 'max' side (`cB0`-`cB3`) touching the face [S1, unofficial]. If a parent has no children, all its corresponding child pointers are set to the parent itself [S1, unofficial].
    -   Map child indices for `n0` based on `axis`: Axis 0 (X): `cA0=Children[1]`, `cA1=Children[2]`, `cA2=Children[6]`, `cA3=Children[5]`; Axis 1 (Y): `cA0=Children[4]`, `cA1=Children[5]`, `cA2=Children[6]`, `cA3=Children[7]`; Axis 2 (Z): `cA0=Children[3]`, `cA1=Children[2]`, `cA2=Children[6]`, `cA3=Children[7]` [S1, unofficial].
    -   Map child indices for `n1` based on `axis`: Axis 0 (X): `cB0=Children[0]`, `cB1=Children[4]`, `cB2=Children[7]`, `cB3=Children[3]`; Axis 1 (Y): `cB0=Children[0]`, `cB1=Children[1]`, `cB2=Children[2]`, `cB3=Children[3]`; Axis 2 (Z): `cB0=Children[0]`, `cB1=Children[1]`, `cB2=Children[5]`, `cB3=Children[4]` [S1, unofficial].
    -   Recursively traverse the 4 sub-faces formed by the child pairs: `(cA0,cB0)`, `(cA1,cB1)`, `(cA2,cB2)`, `(cA3,cB3)` [S1, unofficial].
    -   Traverse the 4 internal edges and 1 internal vertex within the face, depending on `axis` [S1, unofficial]:
        -   Axis 0 (X): Edges `(cA0,cB0,cB3,cA3,1)`, `(cA1,cB1,cB2,cA2,1)`, `(cA0,cB0,cB1,cA1,2)`, `(cA3,cB3,cB2,cA2,2)`; Vertex `(cA0,cB0,cB3,cA3,cA1,cB1,cB2,cA2)`.
        -   Axis 1 (Y): Edges `(cA0,cA3,cB3,cB0,0)`, `(cA1,cA2,cB2,cB1,0)`, `(cA0,cA1,cB1,cB0,2)`, `(cA3,cA2,cB2,cB3,2)`; Vertex `(cA0,cA1,cA2,cA3,cB0,cB1,cB2,cB3)`.
        -   Axis 2 (Z): Edges `(cA0,cB0,cB3,cA3,0)`, `(cA1,cB1,cB2,cA2,0)`, `(cA0,cA1,cB1,cB0,1)`, `(cA3,cA2,cB2,cB3,1)`; Vertex `(cA0,cA1,cB1,cB0,cA3,cA2,cB2,cB3)` [S1, unofficial].

3.  **TraverseEdge(n00, n10, n11, n01, axis)**:
    -   This function traverses the dual cells on the edge shared by four nodes `n00`, `n10`, `n11`, `n01` along `axis` [S1, unofficial]. The parameters are named based on their relative position: `n00` touches faces with `n01` and `n10` at the min corner, while `n11` is at the max corner [S1, unofficial].
    -   If all four nodes are leaves, yield break [S1, unofficial].
    -   Gather 8 child nodes (`cA0, cA1` for `n00`; `cB0, cB1` for `n10`; `cC0, cC1` for `n11`; `cD0, cD1` for `n01`) [S1, unofficial]. If a parent has no children, both its child pointers are set to the parent [S1, unofficial].
    -   Map child indices for `n00` (A) based on `axis`: Axis 0 (X): `cA0=Children[7]`, `cA1=Children[6]`; Axis 1 (Y): `cA0=Children[2]`, `cA1=Children[6]`; Axis 2 (Z): `cA0=Children[5]`, `cA1=Children[6]` [S1, unofficial].
    -   Map child indices for `n10` (B) based on `axis`: Axis 0 (X): `cB0=Children[4]`, `cB1=Children[5]`; Axis 1 (Y): `cB0=Children[3]`, `cB1=Children[7]`; Axis 2 (Z): `cB0=Children[4]`, `cB1=Children[7]` [S1, unofficial].
    -   Map child indices for `n11` (C) based on `axis`: Axis 0 (X): `cC0=Children[0]`, `cC1=Children[1]`; Axis 1 (Y): `cC0=Children[0]`, `cC1=Children[4]`; Axis 2 (Z): `cC0=Children[0]`, `cC1=Children[3]` [S1, unofficial].
    -   Map child indices for `n01` (D) based on `axis`: Axis 0 (X): `cD0=Children[3]`, `cD1=Children[2]`; Axis 1 (Y): `cD0=Children[1]`, `cD1=Children[5]`; Axis 2 (Z): `cD0=Children[1]`, `cD1=Children[2]` [S1, unofficial].
    -   Recursively traverse two child edges: `(cA0, cB0, cC0, cD0)` and `(cA1, cB1, cC1, cD1)` [S1, unofficial].
    -   Traverse the central vertex of the edge [S1, unofficial]:
        -   Axis 0 (X): Vertex `(cA0, cA1, cB1, cB0, cD0, cD1, cC1, cC0)`
        -   Axis 1 (Y): Vertex `(cA0, cB0, cC0, cD0, cA1, cB1, cC1, cD1)`
        -   Axis 2 (Z): Vertex `(cA0, cB0, cB1, cA1, cD0, cC0, cC1, cD1)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class DualCell:
    def __init__(self, n0, n1, n2, n3, n4, n5, n6, n7):
        self.n0, self.n1, self.n2, self.n3 = n0, n1, n2, n3
        self.n4, self.n5, self.n6, self.n7 = n4, n5, n6, n7

def traverse_vertex(n0, n1, n2, n3, n4, n5, n6, n7):
    if any(node.children is not None for node in [n0, n1, n2, n3, n4, n5, n6, n7]):
        yield from traverse_vertex(
            n0.children[6], n1.children[7], n2.children[4], n3.children[5],
            n4.children[2], n5.children[3], n6.children[0], n7.children[1]
        )
    else:
        yield DualCell(n0, n1, n2, n3, n4, n5, n6, n7)

def traverse_face(n0, n1, axis):
    if n0.children is None and n1.children is None:
        return

    if n0.children is None:
        cA0 = cA1 = cA2 = cA3 = n0
    else:
        if axis == 0:
            cA0, cA1, cA2, cA3 = n0.children[1], n0.children[2], n0.children[6], n0.children[5]
        elif axis == 1:
            cA0, cA1, cA2, cA3 = n0.children[4], n0.children[5], n0.children[6], n0.children[7]
        else:
            cA0, cA1, cA2, cA3 = n0.children[3], n0.children[2], n0.children[6], n0.children[7]

    if n1.children is None:
        cB0 = cB1 = cB2 = cB3 = n1
    else:
        if axis == 0:
            cB0, cB1, cB2, cB3 = n1.children[0], n1.children[4], n1.children[7], n1.children[3]
        elif axis == 1:
            cB0, cB1, cB2, cB3 = n1.children[0], n1.children[1], n1.children[2], n1.children[3]
        else:
            cB0, cB1, cB2, cB3 = n1.children[0], n1.children[1], n1.children[5], n1.children[4]

    yield from traverse_face(cA0, cB0, axis)
    yield from traverse_face(cA1, cB1, axis)
    yield from traverse_face(cA2, cB2, axis)
    yield from traverse_face(cA3, cB3, axis)

    if axis == 0:
        yield from traverse_edge(cA0, cB0, cB3, cA3, 1)
        yield from traverse_edge(cA1, cB1, cB2, cA2, 1)
        yield from traverse_edge(cA0, cB0, cB1, cA1, 2)
        yield from traverse_edge(cA3, cB3, cB2, cA2, 2)
        yield from traverse_vertex(cA0, cB0, cB3, cA3, cA1, cB1, cB2, cA2)
    elif axis == 1:
        yield from traverse_edge(cA0, cA3, cB3, cB0, 0)
        yield from traverse_edge(cA1, cA2, cB2, cB1, 0)
        yield from traverse_edge(cA0, cA1, cB1, cB0, 2)
        yield from traverse_edge(cA3, cA2, cB2, cB3, 2)
        yield from traverse_vertex(cA0, cA1, cA2, cA3, cB0, cB1, cB2, cB3)
    else:
        yield from traverse_edge(cA0, cB0, cB3, cA3, 0)
        yield from traverse_edge(cA1, cB1, cB2, cA2, 0)
        yield from traverse_edge(cA0, cA1, cB1, cB0, 1)
        yield from traverse_edge(cA3, cA2, cB2, cB3, 1)
        yield from traverse_vertex(cA0, cA1, cB1, cB0, cA3, cA2, cB2, cB3)

def traverse_edge(n00, n10, n11, n01, axis):
    if all(node.children is None for node in [n00, n10, n11, n01]):
        return

    if n00.children is None:
        cA0 = cA1 = n00
    else:
        if axis == 0:
            cA0, cA1 = n00.children[7], n00.children[6]
        elif axis == 1:
            cA0, cA1 = n00.children[2], n00.children[6]
        else:
            cA0, cA1 = n00.children[5], n00.children[6]

    if n10.children is None:
        cB0 = cB1 = n10
    else:
        if axis == 0:
            cB0, cB1 = n10.children[4], n10.children[5]
        elif axis == 1:
            cB0, cB1 = n10.children[3], n10.children[7]
        else:
            cB0, cB1 = n10.children[4], n10.children[7]

    if n11.children is None:
        cC0 = cC1 = n11
    else:
        if axis == 0:
            cC0, cC1 = n11.children[0], n11.children[1]
        elif axis == 1:
            cC0, cC1 = n11.children[0], n11.children[4]
        else:
            cC0, cC1 = n11.children[0], n11.children[3]

    if n01.children is None:
        cD0 = cD1 = n01
    else:
        if axis == 0:
            cD0, cD1 = n01.children[3], n01.children[2]
        elif axis == 1:
            cD0, cD1 = n01.children[1], n01.children[5]
        else:
            cD0, cD1 = n01.children[1], n01.children[2]

    yield from traverse_edge(cA0, cB0, cC0, cD0, axis)
    yield from traverse_edge(cA1, cB1, cC1, cD1, axis)

    if axis == 0:
        yield from traverse_vertex(cA0, cA1, cB1, cB0, cD0, cD1, cC1, cC0)
    elif axis == 1:
        yield from traverse_vertex(cA0, cB0, cC0, cD0, cA1, cB1, cC1, cD1)
    else:
        yield from traverse_vertex(cA0, cB0, cB1, cA1, cD0, cC0, cC1, cD1)

def generate_dual_grid(root_node):
    if root_node.children is None:
        return
    
    yield from traverse_face(root_node.children[0], root_node.children[1], 0)
    yield from traverse_face(root_node.children[3], root_node.children[2], 0)
    yield from traverse_face(root_node.children[4], root_node.children[5], 0)
    yield from traverse_face(root_node.children[7], root_node.children[6], 0)
    
    yield from traverse_face(root_node.children[0], root_node.children[4], 1)
    yield from traverse_face(root_node.children[1], root_node.children[5], 1)
    yield from traverse_face(root_node.children[3], root_node.children[7], 1)
    yield from traverse_face(root_node.children[2], root_node.children[6], 1)
    
    yield from traverse_face(root_node.children[0], root_node.children[3], 2)
    yield from traverse_face(root_node.children[1], root_node.children[2], 2)
    yield from traverse_face(root_node.children[4], root_node.children[7], 2)
    yield from traverse_face(root_node.children[5], root_node.children[6], 2)
    
    yield from traverse_edge(root_node.children[0], root_node.children[3], root_node.children[7], root_node.children[4], 0)
    yield from traverse_edge(root_node.children[1], root_node.children[2], root_node.children[6], root_node.children[5], 0)
    
    yield from traverse_edge(root_node.children[0], root_node.children[1], root_node.children[2], root_node.children[3], 1)
    yield from traverse_edge(root_node.children[4], root_node.children[5], root_node.children[6], root_node.children[7], 1)
    
    yield from traverse_edge(root_node.children[3], root_node.children[2], root_node.children[6], root_node.children[7], 2)
    yield from traverse_edge(root_node.children[0], root_node.children[1], root_node.children[5], root_node.children[4], 2)
    
    yield from traverse_vertex(
        root_node.children[0], root_node.children[1], root_node.children[2], root_node.children[3],
        root_node.children[4], root_node.children[5], root_node.children[6], root_node.children[7]
    )
```

**Edge Cases and Pitfalls:**
-   **Duplicate References:** Large nodes adjacent to smaller nodes cause duplicate references in `DualCell`. The algorithm handles this by allowing duplicates in the 8-node array [S1, unofficial].
-   **Ordering Errors:** In `TraverseFace` for the Shared Z edge, the Y order of child nodes is flipped compared to some reference articles because the reference was deemed incorrect [S1, unofficial]. Implementers must use the specific index mappings provided above.
-   **Leaf Nodes:** If a node is a leaf, its children are `null`. The algorithm explicitly checks for this and uses the parent node itself as a stand-in for missing children to ensure the dual grid remains connected across level boundaries [S1, unofficial].

## Subtopics

### [Octree Connectivity Algorithm](5035-octree-connectivity-algorithm.md)
The Octree Connectivity Algorithm connects dual vertices generated by the Dual Grid Traversal for Adaptive Octree into a manifold triangle mesh. It solves the problem of ensuring that every intercepted edge of the isosurface is processed exactly once, preventing duplicate geometry and ensuring topological consistency in adaptive octrees where adjacent nodes have varying depths. Unlike sibling methods that may process cells independently, this algorithm uses a global marking system on edges and cells to coordinate connectivity across different levels of refinement.

### Dual Grid Construction
Dual Grid Construction constructs a dual grid where each vertex corresponds to a point in the octree, represented as a set of 2-8 surrounding octree nodes to handle adaptive boundaries [S1, unofficial]. It traverses nodes recursively; if a node is a leaf, it yields break, otherwise it recursively traverses children and processes shared X, Y, and Z faces and edges between child quartets, followed by the central vertex [S1, unofficial]. The `DualCell` layout contains 8 `OctreeNode` references ordered identically to octree children, with a `Vertex` property returning `n0.Bounds.max` [S1, unofficial]. Large nodes adjacent to smaller nodes cause duplicate references in `DualCell`, which the algorithm handles by allowing duplicates in the 8-node array [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The time complexity is proportional to the number of dual cells generated, which depends on the number of boundaries between octree nodes of different depths [S1, unofficial]. In the worst case (uniform refinement), it is linear with respect to the number of nodes [G]. The memory complexity is also linear with respect to the output size, as each dual cell is yielded and processed [G].

**Measured Performance:**
Not given in the sources.

**Scaling:**
The algorithm scales well with adaptive refinement because it only traverses the boundaries where refinement changes [S1, unofficial]. It avoids processing the interior of uniform regions [G].

**Strengths and Limitations:**
-   **Strengths:** Handles arbitrary adaptive octree structures without requiring a uniform grid [S1, unofficial]. Ensures topological consistency for mesh generation [S1, unofficial].
-   **Limitations:** The recursive nature may lead to deep call stacks for very deep octrees [G]. The complexity of the index mappings makes implementation error-prone [G].

**When to use it and when not:**
Use this method when generating isosurfaces from adaptive octree data structures where topological consistency is critical [S1, unofficial]. Do not use it if a uniform grid is sufficient, as simpler traversal methods would be more efficient [G].

## Background
This method originates from the implementation of Dual Marching Cubes for adaptive octrees, specifically as described in the unofficial source [S1, unofficial]. It is designed to bridge the gap between adaptive volumetric data and polygonal mesh generation, addressing the challenges of connectivity at level boundaries [S1, unofficial].

## Sources
[S1] heyx3/DualMarchingCubes, https://github.com/heyx3/DualMarchingCubes, fetched 2026-10-27 (unofficial)
