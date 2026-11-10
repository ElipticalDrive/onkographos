# O-Voxel

Also called: Flexible Dual Grid, O-Voxel representation

## Overview
O-Voxel is a sparse, voxel-based native 3D representation designed for high-quality 3D generation and reconstruction [S1, unofficial]. It utilizes a Flexible Dual Grid formulation to robustly represent surfaces with arbitrary topology, including non-manifold and open surfaces, and supports volumetric surface properties such as Physically-Based Rendering (PBR) material attributes [S1, unofficial]. Unlike traditional field-based methods (e.g., Occupancy fields, SDFs), O-Voxel solves an enhanced Quadratic Error Function (QEF) to accurately capture sharp features and open boundaries without requiring watertight meshes [S1, unofficial]. It builds on [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md) by providing specific QEF weighting and data structures for this representation [S1, unofficial].

## How it works
**Inputs:** A triangular mesh (vertices and faces) and a target grid resolution $R$ [S1, unofficial].
**Outputs:** A sparse set of active voxel indices, dual vertex positions within those voxels, intersection flags, and optional volumetric attributes (e.g., PBR colors) [S1, unofficial].

**Data Structures:**
1. **Voxel Indices:** A list of 3D integer coordinates $(x, y, z)$ for active voxels [S1, unofficial].
2. **Dual Vertices:** For each active voxel, a 3D position $(u, v, w)$ normalized to $[0, 1]$ relative to the voxel grid, stored as `uint8` (0-255) or float [S1, unofficial].
3. **Intersected Flags:** A 3-bit integer per active voxel indicating which of the 8 corners of the voxel are intersected by the mesh surface. Bit 0 corresponds to corner $(0,0,0)$, bit 1 to $(1,0,0)$, etc. [S1, unofficial].
4. **Attributes:** Optional dense arrays aligned with voxel indices for PBR properties (Base Color, Metallic, Roughness, Opacity) [S1, unofficial].

**Parameters:**
* `face_weight`: Weight of the face term in the QEF. Default: 1.0 [S1, unofficial].
* `boundary_weight`: Weight of the boundary term in the QEF. Default: 1.0 [S1, unofficial].
* `regularization_weight`: Weight of the regularization term in the QEF. Default: 0.1 [S1, unofficial].

**Algorithm Steps:**
1. **Occupancy Determination:** Determine which voxels in the $R \times R \times R$ grid are intersected by the mesh. This is handled by the parent method [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md) [S1, unofficial].
2. **QEF Accumulation:** For each active voxel, accumulate the Quadratic Error Function (QEF) matrices and vectors based on the mesh geometry intersecting the voxel. The QEF is enhanced with three terms: Face Term (derived from plane equations of intersecting mesh faces), Boundary Term (derived from mesh edges passing through voxel boundaries), and Regularization Term (a small identity matrix scaled by `regularization_weight` for numerical stability) [S1, unofficial].
3. **Dual Vertex Solve:** Solve the linear system $A\mathbf{x} = \mathbf{b}$ for each active voxel, where $A$ is the accumulated QEF matrix and $\mathbf{b}$ is the accumulated vector. The solution $\mathbf{x}$ is the dual vertex position within the voxel [S1, unofficial].
4. **Clamping:** Clamp the solved dual vertex coordinates to the range $[0, 1]$ to ensure they remain within the voxel bounds [S1, unofficial].
5. **Attribute Assignment:** Assign volumetric attributes (e.g., PBR colors) to the active voxels, typically by sampling or averaging the mesh attributes at the dual vertex position or voxel center [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def solve_o_voxel(mesh_vertices, mesh_faces, grid_size, face_weight=1.0, boundary_weight=1.0, regularization_weight=0.1):
    # Step 1: Occupancy determination is delegated to the parent method [Flexible Dual Grid Mesh Voxelization]
    # It returns active voxel indices and initial corner intersection flags.
    # voxel_indices, intersected_flags = parent_occupancy_step(...)
    voxel_indices = torch.empty((0, 3), dtype=torch.int32, device=mesh_vertices.device)
    intersected_flags = torch.empty((0, 3), dtype=torch.bool, device=mesh_vertices.device)
    
    if voxel_indices.shape[0] == 0:
        return voxel_indices, torch.empty((0, 3), dtype=torch.uint8), torch.empty((0, 3), dtype=torch.uint8), {}
    
    N = voxel_indices.shape[0]
    
    # Step 2: QEF Accumulation
    A = torch.zeros((N, 3, 3), device=mesh_vertices.device)
    b = torch.zeros((N, 3), device=mesh_vertices.device)
    A += regularization_weight * torch.eye(3, device=mesh_vertices.device).unsqueeze(0)
    
    # [G] Iterate over mesh faces and edges intersecting each active voxel.
    # For each face i with normal n_i and distance d_i:
    #   A[k] += face_weight * (n_i @ n_i.T)
    #   b[k] += face_weight * (n_i * d_i)
    # For each edge j with direction e_j:
    #   A[k] += boundary_weight * (e_j @ e_j.T)
    #   b[k] += boundary_weight * (e_j * projection_term)
    # Note: CUDA FMA contraction in cross-product based normal computation must be handled
    # to prevent tiny non-zero normals for nearly degenerate triangles [S1, unofficial].
    
    # Step 3: Solve QEF
    dual_vertices = torch.linalg.solve(A, b)
    
    # Step 4: Clamp to [0, 1]
    dual_vertices = torch.clamp(dual_vertices, 0.0, 1.0)
    
    # Step 5: Pack dual vertices to uint8 (0-255) and intersected flags to uint8 (3-bit packing)
    dual_vertices_uint8 = (dual_vertices * 255).to(torch.uint8)
    intersected_packed = (intersected_flags[:, 0:1].to(torch.uint8) + 
                          2 * intersected_flags[:, 1:2].to(torch.uint8) + 
                          4 * intersected_flags[:, 2:3].to(torch.uint8))
    
    # Step 6: Attribute assignment
    attributes = {}
    
    return voxel_indices, dual_vertices_uint8, intersected_packed, attributes
```

**Edge Cases and Pitfalls:**
* **Degenerate Triangles:** Nearly degenerate triangles can produce tiny non-zero normals, leading to numerical instability in QEF accumulation. The CUDA implementation addresses this by handling FMA contraction issues [S1, unofficial].
* **Open Surfaces:** O-Voxel is designed to handle open surfaces and non-manifold geometry, which traditional watertight mesh methods struggle with. The boundary term in the QEF helps capture open boundaries accurately [S1, unofficial].
* **Precision:** Dual vertex positions are stored as `uint8` (0-255) in the compressed format [S1, unofficial].

## Subtopics

### Flexible Dual Grid Occupancy
Determines the active sparse voxels and solves the QEF to determine dual vertex positions within voxels based on mesh-voxel grid intersections. It focuses on the occupancy and QEF solving aspects. [Flexible Dual Grid Occupancy](3522-flexible-dual-grid-occupancy.md)

## Operating characteristics
* **Time Complexity:** The conversion time depends on the number of mesh faces and the grid resolution [S1, unofficial]. On an NVIDIA GeForce RTX 4090 with PyTorch 2.6.0+cu124 and CUDA 12.4, converting a mesh with 268k vertices and 280k faces at grid size 512 takes 27.1 ms warm time [S1, unofficial]. For a subdivided mesh with 3.6M vertices and 10.3M faces, it takes 82.1 ms warm time [S1, unofficial].
* **Memory Complexity:** Memory usage scales with the number of active voxels and the attributes stored per voxel [S1, unofficial]. For the 268k vertex mesh, warm peak memory is 939.7 MB [S1, unofficial]. For the 3.6M vertex mesh, it is 1836.8 MB [S1, unofficial].
* **Strengths:** Instant bidirectional conversion between meshes and O-Voxels without expensive SDF evaluation or iterative optimization [S1, unofficial]. Native support for PBR attributes and arbitrary topology [S1, unofficial].
* **Limitations:** Requires a CUDA implementation for optimal performance [S1, unofficial].
* **When to Use:** High-quality 3D generation and reconstruction where PBR attributes and arbitrary topology are important [S1, unofficial].
* **When Not to Use:** When strict watertight mesh representation is required, or when CPU-only performance is critical [S1, unofficial].

## Background
O-Voxel was introduced by quantaji in the `o-voxel-gpu` repository [S1, unofficial]. It is designed for high-quality 3D generation and reconstruction, leveraging the Flexible Dual Grid formulation to handle complex geometries [S1, unofficial]. The implementation provides efficient CUDA kernels for mesh-to-O-Voxel conversion, aiming for parity with CPU implementations while offering significant speedups [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
