# Triangulated Active Brick Boundary Mesh for Space Skipping

Also called: Active Brick Boundary Mesh, Active Region Boundary Geometry, Boundary Mesh Space Skipping, RTX-Accelerated Empty Space Skipping using Triangulated Active Region Boundary Geometry

## Overview
A GPU-accelerated space-skipping technique for structured volumes that replaces clustered proxy boxes with a triangle mesh bounding the surface between active and inactive macrocells [S1, unofficial]. It solves the host-GPU transfer bottleneck and high depth complexity of prior proxy-box methods by generating the boundary geometry entirely on the GPU and using native RTX hardware for ray-triangle intersection [S1, unofficial].

## How it works
**Inputs:**
- Input volume dimensions $V_x \times V_y \times V_z$ [S1, unofficial].
- Brick size $K$ (default 16), defining macrocell resolution [S1, unofficial].
- Transfer function (TF) defining active scalar ranges [S1, unofficial].
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1, unofficial].

**Outputs:**
- Volume integration intervals $[t_0, t_1]$ for active regions [S1, unofficial].
- Final pixel color [S1, unofficial].

**Data Structures:**
1. **Macrocell Grid:** Size $M_x \times M_y \times M_z$, where $M_x = \lfloor V_x/K \rfloor$, etc. Each cell stores the minimum and maximum scalar values of input voxels mapping to it [S1, unofficial].
2. **Active Bricks Mask:** Boolean grid $M_x \times M_y \times M_z$ on GPU. `true` if macrocell has non-zero alpha for current TF [S1, unofficial].
3. **Vertex Array:** Pre-allocated `float3` array of size $N_{vtxMax} = (M_x+1)(M_y+1)(M_z+1)$. Contains all possible macrocell grid vertices [S1, unofficial].
4. **Triangle Index Array:** Pre-allocated `int3` array of size $N_{triMax}$ (upper bound based on boundary geometry on macrocell faces). Stores indices referencing the Vertex Array [S1, unofficial].
5. **numTriangles:** Device-side integer counter, initialized to 0, used as an atomic offset for writing triangles [S1, unofficial].
6. **OptiX BVH:** Built over the generated triangle mesh using OptiX 7.2 [S1, unofficial].

**Steps:**

1. **Initial Setup:** Build Macrocell Grid on host, upload to GPU. Allocate GPU memory for Active Bricks Mask, Vertex Array, and Triangle Index Array. Initialize `numTriangles = 0` [S1, unofficial].
2. **Active Cell Classification (Per TF Change):** Launch CUDA kernel. Each thread maps to one macrocell $(i,j,k)$. Check if macrocell's scalar range maps to non-zero alpha in TF. Write result to `ActiveBricksMask[i][j][k]` [S1, unofficial].
3. **Boundary Triangle Generation (Per TF Change):** Clear `numTriangles = 0`. Run three templated CUDA kernels for X, Y, Z orientations. For orientation $D \in \{0,1,2\}$ (X,Y,Z): Thread maps to face perpendicular to axis $D$. Let `curr` be macrocell index on positive side of face, `prev` on negative side. If `prev` is out-of-bounds, treat as inactive. If `ActiveBricksMask[prev] == ActiveBricksMask[curr]`, return. Get 4 vertex indices $\{v_0, v_1, v_2, v_3\}$ spanning the face. `triID = atomicAdd(&numTriangles, 2)`. If `ActiveBricksMask[prev]` is true (prev active, curr inactive): `Triangles[triID] = {v_0, v_1, v_2}`, `Triangles[triID+1] = {v_0, v_2, v_3}`. Else (`curr` active, `prev` inactive): `Triangles[triID] = {v_1, v_0, v_2}`, `Triangles[triID+1] = {v_2, v_0, v_3}`. Winding order points away from active towards inactive. Download `numTriangles` to host [S1, unofficial].
4. **BVH Rebuild:** Update OptiX geometry to reference generated mesh. Rebuild BVH with "optimize for traversal" flag. Compact BVH [S1, unofficial].
5. **Ray Tracing & Integration:** Initialize ray with $t_{min}=0, t_{max}=\infty$. Trace ray against BVH with `DISABLE_ANYHIT` flag. **No Hit:** Terminate ray. **Hit:** Get hit distance $t_{hit}$ and hit kind (front/back). **Back-Facing Hit:** Ray is inside volume. Integration interval $[t_0, t_1] = [0, t_{hit}]$. **Front-Facing Hit:** Ray enters volume at $t_0 = t_{hit}$. Trace same ray with $t_{min}=t_0$ to find exit $t_1$. **Volume Integration:** Sample volume texture at fixed rate $dt=0.5$ (interleaved) over $[t_0, t_1]$. Apply gradient shading and early ray termination. If ray terminated early, return. Else, trace new ray from $t_1$ to find next segment. Repeat [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_boundary_mesh(Mx, My, Mz, active_cells, num_triangles, triangles):
    # active_cells: 3D boolean array of shape (Mx+1, My+1, Mz+1)
    # num_triangles: mutable counter [0]
    # triangles: pre-allocated list of tuples (int3)
    # Dimensions for faces perpendicular to X, Y, Z
    dims = [
        (range(Mx + 1), range(My), range(Mz)),       # X-perpendicular faces
        (range(Mx), range(My + 1), range(Mz)),       # Y-perpendicular faces
        (range(Mx), range(My), range(Mz + 1))        # Z-perpendicular faces
    ]
    for dim_idx, (range_i, range_j, range_k) in enumerate(dims):
        for i in range_i:
            for j in range_j:
                for k in range_k:
                    curr = (i, j, k)
                    # prev is on the negative side of the face
                    if dim_idx == 0: prev = (i - 1, j, k)
                    elif dim_idx == 1: prev = (i, j - 1, k)
                    else: prev = (i, j, k - 1)

                    # Out-of-bounds treated as inactive
                    prev_active = active_cells[prev] if is_inside(prev, Mx, My, Mz) else False
                    curr_active = active_cells[curr]

                    if prev_active == curr_active:
                        continue

                    # Get 4 vertex indices spanning the face
                    vi = get_face_vertex_indices(curr, dim_idx, Mx, My, Mz)
                    tri_id = num_triangles[0]
                    num_triangles[0] += 2

                    if prev_active:
                        triangles[tri_id] = (vi[0], vi[1], vi[2])
                        triangles[tri_id + 1] = (vi[0], vi[2], vi[3])
                    else:
                        triangles[tri_id] = (vi[1], vi[0], vi[2])
                        triangles[tri_id + 1] = (vi[2], vi[0], vi[3])

def is_inside(idx, Mx, My, Mz):
    return 0 <= idx[0] <= Mx and 0 <= idx[1] <= My and 0 <= idx[2] <= Mz

def get_face_vertex_indices(curr, dim_idx, Mx, My, Mz):
    # Returns 4 vertex indices into the flat vertex array of size (Mx+1)*(My+1)*(Mz+1)
    # Vertex index formula: i * (My+1)*(Mz+1) + j * (Mz+1) + k
    stride_yz = (My + 1) * (Mz + 1)
    stride_z = Mz + 1
    i, j, k = curr
    if dim_idx == 0: # X-perpendicular face at i
        # Vertices are at (i, j, k), (i, j+1, k), (i, j+1, k+1), (i, j, k+1)
        return (i*stride_yz + j*stride_z + k,
                i*stride_yz + (j+1)*stride_z + k,
                i*stride_yz + (j+1)*stride_z + (k+1),
                i*stride_yz + j*stride_z + (k+1))
    elif dim_idx == 1: # Y-perpendicular face at j
        return (i*stride_yz + j*stride_z + k,
                (i+1)*stride_yz + j*stride_z + k,
                (i+1)*stride_yz + j*stride_z + (k+1),
                i*stride_yz + j*stride_z + (k+1))
    else: # Z-perpendicular face at k
        return (i*stride_yz + j*stride_z + k,
                (i+1)*stride_yz + j*stride_z + k,
                (i+1)*stride_yz + (j+1)*stride_z + k,
                i*stride_yz + (j+1)*stride_z + k)
```

**Pitfalls:**
- Warp divergence: Reduced by lowering depth complexity to transitions between active/inactive regions [S1, unofficial].
- Host-GPU transfer bottleneck: Avoided by generating geometry on GPU, enabling interactive TF edits [S1, unofficial].
- Boundary Faces: Faces on the outside of the volume may have indices mapping outside the macrocell grid; treated as inactive macrocells [S1, unofficial].
- Unused Vertices: Pre-allocating all possible vertices results in unused vertices; accepted as a memory cost to avoid tracking usage [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Mesh generation is $O(M_x M_y M_z)$ per TF change, taking low milliseconds [S1, unofficial]. BVH rebuild is hardware-accelerated [S1, unofficial].
- **Memory Complexity:** Vertex array is fixed size $(M_x+1)(M_y+1)(M_z+1)$. Triangle array is pre-allocated to upper bound [S1, unofficial].
- **Performance:** Significantly faster than Ganter and Manzke [GM19] due to no host-GPU transfer and no CPU clustering [S1, unofficial].
- **Strengths:** No host-GPU bottleneck. Lower depth complexity than proxy boxes. Hardware-accelerated intersection [S1, unofficial].
- **Limitations:** BVH must be rebuilt on TF change. Pre-allocated vertex array wastes memory [S1, unofficial].
- **When to use:** Interactive TF editing on GPU. Structured volumes [S1, unofficial].
- **When not to use:** Unstructured data [S1, unofficial].

## Background
- **Origin:** Ganter and Manzke [GM19] introduced RTX-accelerated space skipping using proxy boxes. This method extends it by replacing boxes with a boundary mesh [S1, unofficial].
- **History:** Inspired by unstructured volume rendering techniques using boundary tetrahedra [BKS97] [S1, unofficial]. Implemented using OptiX 7.2 [S1, unofficial].
- **Usage:** Used in interactive volume rendering systems supporting TF editing [S1, unofficial].

## Sources
[S1] Faster RTX-Accelerated Empty Space Skipping using Triangulated Active ..., https://diglib.eg.org/items/dcc28036-9ef9-42d2-bff8-7a4bf1269b57, fetched 2026-10-29 (unofficial)
