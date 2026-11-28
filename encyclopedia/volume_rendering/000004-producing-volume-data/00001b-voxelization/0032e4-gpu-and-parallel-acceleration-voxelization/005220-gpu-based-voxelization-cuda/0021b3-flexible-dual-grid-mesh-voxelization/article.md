# Flexible Dual Grid Mesh Voxelization

Also called: CUDA Mesh-to-Flexible-Dual-Grid

## Overview
Flexible Dual Grid Mesh Voxelization determines the set of active sparse voxels intersected by a triangle mesh and solves an enhanced Quadratic Error Function (QEF) to compute dual vertex positions within those voxels [S1, unofficial]. It serves as the foundational occupancy and geometry reconstruction stage for [O-Voxel](3835-o-voxel.md) [S1, unofficial]. It builds on [GPU-based Voxelization (CUDA)](14644-gpu-based-voxelization-cuda.md) by reproducing the original CPU Flexible Dual Grid computation on GPU [S1, unofficial].

## How it works
**Inputs:** `vertices` (shape $(N_v, 3)$, `float32`), `faces` (shape $(N_f, 3)$, `int32` or `long`), `grid_size` (shape $(3)$, `int32`), `aabb` (shape $(2, 3)$, `float32`), `face_weight` (scalar, default $1.0$), `boundary_weight` (scalar, default $0.2$), `regularization_weight` (scalar, default $10^{-2}$) [S1, unofficial].
**Outputs:** `voxel_indices` (occupied voxel coordinates), `dual_vertices` (shape $(N_{occ}, 3)$, `float32`), `intersected` (shape $(N_{occ}, 3)$, `bool` or packed `uint8`) [S1, unofficial].
**Data Structures:** Sparse voxel lookup uses a hashmap with `hashmap_keys` (size `capacity`, `uint32` if volume $< 2^{32}$ else `uint64`, initialized to max value) and `hashmap_vals` (size `capacity`, `uint32`) [S1, unofficial].
**Algorithm Steps:**
1. Validate inputs: `vertices` and `faces` must be 2D tensors with 3 columns on the same device, and either `voxel_size` or `grid_size` must be provided [S1, unofficial].
2. Normalize inputs: Convert `vertices` to `float32` contiguous tensor and `faces` to `int32` contiguous tensor on the target device [S1, unofficial]. Convert `voxel_size`, `grid_size`, and `aabb` from lists/tuples/ndarrays to `torch` tensors, ensuring metadata stays on CPU to avoid implicit CUDA synchronization before moving to device [S1, unofficial].
3. Compute or adjust AABB: If `aabb` is not provided, compute `min_xyz` and `max_xyz` from `vertices` [S1, unofficial]. If `voxel_size` is provided, compute padding as $padding = \lceil (max\_xyz - min\_xyz) / voxel\_size \rceil \cdot voxel\_size - (max\_xyz - min\_xyz)$, then adjust bounds: $min\_xyz \leftarrow min\_xyz - 0.5 \cdot padding$, $max\_xyz \leftarrow max\_xyz + 0.5 \cdot padding$ [S1, unofficial]. If `grid_size` is provided, compute padding as $padding = (max\_xyz - min\_xyz) / (grid\_size - 1)$, then adjust bounds identically [S1, unofficial]. Stack into `aabb` tensor [S1, unofficial].
4. Derive missing grid parameters: If `voxel_size_cpu` is None, compute $voxel\_size\_cpu = ((aabb\_cpu[1] - aabb\_cpu[0]) / grid\_size\_cpu).to(dtype=torch.float32).contiguous()$. If `grid_size_cpu` is None, compute $grid\_size\_cpu = ((aabb\_cpu[1] - aabb\_cpu[0]) / voxel\_size\_cpu).round().to(dtype=torch.int32).contiguous()$ [S1, unofficial].
5. Prepare CUDA kernel arguments: Convert `grid_size_cpu` to device tensor. Create `voxel_size_arg` as a list of floats from `voxel_size_cpu.tolist()`. Create `grid_range_arg` as `[0, 0, 0]` concatenated with integers from `grid_size_cpu.tolist()` [S1, unofficial].
6. Coordinate transformation: Translate vertices relative to AABB: $v' = v - aabb[0]$. Extract triangle coordinates: $triangles = vertices[faces]$ [S1, unofficial].
7. Occupancy computation: Invoke CUDA kernel `intersect_occ_cuda` (or CPU equivalent) with `triangles`, `voxel_size_arg`, and `grid_range_arg` to return occupied voxel indices [S1, unofficial].
8. QEF accumulation and dual vertex solve: For each occupied voxel, accumulate contributions from intersecting faces and boundaries to form the QEF matrix and vector. The enhanced QEF minimizes distance to the mesh surface using face term (weight `face_weight`), boundary term (weight `boundary_weight`), and regularization term (weight `regularization_weight`) [S1, unofficial]. Solve $A^T A x = A^T b$ for dual vertex position $x$ [S1, unofficial].
9. Pitfall handling: CUDA FMA contraction in cross-product based normal computation can produce a tiny non-zero normal for nearly degenerate triangles, amplified by normalization. The implementation uses an aligned cross-normalize path to fix this mismatch [S1, unofficial].
10. Sorting and serialization: Compute sequence ID $vid = \text{encode\_seq}(voxel\_indices)$. Compute mapping $mapping = \text{argsort}(vid)$. Apply mapping to `voxel_indices`, `dual_vertices`, and `intersected` to ensure alignment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def mesh_to_flexible_dual_grid(vertices, faces, grid_size=None, voxel_size=None, aabb=None,
                               face_weight=1.0, boundary_weight=0.2, regularization_weight=1e-2):
    # 1. Input Validation
    assert vertices.dim() == 2 and vertices.size(1) == 3
    assert faces.dim() == 2 and faces.size(1) == 3
    assert vertices.device == faces.device
    assert voxel_size is not None or grid_size is not None

    device = vertices.device
    vertices = vertices.float().contiguous()
    faces = faces.to(torch.int32).contiguous()

    # 2. Normalize inputs & handle metadata on CPU
    voxel_size_cpu = None
    grid_size_cpu = None
    if voxel_size is not None:
        if isinstance(voxel_size, float): voxel_size = [voxel_size, voxel_size, voxel_size]
        if isinstance(voxel_size, (list, tuple)): voxel_size = np.array(voxel_size)
        if isinstance(voxel_size, np.ndarray): voxel_size_cpu = torch.tensor(voxel_size, dtype=torch.float32)
        voxel_size = voxel_size_cpu.to(device=device).contiguous()
    if grid_size is not None:
        if isinstance(grid_size, (list, tuple)): grid_size = np.array(grid_size)
        if isinstance(grid_size, np.ndarray): grid_size_cpu = torch.tensor(grid_size, dtype=torch.int32)
        grid_size = grid_size_cpu.to(device=device).contiguous()

    # 3. Compute or adjust AABB
    if aabb is None:
        min_xyz = vertices.min(dim=0).values
        max_xyz = vertices.max(dim=0).values
    else:
        min_xyz = aabb[0]
        max_xyz = aabb[1]

    if voxel_size is not None:
        padding = torch.ceil((max_xyz - min_xyz) / voxel_size) * voxel_size - (max_xyz - min_xyz)
        min_xyz = min_xyz - 0.5 * padding
        max_xyz = max_xyz + 0.5 * padding
    if grid_size is not None:
        padding = (max_xyz - min_xyz) / (grid_size - 1)
        min_xyz = min_xyz - 0.5 * padding
        max_xyz = max_xyz + 0.5 * padding

    aabb_final = torch.stack([min_xyz, max_xyz])

    # 4. Derive missing grid parameters
    if voxel_size_cpu is None:
        voxel_size_cpu = ((aabb_final[1] - aabb_final[0]) / grid_size_cpu).to(dtype=torch.float32).contiguous()
    if grid_size_cpu is None:
        grid_size_cpu = ((aabb_final[1] - aabb_final[0]) / voxel_size_cpu).round().to(dtype=torch.int32).contiguous()

    # 5. Prepare CUDA arguments
    voxel_size_arg = [float(x) for x in voxel_size_cpu.tolist()]
    grid_range_arg = [0, 0, 0] + [int(x) for x in grid_size_cpu.tolist()]

    # 6. Coordinate transformation
    vertices_rel = vertices - aabb_final[0].reshape(1, 3)
    triangles = vertices_rel[faces]

    # 7. Occupancy computation
    # Sources do not provide the exact kernel launch syntax; invoke intersect_occ_cuda here [G]
    voxel_indices = torch.empty(0, dtype=torch.int32)

    # 8. QEF accumulation and dual vertex solve
    # Sources do not provide the exact kernel launch syntax; invoke QEF accumulation and solve here [G]
    dual_vertices, intersected = torch.empty(0, 3), torch.empty(0, 3)

    # 9. Sorting and serialization alignment
    vid = encode_seq(voxel_indices)
    mapping = torch.argsort(vid)
    voxel_indices = voxel_indices[mapping]
    dual_vertices = dual_vertices[mapping]
    intersected = intersected[mapping]

    return voxel_indices, dual_vertices, intersected
```

## Subtopics

### [Flexible Dual Grid Occupancy](3522-flexible-dual-grid-occupancy.md)
Computes only the set of sparse voxel coordinates intersected by a triangle mesh. It differs from [O-Voxel](3835-o-voxel.md) by strictly returning occupied indices without solving the QEF or computing edge intersection data. Choose this when only occupancy is needed.

### [O-Voxel](3835-o-voxel.md)
A sparse, voxel-based native 3D representation that utilizes Flexible Dual Grid formulation to robustly represent surfaces with arbitrary topology. It differs by solving an enhanced QEF for dual vertex positions and supporting volumetric surface properties like PBR attributes. Choose this for high-quality 3D generation and reconstruction requiring sharp features and open boundaries.

## Operating characteristics
*   **Time Complexity**: Dominated by the occupancy computation and QEF solve. The CUDA implementation provides significant speedup over CPU counterparts [S1, unofficial].
*   **Memory Complexity**: Sparse storage using a hashmap. Memory usage scales with the number of occupied voxels, not the total grid volume [S1, unofficial].
*   **Performance**:
    *   On a subdivided mesh (3.6M verts / 10.3M faces) at resolution 512:
        *   Reference CUDA: 113.9 ms, Dual RMS error 2.08e-4 [S1, unofficial].
        *   Optimized CUDA: 85.9 ms, Dual RMS error 1.16e-4 [S1, unofficial].
    *   Memory usage for the optimized CUDA implementation is ~1.8 GB, compared to ~12.9 GB for the reference, due to efficient sparse storage [S1, unofficial].
*   **Strengths**: Exact occupancy match with CPU implementation; zero intersected-flag mismatch; handles non-manifold and open surfaces; robust to nearly degenerate triangles via aligned cross-normalize path [S1, unofficial].
*   **Limitations**: Requires CUDA-capable hardware for optimal performance; precision issues with FMA contraction in cross-product normals if not handled (mitigated by aligned path) [S1, unofficial].
*   **When to use**: When converting triangle meshes to sparse voxel representations for rendering or reconstruction, especially when sharp features and open boundaries must be preserved [S1, unofficial].
*   **When not to use**: When a dense voxel grid is required; when CPU-only execution is mandatory [S1, unofficial].

## Background
The Flexible Dual Grid method was originally implemented on the CPU and later ported to GPU for performance [S1, unofficial]. The CUDA implementation `mesh_to_flexible_dual_grid` aims to reproduce the original CPU computation closely, including occupancy, QEF accumulation, boundary handling, and dual-vertex solve semantics [S1, unofficial]. It serves as the core geometry voxelization component for the O-Voxel library, which provides instant bidirectional conversion between Meshes and O-Voxels [S1, unofficial]. The implementation benefited from a reference CUDA implementation provided by `yuyujunjun/o-voxel-gpu` [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
