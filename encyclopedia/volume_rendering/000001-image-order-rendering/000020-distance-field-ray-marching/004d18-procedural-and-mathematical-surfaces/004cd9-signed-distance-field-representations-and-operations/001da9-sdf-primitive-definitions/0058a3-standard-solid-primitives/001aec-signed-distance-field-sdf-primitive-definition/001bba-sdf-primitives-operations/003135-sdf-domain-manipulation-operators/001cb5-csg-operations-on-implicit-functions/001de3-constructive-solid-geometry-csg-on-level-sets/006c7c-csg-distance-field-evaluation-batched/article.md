# CSG Distance Field Evaluation (Batched)

Also called: Scalar CSG Evaluation

## Overview
This method computes the signed distance and material indices for a batch of points against a Constructive Solid Geometry (CSG) tree [S1, unofficial]. It serves as the core evaluation kernel for ray marching or point-in-solid tests by processing multiple coordinates simultaneously [S1, unofficial]. Unlike the single-point variant, this batched approach maintains arrays of distances and material indices, allowing for efficient traversal of the CSG tree structure across a set of samples [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:**
    *   `pos`: Array of $N$ 3D points (vectors).
    *   `normaldist`: Optional array of $N$ floats used for normal approximation logic.
    *   `d`: Pointer to the root of the CSG tree.
    *   `box`: Axis-Aligned Bounding Box (AABB) defining the region of interest for the current evaluation batch.
    *   `num`: Integer $N$, the number of points in the batch.
*   **Outputs:**
    *   `dist`: Array of $N$ floats, storing the signed distance from each point to the CSG surface.
    *   `matindex`: Array of $N$ integers, storing the material index for each point.

**Data Structures:**
*   **CSG Node (`d`):** Contains a type (`C_UNION`, `C_INTERSECTION`, `C_DIFFERENCE`, `C_REPLACE`, `C_TRANSLATION`, `C_ROTATION`, `C_PLANE`, `C_CYLINDERXY`, `C_CYLINDERXZ`, `C_CYLINDERYZ`, `C_SPHERE`, `C_BOX`), pointers to children (`left`, `right`, `n`), and primitive-specific data (e.g., `p` for plane, `c` for cylinder, `s` for sphere, `b` for box, `t` for translation, `r` for rotation).
*   **AABB (`box`):** Defined by `pmin` and `pmax` vectors.
*   **Constants:** `MAT_AIR_INDEX` represents empty space. `FLT_MAX` represents infinity.

**Algorithm Steps:**

1.  **Initialization:**
    Initialize `dist[i] = FLT_MAX` and `matindex[i] = MAT_AIR_INDEX` for all $i \in [0, N-1]$ [S1, unofficial].

2.  **Recursive Evaluation (`distr`):**
    The function `distr` is called recursively on the CSG tree nodes. For each node type, the algorithm checks if the node's bounding box intersects the query `box`. If not, the branch is skipped [S1, unofficial].

    *   **Union (`C_UNION`):**
        1.  Check if `box` intersects `d->left->box`. If yes, recursively call `distr` on `d->left` [S1, unofficial].
        2.  Initialize temporary arrays `tempdist` to `FLT_MAX` and `tempmatindex` to `MAT_AIR_INDEX` [S1, unofficial].
        3.  Check if `box` intersects `d->right->box`. If yes, recursively call `distr` on `d->right`, writing results to `tempdist` and `tempmatindex` [S1, unofficial].
        4.  Update main arrays:
            *   `matindex[i] = max(matindex[i], tempmatindex[i])` [S1, unofficial].
            *   If `normaldist` is provided:
                *   If `abs(tempdist[i]) < normaldist[i]`, set `dist[i] = tempdist[i]`.
                *   Otherwise, keep `dist[i]` [S1, unofficial].
            *   If `normaldist` is not provided:
                *   `dist[i] = min(dist[i], tempdist[i])` [S1, unofficial].

    *   **Intersection (`C_INTERSECTION`):**
        1.  Check if `box` intersects `d->left->box`. If not, return [S1, unofficial].
        2.  Recursively call `distr` on `d->left` [S1, unofficial].
        3.  Initialize `tempdist` to `FLT_MAX` [S1, unofficial].
        4.  Check if `box` intersects `d->right->box`. If not, return [S1, unofficial].
        5.  Recursively call `distr` on `d->right`, writing to `tempdist` [S1, unofficial].
        6.  Update main arrays:
            *   `dist[i] = max(dist[i], tempdist[i])` [S1, unofficial].
            *   If `dist[i] >= 0`, set `matindex[i] = MAT_AIR_INDEX` [S1, unofficial].

    *   **Difference (`C_DIFFERENCE`):**
        1.  Check if `box` intersects `d->left->box`. If not, return [S1, unofficial].
        2.  Recursively call `distr` on `d->left` [S1, unofficial].
        3.  Check if `box` intersects `d->right->box`. If not, return [S1, unofficial].
        4.  Initialize `tempdist` to `FLT_MAX` and `tempmatindex` to default [S1, unofficial].
        5.  Recursively call `distr` on `d->right`, writing to `tempdist` and `tempmatindex` [S1, unofficial].
        6.  Update main arrays:
            *   `dist[i] = max(dist[i], -tempdist[i])` [S1, unofficial].
            *   If `dist[i] >= 0`, set `matindex[i] = MAT_AIR_INDEX` [S1, unofficial].

    *   **Replace (`C_REPLACE`):**
        1.  Check if `box` intersects `d->left->box`. If not, return [S1, unofficial].
        2.  Recursively call `distr` on `d->left` [S1, unofficial].
        3.  Check if `box` intersects `d->right->box`. If not, return [S1, unofficial].
        4.  Initialize `tempdist` and `tempmatindex` [S1, unofficial].
        5.  Recursively call `distr` on `d->right`, writing to `tempdist` and `tempmatindex` [S1, unofficial].
        6.  Update main arrays:
            *   If `tempdist[i] < 0` and `dist[i] < 0` (both inside), set `matindex[i] = tempmatindex[i]` [S1, unofficial].
            *   If `normaldist` is provided:
                *   If `dist[i] < 0` and `abs(tempdist[i]) < normaldist[i]`, set `dist[i] = tempdist[i]`.
                *   Otherwise, keep `dist[i]` [S1, unofficial].

    *   **Translation (`C_TRANSLATION`):**
        1.  Check if `box` intersects `n->box` (where `n` is the child node). If not, return [S1, unofficial].
        2.  Get translation vector `t->p` [S1, unofficial].
        3.  For each point $i$:
            *   `tpos[i] = pos[i] - t->p` [S1, unofficial].
            *   `tbox.pmin = box.pmin - t->p`
            *   `tbox.pmax = box.pmax - t->p` [S1, unofficial].
        4.  Recursively call `distr` on `t->n` with `tpos`, `normaldist`, `dist`, `matindex`, `num`, and `tbox` [S1, unofficial].

    *   **Rotation (`C_ROTATION`):**
        1.  Check if `box` intersects `n->box`. If not, return [S1, unofficial].
        2.  Get rotation quaternion `r->q` [S1, unofficial].
        3.  For each point $i$:
            *   `tpos[i] = xfmpoint(conj(r->q), pos[i])` [S1, unofficial].
        4.  Recursively call `distr` on `r->n` with `tpos`, `normaldist`, `dist`, `matindex`, `num`, and `aabb::all()` (unbounded box) [S1, unofficial].

    *   **Plane (`C_PLANE`):**
        1.  Check if `box` intersects `n->box`. If not, return [S1, unofficial].
        2.  For each point $i$:
            *   `dist[i] = dot(pos[i], p->p.xyz()) + p->p.w` [S1, unofficial].
            *   If `dist[i] < 0`, `matindex[i] = p->matindex` [S1, unofficial].

    *   **Cylinder (`C_CYLINDERXY`, `C_CYLINDERXZ`, `C_CYLINDERYZ`):**
        1.  Check if `box` intersects `n->box`. If not, return [S1, unofficial].
        2.  For each point $i$:
            *   Let `COORD` be the relevant 2D coordinates (xy, xz, or yz).
            *   `dist[i] = length(pos[i].COORD() - c->cCOORD) - c->r` [S1, unofficial].
            *   If `dist[i] < 0`, `matindex[i] = c->matindex` [S1, unofficial].

    *   **Sphere (`C_SPHERE`):**
        1.  Check if `box` intersects `n->box`. If not, return [S1, unofficial].
        2.  For each point $i$:
            *   `dist[i] = length(pos[i]) - s->r` [S1, unofficial].
            *   If `dist[i] < 0`, `matindex[i] = s->matindex` [S1, unofficial].

    *   **Box (`C_BOX`):**
        1.  Check if `box` intersects `n->box`. If not, return [S1, unofficial].
        2.  Get box extent `b->extent` [S1, unofficial].
        3.  For each point $i$:
            *   `pd = abs(pos[i]) - extent` [S1, unofficial].
            *   `dist[i] = min(max(pd.x, max(pd.y, pd.z)), 0.0f) + length(max(pd, vec3f(0)))` [S1, unofficial].
            *   If `dist[i] < 0`, `matindex[i] = b->matindex` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def distr(pos, normaldist, d, box, num, dist, matindex):
    """
    Recursive CSG evaluation for a batch of points.
    pos: list of vec3
    normaldist: list of float or None
    d: CSG node
    box: AABB {pmin: vec3, pmax: vec3}
    num: int
    dist: list of float (output)
    matindex: list of int (output)
    """
    # Check bounding box intersection
    if not aabb_intersects(d.box, box):
        return

    node_type = d.type

    if node_type == C_UNION:
        # Left child
        if aabb_intersects(d.left.box, box):
            distr(pos, normaldist, d.left, box, num, dist, matindex)
        
        # Right child
        tempdist = [FLT_MAX] * num
        tempmatindex = [MAT_AIR_INDEX] * num
        
        if aabb_intersects(d.right.box, box):
            distr(pos, normaldist, d.right, box, num, tempdist, tempmatindex)
        
        # Update
        for i in range(num):
            matindex[i] = max(matindex[i], tempmatindex[i])
            if normaldist is not None:
                if abs(tempdist[i]) < normaldist[i]:
                    dist[i] = tempdist[i]
                # else keep dist[i]
            else:
                dist[i] = min(dist[i], tempdist[i])

    elif node_type == C_INTERSECTION:
        if not aabb_intersects(d.left.box, box):
            return
        distr(pos, normaldist, d.left, box, num, dist, matindex)
        
        tempdist = [FLT_MAX] * num
        if not aabb_intersects(d.right.box, box):
            return
        distr(pos, normaldist, d.right, box, num, tempdist, [MAT_AIR_INDEX] * num)
        
        for i in range(num):
            dist[i] = max(dist[i], tempdist[i])
            if dist[i] >= 0.0:
                matindex[i] = MAT_AIR_INDEX

    elif node_type == C_DIFFERENCE:
        if not aabb_intersects(d.left.box, box):
            return
        distr(pos, normaldist, d.left, box, num, dist, matindex)
        
        if not aabb_intersects(d.right.box, box):
            return
        
        tempdist = [FLT_MAX] * num
        tempmatindex = [MAT_AIR_INDEX] * num
        distr(pos, normaldist, d.right, box, num, tempdist, tempmatindex)
        
        for i in range(num):
            dist[i] = max(dist[i], -tempdist[i])
            if dist[i] >= 0.0:
                matindex[i] = MAT_AIR_INDEX

    elif node_type == C_REPLACE:
        if not aabb_intersects(d.left.box, box):
            return
        distr(pos, normaldist, d.left, box, num, dist, matindex)
        
        if not aabb_intersects(d.right.box, box):
            return
            
        tempdist = [FLT_MAX] * num
        tempmatindex = [MAT_AIR_INDEX] * num
        distr(pos, normaldist, d.right, box, num, tempdist, tempmatindex)
        
        for i in range(num):
            if tempdist[i] < 0 and dist[i] < 0:
                matindex[i] = tempmatindex[i]
            if normaldist is not None:
                if dist[i] < 0 and abs(tempdist[i]) < normaldist[i]:
                    dist[i] = tempdist[i]
                # else keep dist[i]

    elif node_type == C_TRANSLATION:
        if not aabb_intersects(d.n.box, box):
            return
        t = d.t
        tbox = {
            'pmin': box.pmin - t.p,
            'pmax': box.pmax - t.p
        }
        tpos = [pos[i] - t.p for i in range(num)]
        distr(tpos, normaldist, d.n, tbox, num, dist, matindex)

    elif node_type == C_ROTATION:
        if not aabb_intersects(d.n.box, box):
            return
        r = d.r
        # Assuming xfmpoint and conj are defined elsewhere or standard
        tpos = [xfmpoint(conj(r.q), pos[i]) for i in range(num)]
        distr(tpos, normaldist, d.n, aabb_all(), num, dist, matindex)

    elif node_type == C_PLANE:
        if not aabb_intersects(d.n.box, box):
            return
        p = d.p
        for i in range(num):
            dist[i] = dot(pos[i], p.p.xyz()) + p.p.w
            if dist[i] < 0.0:
                matindex[i] = p.matindex

    elif node_type in [C_CYLINDERXY, C_CYLINDERXZ, C_CYLINDERYZ]:
        if not aabb_intersects(d.n.box, box):
            return
        c = d.c
        # Determine COORD based on type
        if node_type == C_CYLINDERXY:
            for i in range(num):
                v = vec3(pos[i].x, pos[i].y, 0)
                dist[i] = length(v - c.cCOORD) - c.r
                if dist[i] < 0.0:
                    matindex[i] = c.matindex
        elif node_type == C_CYLINDERXZ:
            for i in range(num):
                v = vec3(pos[i].x, 0, pos[i].z)
                dist[i] = length(v - c.cCOORD) - c.r
                if dist[i] < 0.0:
                    matindex[i] = c.matindex
        elif node_type == C_CYLINDERYZ:
            for i in range(num):
                v = vec3(0, pos[i].y, pos[i].z)
                dist[i] = length(v - c.cCOORD) - c.r
                if dist[i] < 0.0:
                    matindex[i] = c.matindex

    elif node_type == C_SPHERE:
        if not aabb_intersects(d.n.box, box):
            return
        s = d.s
        for i in range(num):
            dist[i] = length(pos[i]) - s.r
            if dist[i] < 0.0:
                matindex[i] = s.matindex

    elif node_type == C_BOX:
        if not aabb_intersects(d.n.box, box):
            return
        b = d.b
        for i in range(num):
            pd = vec3(
                abs(pos[i].x) - b.extent.x,
                abs(pos[i].y) - b.extent.y,
                abs(pos[i].z) - b.extent.z
            )
            max_pd = max(pd.x, max(pd.y, pd.z))
            dist[i] = min(max_pd, 0.0) + length(max(pd, vec3(0)))
            if dist[i] < 0.0:
                matindex[i] = b.matindex

def max(a, b):
    return a if a > b else b

def min(a, b):
    return a if a < b else b

def length(v):
    return math.sqrt(v.x**2 + v.y**2 + v.z**2)

def dot(v1, v2):
    return v1.x*v2.x + v1.y*v2.y + v1.z*v2.z

def vec3(x, y, z):
    return type('vec3', (), {'x': x, 'y': y, 'z': z})()

def abs(v):
    return vec3(abs(v.x), abs(v.y), abs(v.z))

def max_vec3(v, threshold):
    return vec3(max(v.x, threshold), max(v.y, threshold), max(v.z, threshold))

def aabb_intersects(box1, box2):
    # Simplified AABB intersection check
    return (box1.pmin.x <= box2.pmax.x and box1.pmax.x >= box2.pmin.x and
            box1.pmin.y <= box2.pmax.y and box1.pmax.y >= box2.pmin.y and
            box1.pmin.z <= box2.pmax.z and box1.pmax.z >= box2.pmin.z)

def aabb_all():
    return {'pmin': vec3(-FLT_MAX, -FLT_MAX, -FLT_MAX), 'pmax': vec3(FLT_MAX, FLT_MAX, FLT_MAX)}

def xfmpoint(q, p):
    # Placeholder for quaternion multiplication logic
    # In a real implementation, this rotates p by q
    return p

def conj(q):
    # Conjugate of quaternion
    return type('quat', (), {'x': -q.x, 'y': -q.y, 'z': -q.z, 'w': q.w})()
```

**Parameters:**
*   `MAT_AIR_INDEX`: Integer constant representing empty space [S1, unofficial].
*   `FLT_MAX`: Floating-point maximum value, used for initialization [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Signed distance calculations involving `min`, `max`, and `length` can suffer from floating-point precision issues near surfaces. The use of `normaldist` for Union/Replace operations helps mitigate normal estimation errors but requires careful handling of the `abs(tempdist[i]) < normaldist[i]` condition [S1, unofficial].
*   **Boundaries:** Bounding box intersection checks are critical for pruning. Incorrect intersection logic can lead to incorrect distance values or infinite recursion if not handled properly [S1, unofficial].
*   **Degenerate Input:** Empty batches or invalid CSG trees (e.g., null pointers) are not explicitly handled in the provided snippets, but standard guards should be applied [G].

## Subtopics

### [CSG Distance Field Evaluation (Single Point)](23375-csg-distance-field-evaluation-single-point.md)
A recursive algorithm to compute the signed distance from a single point to a Constructive Solid Geometry (CSG) tree, supporting Union, Intersection, Difference, Replace, Translation, Rotation, Plane, Cylinder, Sphere, and Box primitives. It differs from the batched variant by operating on a single coordinate vector and returning a scalar distance, serving as the fundamental evaluation kernel for ray marching or point-in-solid tests.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of points $N$ and the depth of the CSG tree. Each point is processed independently within the batch, but the tree traversal is shared.
*   **Memory Complexity:** $O(N)$ for the distance and material index arrays, plus $O(N)$ for temporary arrays in Union/Intersection/Difference operations.
*   **Performance:** The batched approach allows for potential SIMD optimization, although the provided implementation is described as non-SIMD (plain C) [S1, unofficial].
*   **Strengths:** Efficient for evaluating many points against complex CSG structures. Supports various primitive types and transformations.
*   **Limitations:** The non-SIMD nature may limit performance on modern hardware compared to vectorized implementations. The batched approach requires more memory than the single-point variant.
*   **When to use:** When evaluating a large number of points (e.g., for ray marching steps or voxelization).
*   **When not to use:** When memory is constrained or when evaluating a single point is sufficient.

## Background
The method is implemented in the `mini.q` project by bsegovia [S1, unofficial]. It is part of a broader effort to provide efficient CSG evaluation for rendering and geometric processing tasks. The specific implementation `distr` is a batched version of the CSG distance field evaluation, designed to handle multiple points simultaneously.

## Sources
[S1] bsegovia/mini.q, https://github.com/bsegovia/mini.q, fetched 2026-10-26 (unofficial)
