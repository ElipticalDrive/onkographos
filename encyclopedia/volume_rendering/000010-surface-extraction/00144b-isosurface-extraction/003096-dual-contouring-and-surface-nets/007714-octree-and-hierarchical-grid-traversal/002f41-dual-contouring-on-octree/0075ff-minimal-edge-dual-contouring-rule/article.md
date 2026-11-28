# Minimal Edge Dual Contouring Rule

Also called: Dual contouring signed octrees, Minimal Edge Polygon Generation, Polygon Generation for Simplified Octrees

## Overview
The Minimal Edge Dual Contouring Rule is a polygon generation method for extracting isosurfaces from simplified signed octrees that eliminates the need for extra subdivision and crack repair [S1, unofficial]. It solves the problem of topological inconsistencies (cracks) in dual contouring by restricting polygon generation to "minimal edges"—leaf cube edges that do not properly contain an edge of a neighboring leaf [S1, unofficial]. Unlike the Perry/Frisken rule, which enumerates edges and then searches for containing cubes via neighbor walking, this method uses recursive functions to directly enumerate the sets of cubes containing a common minimal edge [S1, unofficial].

## How it works
**Inputs:** A simplified signed octree where each leaf node stores a scalar value (sign) and a minimizing vertex position.
**Outputs:** A closed polygonal mesh (triangle and quad indices).

**Data Structures:**
*   **Octree Node:** Contains a sign (boolean or integer), a minimizing vertex (3D vector), and children pointers (for non-leaves).
*   **Minimal Edge:** An edge of a leaf cube.

**Definitions:**
*   **Minimal Edge:** An edge of a leaf cube that does not properly contain an edge of a neighboring leaf [S1, unofficial].
*   **Sign Change:** An edge exhibits a sign change if the scalar values at its endpoints have different signs [S1, unofficial].

**Algorithm Steps:**
1.  Enumerate all minimal edges in the octree using recursive traversal functions `faceProc` and `edgeProc` [S1, unofficial].
2.  For each minimal edge identified:
    a. Check if the edge exhibits a sign change [S1, unofficial].
    b. If a sign change exists, generate a polygon connecting the minimizing vertices of all leaf cubes that contain this edge [S1, unofficial].

**Pseudocode:**
The enumeration of cubes containing a common minimal edge is performed via two recursive functions. The following Python-like pseudocode implements the enumeration logic for a quadtree (noted as extendable to octrees) [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def faceProc(q):
    """
    Enumerates pairs of leaf squares containing a common edge on the boundary of node q.
    q is an interior node in the quadtree.
    """
    if q is a leaf:
        return
    
    # q has 4 children: q0, q1, q2, q3 (ordered spatially)
    # Process internal edges of q's boundary
    # Edge between q0 and q1
    if q0 is leaf and q1 is leaf:
        # q0 and q1 are face-adjacent leaves sharing an edge
        # This edge is a candidate minimal edge if it meets the definition
        # Generate polygon for this edge using minimizing vertices of q0 and q1
        generate_polygon([q0.min_vertex, q1.min_vertex], edge_between(q0, q1))
    elif q0 is leaf and q1 is not leaf:
        # q1 is interior, recurse into q1 to find leaves on its boundary facing q0
        edgeProc(q0, q1)
    elif q0 is not leaf and q1 is leaf:
        edgeProc(q0, q1)
    else:
        # Both interior, recurse both
        edgeProc(q0, q1)

    # Repeat for other adjacent pairs (q1-q2, q2-q3, q3-q0)
    # ...

def edgeProc(q1, q2):
    """
    Enumerates cubes containing a common minimal edge between interior node q1 and neighbor q2.
    """
    if q1 is leaf and q2 is leaf:
        # Both are leaves sharing an edge
        # Check if this edge is minimal (does not properly contain neighbor's edge)
        # If minimal and sign change, generate polygon
        if is_minimal_edge(q1, q2) and has_sign_change(q1, q2):
            generate_polygon([q1.min_vertex, q2.min_vertex], edge_between(q1, q2))
    elif q1 is leaf and q2 is not leaf:
        # q2 is interior, recurse into q2's children that are adjacent to q1
        for child in q2.children_adjacent_to(q1):
            edgeProc(q1, child)
    elif q1 is not leaf and q2 is leaf:
        for child in q1.children_adjacent_to(q2):
            edgeProc(child, q2)
    else:
        # Both interior, recurse into adjacent children
        for c1 in q1.children:
            for c2 in q2.children:
                if c1 and c2 are adjacent:
                    edgeProc(c1, c2)

def is_minimal_edge(leaf1, leaf2):
    """
    Returns True if the shared edge between leaf1 and leaf2 is a minimal edge.
    An edge is minimal if it does not properly contain an edge of a neighboring leaf.
    """
    # Implementation detail: Check if the shared edge is strictly shorter than
    # any potential edge from a coarser neighbor that would contain it.
    # In a simplified octree, this typically means checking if the parent of the
    # coarser side also contains the edge.
    # [G] Specific implementation of "properly contain" depends on octree structure.
    # Generally, if the edge is on the boundary of the finest level, it is minimal.
    return True # Placeholder for specific geometric check [G]

def has_sign_change(leaf1, leaf2):
    return leaf1.sign != leaf2.sign

def generate_polygon(vertices, edge):
    """
    Generates a polygon (triangle or quad) from the minimizing vertices.
    """
    if len(vertices) == 2:
        # In uniform case, usually 2 cubes share an edge -> Quad
        # However, transitional areas may involve 3 cubes -> Triangles
        pass
    elif len(vertices) == 3:
        # Transitional area: 3 cubes share an edge -> Triangles
        pass
```

**Edge Cases and Pitfalls:**
*   **Transitional Areas:** In areas where a single coarse cube is face-adjacent to four fine cubes, minimal edges in the middle of the shared coarse face are contained by only three cubes. This generates triangles instead of quads to form a transition between coarse quads and fine quads [S1, unofficial].
*   **Closed Mesh Property:** The rule always produces a closed polygonal mesh where every edge is contained by an even number of polygons [S1, unofficial]. This is because edges in the dual contour are generated by pairs of face-adjacent leaf cubes, and the minimal edges tiling the boundary of their common square face always exhibit an even number of sign changes [S1, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The method avoids the overhead of neighbor walking or explicit link maintenance required by the Perry/Frisken rule [S1, unofficial].
*   **Strengths:** Eliminates the need for extra subdivision to repair cracks [S1, unofficial]. Produces a closed mesh [S1, unofficial].
*   **Limitations:** Generates triangles in transitional areas between coarse and fine levels, which may not be ideal for all applications requiring pure quad meshes [S1, unofficial].

## Background
The Minimal Edge Dual Contouring Rule was proposed as a simpler alternative to existing methods for dual contouring signed octrees that required extra subdivision to avoid cracks [S1, unofficial]. It is based on the observation that only edges of leaf cubes that do not properly contain an edge of a neighboring leaf should generate a polygon [S1, unofficial]. The method was presented in a paper from Rice University [S1, unofficial].

## Sources
[S1] Dual Contouring of Hermite Data, https://www.cs.rice.edu/~jwarren/papers/dualcontour.pdf, fetched 2026-10-27 (unofficial)
