# Coarse Mesh Traversal with Dual-Ray Culling

Also called: Front/back face culling traversal, Two-ray coarse mesh traversal

## Overview
A traversal algorithm for a coarse tetrahedral mesh that uses two rays (one with front-face culling, one with back-face culling) to determine the ray's position relative to the coarse mesh elements, handling non-convex shapes and holes [S1, unofficial]. It builds on [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md) by explicitly resolving the entry and exit boundaries of the current coarse element using dual-ray intersection tests, enabling robust nested traversal into the fine structure [S1, unofficial]. Unlike single-pass methods that compute intersections in the fragment shader, this approach uses two distinct ray traces per coarse element to establish precise $t$-bounds for the inner loop [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a BVH constructed from the coarse tetrahedral mesh faces [S1, unofficial].
*   **Output:** A tuple `(coarseID, tBack, tNextFace)` where `coarseID` is the identifier of the current coarse tetrahedron, `tBack` is the exit distance of the ray from the current coarse element, and `tNextFace` is the entry distance for the next coarse element [S1, unofficial]. If the ray is outside the mesh, it returns `(-1, 0, 0)` [S1, unofficial].

**Data Structures:**
*   **BVH:** Built from the coarse mesh elements. Each leaf node represents a face of a coarse tetrahedron. The BVH stores the coarse tetrahedron ID associated with each face [S1, unofficial].
*   **Ray Record:** Contains the hit distance $t$, the hit point, and the `coarseTetID` of the intersected element [S1, unofficial].

**Formulas:**
*   $t_{Back}$: Hit distance from the front-face cull ray, representing the exit boundary of the current coarse element [S1, unofficial].
*   $t_{Front}$: Hit distance from the back-face cull ray, representing the entry boundary of the next coarse element or the exit of the current one [S1, unofficial].
*   $t_{NextFace} = \text{nextafterf}(t, \infty)$: Advances $t$ to the next representable floating-point number to prevent re-intersection with the same geometry [S1, unofficial].

**Algorithm Steps:**
1.  **Front-Face Trace:** Trace a ray with front-face culling enabled (ignoring back-facing triangles) [S1, unofficial].
    *   If the ray misses, return `(-1, 0, 0)` [S1, unofficial].
    *   If it hits, store the hit distance as $t_{Back}$ and the associated tetrahedron ID as `coarseID` [S1, unofficial].
2.  **Back-Face Trace:** Trace the same ray with back-face culling enabled (ignoring front-facing triangles) [S1, unofficial].
    *   If the ray misses, the ray is inside the last element of the mesh. Set $t_{NextFace} = \text{nextafterf}(t_{Back}, \infty)$ [S1, unofficial].
    *   If the ray hits, let the hit distance be $t_{Front}$ and the tetrahedron ID be `hitID` [S1, unofficial].
3.  **Boundary Resolution:**
    *   If `hitID` equals `coarseID` (the ray enters and exits the same coarse element):
        *   Set $t_{Front} = \text{nextafterf}(t_{Front}, \infty)$ [S1, unofficial].
        *   Set $t_{NextFace} = \text{nextafterf}(t_{Back}, \infty)$ [S1, unofficial].
    *   If `hitID` differs from `coarseID` (the ray enters a new coarse element):
        *   Set $t_{NextFace} = \text{nextafterf}(t_{Front}, \infty)$ [S1, unofficial].
4.  **Return:** Return `(coarseID, tBack, tNextFace)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def nextafterf(t, direction):
    """Return the next representable floating-point number after t."""
    if direction == float('inf'):
        return math.nextafter(t, float('inf'))
    return math.nextafter(t, float('-inf'))

def trace_ray(ray, cull_mode):
    """Perform BVH traversal for the given ray, respecting face culling mode.
    Returns a record with attributes: t (hit distance), coarse_tet_id, missed (bool)."""
    # Standard BVH traversal logic would go here.
    pass

def traverse_coarse_mesh(ray):
    # Step 1: Trace with front-face culling
    ray_record = trace_ray(ray, cull_mode='CULL_FRONT_FACING_TRIANGLES')

    if ray_record.missed:
        return -1, 0, 0

    t_back = ray_record.t
    coarse_id = ray_record.coarse_tet_id

    # Step 2: Trace with back-face culling
    ray_record = trace_ray(ray, cull_mode='CULL_BACK_FACING_TRIANGLES')

    if not ray_record.missed:
        t_front = ray_record.t
        hit_id = ray_record.coarse_tet_id

        if coarse_id == hit_id:
            # Ray enters and exits the same coarse element
            t_front = nextafterf(t_front, float('inf'))
            t_next_face = nextafterf(t_back, float('inf'))
        else:
            # Ray exits current element and enters a new one
            t_next_face = nextafterf(t_front, float('inf'))
    else:
        # Ray is inside the last element of the mesh
        t_next_face = nextafterf(t_back, float('inf'))

    return coarse_id, t_back, t_next_face
```

**Parameters:**
*   `nextafterf(t, direction)`: A function that returns the next representable floating-point number after `t` in the specified direction (usually `INFINITY` for advancing forward) [S1, unofficial]. This prevents re-intersection with the same geometry in subsequent iterations [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision Issues:** When exiting faces, standard floating-point arithmetic may cause the ray to re-intersect the same geometry in the next step. Using `nextafterf` advances $t$ slightly to avoid this [S1, unofficial].
*   **Non-Convex Shapes:** The dual-ray approach correctly handles non-convex shapes and holes by distinguishing between entry and exit faces based on face normal orientation relative to the ray [S1, unofficial].

## Subtopics

### Coarse Mesh Macrocell Optimization
A technique where coarse mesh elements are treated as macrocells storing pre-computed minimum and maximum density values of their child elements [S1, unofficial]. This enables empty space skipping and tighter majorant estimation for Woodcock tracking [S1, unofficial].
*   **Steps:** Calculate macrocell densities by applying the transfer function to scalar values of child elements. Recalculate these values using a CUDA kernel whenever the transfer function changes [S1, unofficial]. Before processing a coarse element, check if its maximum macrocell density is zero. If so, skip the element and move to the next [S1, unofficial].
*   **Comparison:** Macrocells allow for empty space skipping and tighter majorant extinction, reducing sample rejection rates compared to not using macrocells [S1, unofficial].

### View-dependent Early Tree Traversal Return
A technique to early-return from refinement tree traversal based on the normalized screen space area of the tetrahedron [S1, unofficial].
*   **Steps:** Calculate the total normalized screen space area of a tetrahedron by summing the areas of the four triangles created by its four vertices [S1, unofficial].
*   **Parameters:** Refinement criteria: A threshold for normalized screen space size. It can be converted to true screen space area by multiplying by total screen space [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Requires two BVH traversals per coarse element [S1, unofficial]. Not given in the sources.
*   **Memory Complexity:** Requires storing the BVH and macrocell statistics (min/max density) for each coarse element [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Robustly handles non-convex geometries and holes. Macrocell optimization significantly reduces sample rejection rates in Woodcock tracking [S1, unofficial].
*   **Limitations:** Requires two ray traces per coarse element, doubling the traversal overhead compared to single-ray methods. Macrocell values must be recomputed if the transfer function changes [S1, unofficial].
*   **When to use:** When rendering non-convex volumes or volumes with holes where simple bounding box culling is insufficient [S1, unofficial].

## Background
Originated in the context of direct volume rendering of tree-based tetrahedral adaptive mesh refinement (AMR) data [S1, unofficial]. The method was developed to efficiently traverse coarse tetrahedral meshes while supporting nested refinement trees [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
