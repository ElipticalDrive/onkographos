# Bounding Volume Hierarchy (BVH) Traversal

Also called: BVH for Triangle Traversal, Bounding Volume Hierarchy, GPU BVH Traversal, Iterative BVH Traversal

## Overview
Bounding Volume Hierarchy (BVH) Traversal is a method for accelerating ray-primitive intersection tests by organizing scene geometry (typically triangles) into a hierarchical tree of bounding volumes [S1, unofficial]. It solves the problem of efficiently skipping empty space during ray traversal, replacing stride-based grid walks or linear primitive checks with a tree walk that prunes large portions of the scene [S1, unofficial]. It sets itself apart from [Hierarchical Grid Traversal](896-hierarchical-traversal.md) by abandoning the flat CSR grid layout in favor of a tree structure that adapts to the geometry's spatial distribution, and from [Leaf Collection](899-leaf-collection.md) by maintaining explicit parent-child relationships for traversal control rather than collecting leaf primitives into a flat buffer [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of triangles (or other primitives) defining the scene geometry.
*   **Output:** For each ray, the closest intersection point, the primitive ID, and optionally the normal or material data.

**Data Structures:**
The BVH is a binary tree where each node contains a bounding volume (e.g., AABB, OBB, or DOP) that encloses all primitives in its subtree [S5].
*   **Interior Node:** Contains the bounding volume of the node and indices/pointers to two child nodes (left and right) [S5].
*   **Leaf Node:** Contains the bounding volume of the leaf and a list of primitive indices or pointers to the primitive data [S5].
*   **Primitive Buffer:** A contiguous array storing the primitive data (e.g., triangle vertices) [S2, unofficial].

**Steps:**
1.  **Build:** Construct the BVH over the scene triangles. Common strategies include Surface Area Heuristic (SAH) based splitting [S1, unofficial].
2.  **Initialize:** For each ray, initialize an explicit stack in local memory (thread-local storage) [S2, unofficial]. Push the root node index onto the stack.
3.  **Traverse:** While the stack is not empty:
    a.  Pop the top node index from the stack.
    b.  Check if the ray intersects the node's bounding volume. If not, continue to the next iteration.
    c.  If the node is a leaf:
        i.  Perform intersection tests (e.g., Möller–Trumbore) for all primitives in the leaf.
        ii. Update the closest hit if a nearer intersection is found.
    d.  If the node is interior:
        i.  Determine the order of child traversal (e.g., based on ray entry distance to child bounding volumes).
        ii. Push the farther child's index onto the stack.
        iii. Push the nearer child's index onto the stack.
4.  **Terminate:** When the stack is empty, the closest hit found is the result for the ray.

**Formulas:**
*   **Ray-Box Intersection:** For an AABB defined by $min$ and $max$, the ray $O + tD$ intersects if:
    $$ t_{min} = \max_i \left( \frac{min_i - O_i}{D_i} \right), \quad t_{max} = \min_i \left( \frac{max_i - O_i}{D_i} \right) $$
    Intersection exists if $t_{min} \le t_{max}$ and $t_{max} > 0$ [S1, unofficial].
*   **Möller–Trumbore:** For a triangle with vertices $v_0, v_1, v_2$ and ray $O + tD$:
    $$ E_1 = v_1 - v_0, \quad E_2 = v_2 - v_0, \quad H = D \times E_2, \quad a = E_1 \cdot H, \quad f = 1/a $$
    $$ S = O - v_0, \quad u = f \cdot (S \cdot H), \quad T = D \times E_1, \quad v = f \cdot (S \cdot T) $$
    $$ t = f \cdot (E_2 \cdot T) $$
    Intersection if $a \ne 0$, $u \ge 0$, $v \ge 0$, and $u + v \le 1$ [S1, unofficial].

**Parameters:**
*   **Stack Size:** Must be sufficient to hold the maximum depth of the BVH. For a balanced tree with $N$ leaves, depth is $\approx \log_2 N$ [S2, unofficial].
*   **Primitive Count per Leaf:** Affects the trade-off between traversal depth and leaf intersection cost.

**Edge Cases and Pitfalls:**
*   **Recursion:** Recursive traversal is inefficient on GPUs due to call stack overhead and risk of stack overflows [S2, unofficial]. An explicit stack is required.
*   **Precision:** Floating-point precision issues in ray-box or ray-triangle tests can lead to missed intersections or artifacts [S1, unofficial].
*   **Degenerate Primitives:** Triangles with zero area should be handled or filtered during build/traversal.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bvh_traverse(ray_origin, ray_direction, bvh_root, primitives):
    # Stack stores node indices
    stack = [bvh_root]
    closest_t = infinity
    closest_prim_id = -1

    while stack:
        node_idx = stack.pop()
        node = bvh_root[node_idx]

        # Ray-box intersection test
        if not ray_box_intersect(ray_origin, ray_direction, node.bbox_min, node.bbox_max):
            continue

        if node.is_leaf:
            for prim_idx in node.primitive_indices:
                prim = primitives[prim_idx]
                t, u, v = möller_trumbore(ray_origin, ray_direction, prim.v0, prim.v1, prim.v2)
                if t is not None and 0 < t < closest_t:
                    closest_t = t
                    closest_prim_id = prim_idx
        else:
            # Determine traversal order
            t0, t1 = ray_box_intersect_t(ray_origin, ray_direction, node.left.bbox_min, node.left.bbox_max)
            t2, t3 = ray_box_intersect_t(ray_origin, ray_direction, node.right.bbox_min, node.right.bbox_max)

            # Push farther child first
            if t0 < t2:
                stack.append(node.right_idx)
                stack.append(node.left_idx)
            else:
                stack.append(node.left_idx)
                stack.append(node.right_idx)

    return closest_t, closest_prim_id
```

## Subtopics

### [Mesh BVH](1062-mesh-bvh.md)
Mesh BVH is a GPU-accelerated BVH designed for triangle meshes to support efficient distance queries, signed distance field (SDF) computation, and ray-triangle intersections. It serves as the primary acceleration structure for methods requiring fast nearest-point lookups on high-resolution geometry, such as the `isoext.sdf.TriangleMeshSDF` implementation. Unlike general-purpose BVHs that may prioritize ray intersection throughput, this variant is optimized for batched point-to-mesh distance calculations and sign determination, often leveraging hardware ray tracing (e.g., OptiX) for the signing phase. It distinguishes itself from [IRIS Ray-Geometry Intersection](3250-iris-ray-geometry-intersection.md) by focusing on distance metrics rather than pure intersection, and from [Multi-Hit BVH Traversal](12728-multi-hit-bvh-traversal.md) by typically returning the single closest primitive for distance queries.

### [IRIS Ray-Geometry Intersection](3250-iris-ray-geometry-intersection.md)
IRIS is an intersection-aware ray-based sampling method for implicit editable scenes that determines samples as the maximum response points of Gaussians intersected by each ray. It solves the problem of efficiently gathering depth samples from 3D Gaussian primitives in a hardware-accelerated ray tracing pipeline. It sets itself apart from standard BVH traversal methods by replacing explicit Gaussian volume checks with proxy geometry (affine-transformed icosahedra) and leveraging OptiX Any-hit shaders to compute sampling parameters directly during intersection tests.

### [Multi-Hit BVH Traversal](12728-multi-hit-bvh-traversal.md)
Multi-Hit BVH Traversal is a SIMD-friendly clipping algorithm for ray-traced direct volume rendering (DVR) that identifies visible volume regions by intersecting viewing rays with arbitrary geometric surface primitives (planes, quadrics, triangle meshes) organized in a Bounding Volume Hierarchy (BVH). It sets itself apart from rasterization-based depth peeling or texture-based slicing approaches by implementing clipping as a simple two-pass algorithm that avoids multiple render passes, dynamic memory reallocation, and the conversion of parametric surfaces to polygon meshes.

### [Initial Upper Bound Estimation](15442-initial-upper-bound-estimation.md)
Initial Upper Bound Estimation computes a tight initial upper bound to prune BVH-tree traversal during distance field computation for isosurface approximation. It differs from standard hierarchical traversal by proactively sampling the approximated surface to establish a pruning threshold before traversal begins, rather than waiting for the first leaf encounter. This allows for more aggressive pruning at the beginning of the traversal, reducing the total number of nodes visited.

### [Ray Tracing with BVH and Möller–Trumbore](19369-ray-tracing-with-bvh-and-m-ller-trumbore.md)
A rendering method using a PackedBVH over triangles, traversed with ray-box slab tests for pruning, and Möller–Trumbore tests for triangle intersections. It determines if a ray is occluded by any triangle (shadow ray), returning true on the first hit. Unlike sphere tracing, it requires no distance function and is exact for sharp CSG operations by walking primitive crossings along the ray.

### [BVH Traversal for Closest Point](19482-bvh-traversal-for-closest-point.md)
This method traverses a Bounding Volume Hierarchy (BVH) to find the triangle closest to a specific query point. It solves the problem of efficient nearest-neighbor search in geometric scenes by pruning branches that cannot contain a closer primitive than the current best. It sets itself apart from [Mesh BVH](1062-mesh-bvh.md) and [IRIS Ray-Geometry Intersection](3250-iris-ray-geometry-intersection.md) by operating on point-to-primitive distance rather than ray-parametric distance, and from [Multi-Hit BVH Traversal](12728-multi-hit-bvh-traversal.md) by returning only the single closest hit.

### [Ray Packet BVH Traversal](21088-ray-packet-bvh-traversal.md)
Ray Packet BVH Traversal traverses a Wald-ordered Bounding Volume Hierarchy (BVH) for a packet of rays to find the closest intersection for each active ray in the packet, or to determine which rays are occluded by triangles. It differs from single-ray traversal by processing multiple rays simultaneously, leveraging SIMD-like parallelism within the traversal stack and leaf intersection tests. It builds on [Bounding Volume Hierarchy (BVH) Traversal](18487-bounding-volume-hierarchy-bvh-traversal.md) by extending the node visitation logic to handle a range of ray indices (`first` to `first + packet_size`) rather than a single ray.

### Coarse Mesh Traversal via BVH
Uses a Bounding Volume Hierarchy (BVH) built over the coarse tetrahedral mesh to efficiently locate the coarse element containing a sample point [S3, unofficial]. Build a triangle BVH over the coarse mesh by inserting four triangular faces facing outside for every coarse element. Trace rays into the BVH using OptiX Wrapper Library (OWL) hit and miss functions. The hit function returns the hit point and the coarse tetrahedron ID. The miss function reports no intersection with BVH faces. Unlike element marching which uses connectivity, point location querying skips coarse elements not being sampled and allows varying refinement levels in the coarse mesh [S3, unofficial].

### BVH-based UDF evaluation with analytic gradient
Computes the unsigned distance from query points to a triangle mesh using a BVH, and provides an analytic gradient for backpropagation [S4, unofficial]. Flatten input x to [N, 3]. Query BVH for distance udf, triangle ID tri_id, and barycentric coordinates uvw. Clamp uvw to [1e-9, 1-1e-9]. Interpolate nearest point on triangle using uvw and triangle vertices. Return udf reshaped to input shape. In backward pass, retrieve saved x and nearest_points. Compute diff = x - nearest_points. Compute norm = diff.norm(dim=-1, keepdim=True) + 1e-8. Gradient is grad_output scaled by diff/norm. Formula: $\nabla_x \text{UDF}(x) = \frac{x - p_{\text{nearest}}}{\|x - p_{\text{nearest}}\|}$ where $p_{\text{nearest}}$ is the closest point on the mesh [S4, unofficial]. Input x shape [N, 3]. Output udf shape [N, 1]. Saved tensors: x_flat, nearest_points. Pitfall: uvw is clamped to [1e-9, 1-1e-9] to avoid invalid barycentric coordinates at triangle edges/corners [S4, unofficial].

### DOP-14 Separated Layout
A layout optimization for Discrete Oriented Polytopes (DOP-14) where the 14 bounding planes are split into two separate arrays to improve traversal performance by enabling independent memory access patterns [S5]. First check the AABB planes of the bounding volume (planes 0-2) during traversal. Only test planes 3–6 if warranted by the result of the initial AABB plane check [S5]. The BVHd type defines an Interior node with two children and a Leaf node with primitive count and data. The layout groups fields lo1, hi1, c0, c1 (planes 0-2) in one array, and lo2, hi2 (planes 3-6) in a separate array. In the layout specification, fields lo1, hi1, c0, and c1 correspond to AABB planes occupying the first array, accessed immediately. Fields lo2 and hi2 reside in a separate array, accessed only when the finer-grain intersection test is needed [S5]. Separates the logical bounding volume specification (which planes comprise a DOP-14) from the physical organization (which planes are co-located in memory), facilitating exploration of alternative representations [S5].

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ average case for balanced trees, $O(N)$ worst case for degenerate trees [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the tree and primitives [S2, unofficial].
*   **Performance:** Generally faster than linear traversal for complex scenes. Performance depends on tree balance and build quality [S1, unofficial].
*   **Strengths:** Efficient pruning of empty space, flexible geometry representation, good cache locality with flat layouts [S2, unofficial].
*   **Limitations:** Build time can be significant, sensitive to scene distribution, stack management overhead on GPUs [S2, unofficial].
*   **When to Use:** Complex scenes with many primitives, dynamic scenes with moderate updates (if rebuild is feasible), hardware ray tracing support [S1, unofficial].
*   **When Not to Use:** Very simple scenes, scenes with high dynamic update rates where rebuild cost is prohibitive, scenes with strong spatial coherence where grid-based methods may be simpler [S1, unofficial].

## Background
BVH traversal was introduced as a standard acceleration structure for ray tracing [S1, unofficial]. It has been widely adopted in both CPU and GPU ray tracers due to its efficiency and flexibility [S2, unofficial]. The method has evolved with hardware support for ray tracing (e.g., NVIDIA OptiX) [S3, unofficial].

## Sources
[S1] Radiation: the grid walk's cost is the distance travelled, not the triangles tested — start from the near end and skip empty space, https://github.com/DeGrootResearchGroup/aquaflux/issues/503, fetched 2026-10-25 (unofficial)
[S2] Optimizing Ray-Marching using CUDA for Volumetric Rendering, https://stephenlee129.github.io/volumetric-ray-marching/, fetched 2026-10-29 (unofficial)
[S3] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S4] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S5] Decoupling Data Layouts from Bounding Volume Hierarchies, https://arxiv.org/pdf/2511.15028, fetched 2026-10-29
