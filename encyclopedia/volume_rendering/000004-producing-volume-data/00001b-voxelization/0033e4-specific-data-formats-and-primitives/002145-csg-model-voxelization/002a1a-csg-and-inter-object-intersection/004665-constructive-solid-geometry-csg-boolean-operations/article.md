# Constructive Solid Geometry (CSG) Boolean Operations

Also called: BSP Clipping, Mesh Boolean Operations

## Overview
Computes boolean combinations (union, subtract, intersect, xor, stencil) of two polygonal meshes by partitioning space and clipping polygon sets against each other's BSP trees [G]. Unlike voxel-based methods such as [Solid Voxel CSG](7222-csg-and-inter-object-intersection-solid-voxel-csg.md), this approach operates directly on polygonal meshes, avoiding grid quantization but requiring robust plane tests [G].

## How it works
**Inputs:** Two polygonal meshes, $M_A$ and $M_B$, defined by sets of polygons [G].
**Outputs:** A result polygonal mesh $M_R$ representing the boolean operation [G].

**Data Structures:**
*   **Polygon:** Defined by an ordered list of vertices and a unit normal vector.
*   **BSP Tree:** A binary tree where each node contains a splitting plane. Left children represent space "behind" or "less than" the plane; right children represent space "in front of" or "greater than" the plane [G].

**Steps:**

1.  **Bounding Volume Test:**
    Compute the bounding volumes (e.g., AABBs) for $M_A$ and $M_B$. If they do not intersect, return an empty mesh or handle as a degenerate case [G]. If they intersect, proceed.

2.  **Clip $M_A$ against $M_B$'s BSP:**
    a. Construct a BSP tree from the polygons of $M_B$.
    b. Clip the polygon set of $M_A$ against this BSP tree.
    c. Retain only polygons strictly greater than the BSP planes [G].
    d. Let the clipped set be $A_{clip}$.

3.  **Clip $M_B$ against $M_A$'s BSP:**
    a. Construct a BSP tree from the polygons of $M_A$.
    b. Clip the polygon set of $M_B$ against this BSP tree.
    c. Retain polygons greater than or equal to the BSP planes for Union, or less than for Subtraction [G].
    d. Let the clipped set be $B_{clip}$.

4.  **Handle Boundary Polygons:**
    During the splitting process in steps 2 and 3, polygons that intersect the splitting planes are split into boundary outputs ($A_{out}, B_{out}$) [G]. These boundary polygons represent the intersection silhouette and must be included in the final result [G].

5.  **Combine Results:**
    Concatenate the boundary outputs and the clipped polygons to form the result mesh $M_R = A_{out} \cup B_{out} \cup A_{clip} \cup B_{clip}$ [G].

6.  **Adjust Winding/Normals:**
    For subtraction ($M_A - M_B$), invert the winding/normal of the clipped polygons from the subtracted mesh ($B_{clip}$) to ensure consistent outward-facing normals [G].

**Pitfalls:**
*   **Non-intersecting bounds:** Must be handled explicitly to avoid unnecessary BSP tree construction and clipping overhead [G].
*   **Coplanar polygons:** During plane splitting, coplanar polygons must be assigned to the correct side based on the normal dot product to prevent degenerate geometry or incorrect boolean results [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Plane:
    def __init__(self, normal, distance):
        self.normal = normal  # unit vector (x, y, z)
        self.distance = distance  # distance from origin along normal

class BSPNode:
    def __init__(self, plane=None, front=None, back=None):
        self.plane = plane
        self.front = front
        self.back = back

class Polygon:
    def __init__(self, vertices, normal):
        self.vertices = vertices  # list of (x, y, z)
        self.normal = normal

class Mesh:
    def __init__(self, polygons):
        self.polygons = polygons
        self.bounds = (min_x, min_y, min_z, max_x, max_y, max_z)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def bounds_intersect(b1, b2):
    return not (b1[3] < b2[0] or b1[4] < b2[1] or b1[5] < b2[2] or
                b2[3] < b1[0] or b2[4] < b1[1] or b2[5] < b1[2])

def build_bsp_tree(polygons):
    # Sources do not give exact construction heuristic [G]
    if not polygons:
        return BSPNode()
    plane = Plane(normal=polygons[0].normal, distance=dot(polygons[0].normal, polygons[0].vertices[0]))
    front_polys, back_polys, boundary_polys = [], [], []
    for p in polygons[1:]:
        dist = dot(p.normal, p.vertices[0]) - plane.distance
        if dist > 0: front_polys.append(p)
        elif dist < 0: back_polys.append(p)
        else: boundary_polys.append(p)
    return BSPNode(plane, build_bsp_tree(front_polys), build_bsp_tree(back_polys))

def classify_polygon(poly, node):
    if not node.plane: return 'front'
    dists = [dot(poly.normal, v) - node.plane.distance for v in poly.vertices]
    if all(d > 0 for d in dists): return 'front'
    if all(d < 0 for d in dists): return 'back'
    return 'intersecting'

def split_polygon(poly, plane):
    # Sources do not give exact splitting algorithm [G]
    return [poly]

def clip_polygons(polygons, bsp, operation):
    clipped, boundary = [], []
    for poly in polygons:
        status = classify_polygon(poly, bsp)
        if status == 'front':
            if operation in ['union', 'subtract']:
                clipped.append(poly)
            elif operation == 'intersect':
                boundary.append(poly)
        elif status == 'back':
            if operation == 'intersect':
                clipped.append(poly)
            elif operation == 'union':
                boundary.append(poly)
        elif status == 'intersecting':
            p1, p2 = split_polygon(poly, bsp.plane)
            c1, b1 = clip_polygons(p1, bsp.front, operation)
            c2, b2 = clip_polygons(p2, bsp.back, operation)
            clipped.extend(c1 + c2)
            boundary.extend(b1 + b2)
    return clipped, boundary

def csg_boolean(mesh_a, mesh_b, operation):
    if not bounds_intersect(mesh_a.bounds, mesh_b.bounds):
        return Mesh([])

    bsp_b = build_bsp_tree(mesh_b.polygons)
    a_clipped, a_boundary = clip_polygons(mesh_a.polygons, bsp_b, operation)

    bsp_a = build_bsp_tree(mesh_a.polygons)
    b_clipped, b_boundary = clip_polygons(mesh_b.polygons, bsp_a, operation)

    result_polys = a_boundary + b_boundary + a_clipped + b_clipped

    if operation == 'subtract':
        for p in b_clipped:
            p.normal = (-p.normal[0], -p.normal[1], -p.normal[2])
            p.vertices.reverse()

    return Mesh(result_polys)
```

## Operating characteristics
*   **Time Complexity:** Depends on the complexity of the meshes and the balance of the BSP trees. In the worst case, clipping a polygon against a BSP tree is $O(N)$, and clipping all polygons is $O(N \cdot M)$ where $N, M$ are polygon counts. Construction of BSP trees is $O(N \log N)$ on average [G].
*   **Memory Complexity:** $O(N + M)$ to store the BSP trees and intermediate polygon sets [G].
*   **Strengths:** Produces watertight, manifold meshes with sharp features and no grid quantization artifacts [G]. Suitable for 3D printing and CAD applications where precision is critical [G].
*   **Limitations:** Requires robust geometric predicates and exact arithmetic to handle coplanar and nearly coplanar cases [G]. Can be slow for highly complex meshes with many polygons. Numerical instability can lead to non-manifold geometry or holes if not handled carefully [G].
*   **When to use:** When high-fidelity, watertight polygonal meshes are required from boolean operations, and the input meshes are not excessively complex [G].
*   **When not to use:** For real-time applications with very high polygon counts, or when approximate results are acceptable and voxel-based methods are preferred for simplicity [G].

## Background
The method is based on the principle of Constructive Solid Geometry (CSG) using BSP trees for mesh clipping [G]. It differs from voxel-based marching cubes by operating directly on polygonal meshes [G]. The approach is implemented in the Euclid Swift library for mesh boolean operations [S1, unofficial]. The author of the reference implementation notes the need for an easily understood and implemented solution without complicated maths or third-party libraries [S1, unofficial].

## Sources
[S1] andygeers/DualMarchingCuboids, https://github.com/andygeers/DualMarchingCuboids, fetched 2026-10-26 (unofficial)
