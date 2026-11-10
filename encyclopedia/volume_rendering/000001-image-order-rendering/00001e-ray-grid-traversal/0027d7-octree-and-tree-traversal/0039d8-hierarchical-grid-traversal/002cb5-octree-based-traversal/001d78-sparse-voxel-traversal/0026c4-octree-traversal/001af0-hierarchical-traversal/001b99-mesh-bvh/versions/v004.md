# Mesh BVH

Also called: GPU BVH for Triangle Mesh, MeshBVH, TriangleMeshSDF BVH

## Overview
Mesh BVH is a GPU-accelerated bounding volume hierarchy designed for distance queries on triangle meshes, primarily used by `isoext.sdf.TriangleMeshSDF` [S1, unofficial]. It enables efficient computation of signed distances, closest points, and winding numbers for large-scale volumetric rendering and SDF generation. It distinguishes itself from general-purpose CPU BVHs by being implemented as a CUDA tensor-based structure optimized for batched point queries on the GPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a triangle mesh defined by vertices $V$ (shape $(N_v, 3)$, `float32`) and faces $F$ (shape $(N_f, 3)$, `int32` vertex indices) [S1, unofficial]. It provides three query functions for a set of points $P$ (shape $(N_p, 3)$, `float32`):
1.  `closest(P)`: Returns distances $D$ (shape $(N_p,)$), closest points $C$ (shape $(N_p, 3)$), and face IDs $ID$ (shape $(N_p,)$) [S1, unofficial].
2.  `sign(P)`: Returns a signed distance indicator $S$ (shape $(N_p,)$) where $+1$ is outside and $-1$ is inside [S1, unofficial].
3.  `winding_number(P)`: Returns a winding number $W$ (shape $(N_p,)$) where $1$ is inside a closed mesh and $0$ is outside [S1, unofficial].

**Data Structures:**
The hierarchy is built on CUDA tensors. The internal structure is not explicitly detailed in the sources, but it relies on a BVH traversal logic that tests ray/point proximity to triangle primitives [S1, unofficial]. The closest point calculation on a triangle $abc$ with query point $p$ uses the Voronoi region method [S1, unofficial]:
1.  Compute vectors $ab = b - a$, $ac = c - a$, $ap = p - a$.
2.  Compute dot products $d_1 = ab \cdot ap$, $d_2 = ac \cdot ap$.
3.  If $d_1 \le 0$ and $d_2 \le 0$, the closest point is $a$.
4.  Compute $bp = p - b$, $d_3 = ab \cdot bp$, $d_4 = ac \cdot bp$.
5.  If $d_3 \ge 0$ and $d_4 \le d_3$, the closest point is $b$.
6.  Compute $vc = d_1 d_4 - d_3 d_2$. If $vc \le 0$ and $d_1 \ge 0$ and $d_3 \le 0$, the closest point lies on edge $ab$ at $a + ab \frac{d_1}{d_1 - d_3}$.
7.  Compute $cp = p - c$, $d_5 = ab \cdot cp$, $d_6 = ac \cdot cp$.
8.  If $d_6 \ge 0$ and $d_5 \le d_6$, the closest point is $c$.
9.  Compute $vb = d_5 d_2 - d_1 d_6$. If $vb \le 0$ and $d_2 \ge 0$ and $d_6 \le 0$, the closest point lies on edge $ac$ at $a + ac \frac{d_2}{d_2 - d_6}$ [S1, unofficial].
10. Otherwise, the closest point lies inside the triangle, calculated via barycentric coordinates [S1, unofficial].

**Steps:**
1.  **Construction**: Initialize the BVH with vertices $V$ and faces $F$ as CUDA tensors [S1, unofficial].
2.  **Query**: For each point $p_i$ in $P$, traverse the BVH to find the closest triangle.
3.  **Distance Calculation**: Compute the Euclidean distance from $p_i$ to the closest point on the triangle [S1, unofficial].
4.  **Sign/Winding**: Determine the sign or winding number based on the ray intersection parity or inside/outside test relative to the mesh [S1, unofficial].

**Parameters:**
Not given in the sources.

**Edge Cases:**
Degenerate triangles (zero area) may cause division by zero in barycentric calculations if not handled [G]. The sources do not specify how self-intersecting or non-manifold meshes are handled for the `sign` and `winding_number` functions [S1, unofficial].

## Subtopics

### [Position-Normal Tree](4222-position-normal-tree.md)
A binary tree hierarchy over a triangle mesh where leaves are triangles and internal nodes store an axis-aligned bounding box (AABB) for positions and a bounding cone for shading normals. It enables hierarchical pruning of refractive path searches by eliminating subtrees that cannot contain valid refraction points, addressing the performance bottleneck of testing all triangles in large meshes. It is distinguished from interval-based refinement by prioritizing speed over guaranteed convergence, typically running 2 to 5 times faster while producing visually equivalent results.

### [Voxel Mesh Ray Intersection via BVH](6900-voxel-mesh-ray-intersection-via-bvh.md)
Voxel Mesh Ray Intersection via BVH is a raycasting method for voxel meshes that traverses a grid of oriented sub-cubes to find intersections with voxel geometry using Binary Volume Hierarchy (BVH) triangle intersection tests. It handles multiple meshes by first checking bounding boxes and then delegating to the specific mesh's BVH. This method builds on Mesh BVH by providing the specific traversal logic and data structures required to query triangle meshes stored within a voxelized world structure.

### BVH-based Geometric Attribute Correction
Uses a BVH to compute the unsigned distance and barycentric coordinates from interpolated UV-space positions back to the original high-resolution mesh, correcting geometric errors from simplification/remeshing [S2, unofficial].
1.  Query BVH: `_, face_id, uvw = bvh.unsigned_distance(valid_pos, return_uvw=True)` [S2, unofficial].
2.  Fetch original triangle vertices: `orig_tri_verts = vertices[faces[face_id.long()]]` [S2, unofficial].
3.  Reconstruct corrected position via barycentric interpolation: `valid_pos = (orig_tri_verts * uvw.unsqueeze(-1)).sum(dim=1)` [S2, unofficial].

### BVH for Blending
A bounding volume hierarchy adapted for the management of the blending operation, used for efficient evaluation of field functions on large skeletons [S3].

## Operating characteristics
Time and memory complexity are not explicitly given in the sources. The method is designed for GPU execution using CUDA tensors, implying parallel processing of batched queries [S1, unofficial]. Performance scales with the number of points and the depth of the BVH, which depends on the mesh complexity [S1, unofficial]. Strengths include efficient batched distance queries on the GPU. Limitations include the lack of details on construction time and memory overhead for very large meshes [S1, unofficial].

## Background
Originated in the `isoext` library by Guangyan Cai, providing a GPU-based SDF implementation for triangle meshes [S1, unofficial]. It is used in volumetric rendering pipelines requiring fast signed distance field generation from mesh data.

## Sources
[S1] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
[S2] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
