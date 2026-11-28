# Mesh BVH

Also called: GPU BVH for Triangle Mesh, MeshBVH, TriangleMeshSDF BVH

## Overview
Mesh BVH is a GPU-accelerated bounding volume hierarchy designed for triangle meshes to support efficient distance queries, signed distance field (SDF) computation, and ray-triangle intersections [S1, unofficial][S4]. It serves as the primary acceleration structure for methods requiring fast nearest-point lookups on high-resolution geometry, such as the `isoext.sdf.TriangleMeshSDF` implementation [S1, unofficial]. Unlike general-purpose BVHs that may prioritize ray intersection throughput, this variant is optimized for batched point-to-mesh distance calculations and sign determination, often leveraging hardware ray tracing (e.g., OptiX) for the signing phase [S4]. It distinguishes itself from [IRIS Ray-Geometry Intersection](3250-iris-ray-geometry-intersection.md) by focusing on distance metrics rather than pure intersection, and from [Multi-Hit BVH Traversal](12728-multi-hit-bvh-traversal.md) by typically returning the single closest primitive for distance queries [S1, unofficial][S4].

## How it works
The method constructs a hierarchical tree over a triangle mesh to accelerate queries for the closest point, distance, and topological sign (inside/outside) for arbitrary 3D points.

### Inputs and Outputs
*   **Inputs:**
    *   Vertices $V$: A CUDA tensor of shape $(N_v, 3)$ with `float32` data type, representing mesh vertex positions [S1, unofficial].
    *   Faces $F$: A CUDA tensor of shape $(N_f, 3)$ with `int32` data type, representing vertex indices for each triangle [S1, unofficial].
*   **Outputs:**
    *   `closest`: For input points $P$ of shape $(N, 3)$, returns a tuple:
        *   Distances $D$: Tensor of shape $(N,)$ `float32` [S1, unofficial].
        *   Closest Points $C$: Tensor of shape $(N, 3)$ `float32` [S1, unofficial].
        *   Face IDs $I$: Tensor of shape $(N,)$ identifying the closest triangle [S1, unofficial].
    *   `sign`: For input points $P$, returns a tensor of shape $(N,)$ `float32` with values $+1$ (outside) or $-1$ (inside) [S1, unofficial].
    *   `winding_number`: For input points $P$, returns a tensor of shape $(N,)$ `float32` with value $1$ (inside closed mesh) or $0$ (outside) [S1, unofficial].

### Data Structures
The BVH is a binary tree where each node contains an Axis-Aligned Bounding Box (AABB) [S1, unofficial].
*   **Leaf Nodes:** Contain references to the triangle indices stored in the input face tensor $F$ [S1, unofficial].
*   **Internal Nodes:** Contain an AABB that encloses the union of the bounding volumes of its children [S1, unofficial].
*   **Storage:** The hierarchy is stored in GPU memory (CUDA tensors) to enable parallel traversal [S1, unofficial].

### Algorithm Steps

#### 1. Construction
The constructor takes the vertex tensor $V$ and face tensor $F$ [S1, unofficial].
1.  Compute the AABB for each triangle in $F$ using the vertices in $V$ [S1, unofficial].
2.  Build the binary tree hierarchy bottom-up or via a parallel cluster merging strategy on the GPU [S1, S1, unofficial].
3.  Store the resulting node structure in device memory [S1, unofficial].

#### 2. Unsigned Distance Query (`closest`)
Given a batch of query points $P_{query}$:
1.  Traverse the BVH for each point $p \in P_{query}$ to find the closest triangle [S1, unofficial].
2.  For the candidate triangle with vertices $a, b, c$, compute the closest point on the triangle to $p$ using Voronoi region tests [S1, unofficial]:
    *   Let $ab = b - a$, $ac = c - a$, $ap = p - a$.
    *   Compute dot products: $d_1 = ab \cdot ap$, $d_2 = ac \cdot ap$.
    *   If $d_1 \le 0$ and $d_2 \le 0$, the closest point is $a$ [S1, unofficial].
    *   Let $bp = p - b$. Compute $d_3 = ab \cdot bp$, $d_4 = ac \cdot bp$.
    *   If $d_3 \ge 0$ and $d_4 \le d_3$, the closest point is $b$ [S1, unofficial].
    *   Compute $vc = d_1 d_4 - d_3 d_2$. If $vc \le 0$ and $d_1 \ge 0$ and $d_3 \le 0$, the closest point lies on edge $ab$ at $a + ab \frac{d_1}{d_1 - d_3}$ [S1, unofficial].
    *   Let $cp = p - c$. Compute $d_5 = ab \cdot cp$, $d_6 = ac \cdot cp$.
    *   If $d_6 \ge 0$ and $d_5 \le d_6$, the closest point is $c$ [S1, unofficial].
    *   Compute $vb = d_5 d_2 - d_1 d_6$. If $vb \le 0$ and $d_2 \ge 0$ and $d_6 \le 0$, the closest point lies on edge $ac$ at $a + ac \frac{d_2}{d_2 - d_6}$ [S1, unofficial].
    *   Otherwise, the closest point lies in the triangle interior at $a + ab \frac{d_1 d_4 - d_3 d_2}{vb} + ac \frac{d_5 d_2 - d_1 d_6}{vb}$ (barycentric interpolation) [S1, unofficial].
3.  Return the distance $\|p - closest\_point\|$, the closest point coordinates, and the face ID [S1, unofficial].

#### 3. Sign Determination (`sign`)
To determine if a point is inside or outside:
1.  **Watertight Check:** If the mesh is known to be watertight, the sign can be determined by the dot product of the view direction and the normal of the closest triangle found in step 2. However, this is disabled by default due to incompatibility with non-watertight meshes [S4].
2.  **Stab Ray Method (General):**
    *   Trace 32 "stab rays" distributed uniformly over the sphere using a Fibonacci lattice [S4].
    *   Each ray is pseudorandomly and independently offset for every training position [S4].
    *   Use NVIDIA ray tracing hardware (OptiX 7) for ray-triangle intersections [S4].
    *   If any ray reaches infinity (no intersection), the point is marked **outside** ($+1$) [S4].
    *   If all rays intersect the mesh, the point is marked **inside** ($-1$) [S4].

#### 4. Winding Number (`winding_number`)
Returns $1$ if the point is inside a closed mesh, $0$ otherwise [S1, unofficial]. This is typically computed via the same ray casting logic as `sign`, counting intersections or using the stab ray method [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

class MeshBVH:
    def __init__(self, vertices: torch.Tensor, faces: torch.Tensor):
        """
        vertices: (N_v, 3) float32 CUDA tensor
        faces: (N_f, 3) int32 CUDA tensor
        """
        self.vertices = vertices
        self.faces = faces
        # [G] Construct BVH structure on GPU. 
        # Implementation details depend on the specific builder (e.g., cluster merging).
        self.bvh_tree = build_bvh_gpu(vertices, faces)

    def closest(self, points: torch.Tensor) -> tuple:
        """
        points: (N, 3) float32 CUDA tensor
        Returns: distances (N,), closest_points (N, 3), face_ids (N,)
        """
        # [G] Traverse BVH to find closest triangle for each point.
        # For each point p, find face_id.
        face_ids = traverse_bvh_for_closest_face(self.bvh_tree, points)
        
        # Fetch triangle vertices
        tri_verts = self.vertices[self.faces[face_ids]] # (N, 3, 3)
        
        # Compute closest point on each triangle
        a = tri_verts[:, 0]
        b = tri_verts[:, 1]
        c = tri_verts[:, 2]
        
        ab = b - a
        ac = c - a
        ap = points - a
        
        d1 = torch.sum(ab * ap, dim=1)
        d2 = torch.sum(ac * ap, dim=1)
        
        # Region 1: Vertex A
        mask_a = (d1 <= 0) & (d2 <= 0)
        
        bp = points - b
        d3 = torch.sum(ab * bp, dim=1)
        d4 = torch.sum(ac * bp, dim=1)
        
        # Region 2: Vertex B
        mask_b = (d3 >= 0) & (d4 <= d3)
        
        vc = d1 * d4 - d3 * d2
        # Region 3: Edge AB
        mask_ab = (vc <= 0) & (d1 >= 0) & (d3 <= 0)
        
        cp = points - c
        d5 = torch.sum(ab * cp, dim=1)
        d6 = torch.sum(ac * cp, dim=1)
        
        # Region 4: Vertex C
        mask_c = (d6 >= 0) & (d5 <= d6)
        
        vb = d5 * d2 - d1 * d6
        # Region 5: Edge AC
        mask_ac = (vb <= 0) & (d2 >= 0) & (d6 <= 0)
        
        # Region 6: Interior
        mask_interior = ~(mask_a | mask_b | mask_ab | mask_c | mask_ac)
        
        closest_points = torch.zeros_like(points)
        
        # Compute barycentric coordinates for interior and edge cases
        # Edge AB: t = d1 / (d1 - d3)
        denom_ab = d1 - d3
        safe_denom_ab = torch.where(denom_ab == 0, torch.ones_like(denom_ab), denom_ab)
        t_ab = d1 / safe_denom_ab
        closest_points[mask_ab] = a[mask_ab] + ab[mask_ab] * t_ab[mask_ab, None]
        
        # Edge AC: t = d2 / (d2 - d6)
        denom_ac = d2 - d6
        safe_denom_ac = torch.where(denom_ac == 0, torch.ones_like(denom_ac), denom_ac)
        t_ac = d2 / safe_denom_ac
        closest_points[mask_ac] = a[mask_ac] + ac[mask_ac] * t_ac[mask_ac, None]
        
        # Interior: barycentric coords
        # u = (d2*d5 - d1*d6) / vb
        # v = (d1*d4 - d3*d2) / vb
        # w = 1 - u - v
        # Note: vb can be zero for degenerate triangles, handled by [G] guard
        denom_vb = vb
        safe_denom_vb = torch.where(denom_vb == 0, torch.ones_like(denom_vb), denom_vb)
        
        u = (d2 * d5 - d1 * d6) / safe_denom_vb
        v = (d1 * d4 - d3 * d2) / safe_denom_vb
        w = 1.0 - u - v
        
        closest_points[mask_interior] = (
            u[mask_interior, None] * a[mask_interior] +
            v[mask_interior, None] * b[mask_interior] +
            w[mask_interior, None] * c[mask_interior]
        )
        
        # Vertices A and B are already handled by mask_a and mask_b (closest_point is a or b)
        closest_points[mask_a] = a[mask_a]
        closest_points[mask_b] = b[mask_b]
        closest_points[mask_c] = c[mask_c]
        
        distances = torch.norm(points - closest_points, dim=1)
        
        return distances, closest_points, face_ids

    def sign(self, points: torch.Tensor) -> torch.Tensor:
        """
        points: (N, 3) float32 CUDA tensor
        Returns: (N,) float32 tensor: +1 outside, -1 inside
        """
        # [G] Use OptiX 7 for stab ray tracing if available, otherwise software BVH.
        # Stab rays: 32 rays distributed via Fibonacci lattice.
        num_stab_rays = 32
        # [G] Generate stab ray directions and offsets.
        # If any ray hits infinity, point is outside.
        # This requires a ray-triangle intersection kernel.
        # Simplified logic:
        hits = trace_stab_rays_optix(points, num_stab_rays) # Returns boolean tensor (N,)
        # If any ray misses (hits infinity), it's outside.
        # Assuming hits[i] is True if ALL rays hit the mesh.
        inside = torch.all(hits, dim=1)
        sign = torch.where(inside, -1.0, 1.0)
        return sign
```

### Parameters
*   **Number of stab rays:** 32 [S4].
*   **Ray distribution:** Fibonacci lattice with pseudorandom independent offset per point [S4].

### Edge Cases and Pitfalls
*   **Non-watertight meshes:** The normal-based signing method is disabled by default because it is incompatible with typical non-watertight meshes found in the wild [S4]. The stab ray method is required for correctness in these cases [S4].
*   **Degenerate Triangles:** The barycentric calculation involves division by `vb` or `vc`. If the triangle is degenerate (area zero), these denominators are zero. A guard must handle this, typically by falling back to vertex distances or treating it as a line/point [S1, G, unofficial].
*   **Precision:** Floating-point precision in distance calculations can affect SDF accuracy. Using `float32` is standard, but `float64` may be needed for high-precision applications [S1, G, unofficial].

## Subtopics

### [Position-Normal Tree](4222-position-normal-tree.md)
A binary tree hierarchy over a triangle mesh where leaves are triangles and internal nodes contain a 3D axis-aligned bounding box (AABB) over positions and a bounding cone over shading normals for all descendants. It stores position and normal information for boundary triangles to enable hierarchical pruning of refractive path searches. It is distinguished from interval-based refinement by prioritizing speed over guaranteed convergence, typically running 2 to 5 times faster while producing visually equivalent results. Choose this when refractive path tracing requires fast normal-based pruning. [Position-Normal Tree](4222-position-normal-tree.md)

### [Voxel Mesh Ray Intersection via BVH](6900-voxel-mesh-ray-intersection-via-bvh.md)
A raycasting method for voxel meshes that traverses a grid of oriented sub-cubes to find intersections with voxel geometry using BVH triangle intersection tests. It differs from its parent [Mesh BVH](1062-mesh-bvh.md) by adding a spatial grid traversal layer that maps ray positions to specific voxel meshes before delegating to the mesh's internal BVH. It differs from siblings like [Position-Normal Tree](4222-position-normal-tree.md) by relying on explicit triangle intersection tests within leaf nodes rather than implicit distance fields. Choose this for voxel-based scenes with explicit triangle geometry. [Voxel Mesh Ray Intersection via BVH](6900-voxel-mesh-ray-intersection-via-bvh.md)

### [GPU BVH Construction via Cluster Merging](16461-gpu-bvh-construction-via-cluster-merging.md)
This method constructs a Binary Volume Hierarchy (BVH) on the GPU by iteratively merging clusters of triangles. It solves the problem of building hierarchical acceleration structures for ray tracing without CPU-GPU synchronization bottlenecks by using a fully parallel, double-buffered approach involving neighbor finding, prefix scans, and reallocation. It differs from CPU-based top-down or bottom-up builders by keeping all data on the GPU and using a "merge-compact" loop that reduces the number of active clusters by half (approximately) each iteration until a single root remains. Choose this when BVH construction must be fully GPU-accelerated. [GPU BVH Construction via Cluster Merging](16461-gpu-bvh-construction-via-cluster-merging.md)

### BVH-based Geometric Attribute Correction
Uses a BVH to compute the unsigned distance and barycentric coordinates from interpolated UV-space positions back to the original high-resolution mesh, correcting geometric errors from simplification/remeshing [S2, unofficial]. Query the BVH for the closest face and UVW coordinates, fetch the original triangle vertices, and reconstruct the corrected position via barycentric interpolation [S2, unofficial]. Choose this when high-fidelity geometric attributes are needed from a simplified mesh.

### BVH for Blending
A bounding volume hierarchy adapted for the management of the blending operation, used for efficient evaluation of field functions on large skeletons [S3]. Choose this for skeletal animation blending where field evaluation is the bottleneck.

### Efficient Signed Distance to Triangle Mesh
A method to compute the signed distance from a 3D point to a triangle mesh efficiently using a BVH for unsigned distance and ray casting for sign determination [S4]. Construct a BVH for $O(\log N)$ unsigned distance queries, then trace 32 stab rays using a Fibonacci lattice to determine sign [S4]. Use OptiX 7 for ray-triangle intersections for maximum efficiency [S4]. Choose this for high-performance SDF computation on GPUs.

## Operating characteristics
*   **Time Complexity:** Unsigned distance queries are $O(\log N_{triangles})$ on average [S4]. Sign determination using 32 stab rays is $O(32 \cdot \log N_{triangles})$ or $O(32 \cdot R)$ where $R$ is ray traversal cost [S4].
*   **Memory Complexity:** $O(N_{triangles})$ for storing the BVH nodes and triangle references [S1, unofficial].
*   **Performance:** Using OptiX 7 for ray-triangle intersections is over an order of magnitude faster than software triangle BVH for ray-shape intersections on an RTX 3090 GPU [S4].
*   **Strengths:** High performance on GPU, supports batched queries, handles large meshes efficiently [S1, unofficial][S4].
*   **Limitations:** Requires GPU memory for the BVH [S1, unofficial]. Sign determination via stab rays is computationally expensive compared to normal-based signing for watertight meshes [S4].
*   **When to use:** When fast distance queries and SDF computation are needed for triangle meshes on GPU [S1, unofficial][S4].
*   **When not to use:** For CPU-only environments or when the mesh is small enough that linear search is faster [S4, G].

## Background
Mesh BVH was developed as part of the `isoext` library for GPU-based signed distance field computation [S1, unofficial]. It leverages NVIDIA's OptiX 7 framework for hardware-accelerated ray tracing, which was shown to significantly outperform software BVH implementations in the context of Instant Neural Graphics Primitives [S4]. The method is used in applications requiring efficient geometric queries for neural rendering and SDF-based modeling [S1, unofficial][S4].

## Sources
[S1] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S2] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S4] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
