# O-Voxel

Also called: Flexible Dual Grid, O-Voxel representation

## Overview
O-Voxel is a sparse, voxel-based native 3D representation designed for high-quality 3D generation and reconstruction [S1, unofficial]. It utilizes a Flexible Dual Grid formulation to robustly represent surfaces with arbitrary topology (including non-manifold and open surfaces) and volumetric surface properties such as Physically-Based Rendering (PBR) material attributes [S1, unofficial]. Unlike traditional methods that rely on fields (e.g., Occupancy fields, SDFs), O-Voxel solves an enhanced Quadratic Error Function (QEF) to accurately capture sharp features and open boundaries without requiring watertight meshes [S1, unofficial]. It builds on [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md) by providing the specific QEF weighting and data structures for this representation.

## How it works
**Inputs:** A triangular mesh (vertices and faces) and a target grid resolution $R$.
**Outputs:** A sparse set of active voxel indices, dual vertex positions within those voxels, intersection flags, and optional volumetric attributes (e.g., PBR colors).

**Data Structures:**
1.  **Voxel Indices:** A list of 3D integer coordinates $(x, y, z)$ for active voxels.
2.  **Dual Vertices:** For each active voxel, a 3D position $(u, v, w)$ normalized to $[0, 1]$ relative to the voxel grid, stored as `uint8` (0-255) or float.
3.  **Intersected Flags:** A 3-bit integer per active voxel indicating which of the 8 corners of the voxel are intersected by the mesh surface. Bit 0 corresponds to corner $(0,0,0)$, bit 1 to $(1,0,0)$, etc.
4.  **Attributes:** Optional dense arrays aligned with voxel indices for PBR properties (Base Color, Metallic, Roughness, Opacity) [S1, unofficial].

**Parameters:**
*   `face_weight`: Weight of the face term in the QEF. Default: 1.0 [S1, unofficial].
*   `boundary_weight`: Weight of the boundary term in the QEF. Default: 1.0 [S1, unofficial].
*   `regularization_weight`: Weight of the regularization term in the QEF. Default: 0.1 [S1, unofficial].

**Algorithm Steps:**
1.  **Occupancy Determination:** Determine which voxels in the $R \times R \times R$ grid are intersected by the mesh. This is handled by the parent method [Flexible Dual Grid Mesh Voxelization](2640-flexible-dual-grid-mesh-voxelization.md).
2.  **QEF Accumulation:** For each active voxel, accumulate the Quadratic Error Function (QEF) matrices and vectors based on the mesh geometry intersecting the voxel. The QEF is enhanced with three terms:
    *   **Face Term:** Derived from the plane equations of the mesh faces intersecting the voxel.
    *   **Boundary Term:** Derived from the edges of the mesh that pass through the voxel boundaries.
    *   **Regularization Term:** A small identity matrix scaled by `regularization_weight` to ensure numerical stability [S1, unofficial].
3.  **Dual Vertex Solve:** Solve the linear system $A\mathbf{x} = \mathbf{b}$ for each active voxel, where $A$ is the accumulated QEF matrix and $\mathbf{b}$ is the accumulated vector. The solution $\mathbf{x}$ is the dual vertex position within the voxel [S1, unofficial].
4.  **Clamping:** Clamp the solved dual vertex coordinates to the range $[0, 1]$ to ensure they remain within the voxel bounds [S1, unofficial].
5.  **Attribute Assignment:** Assign volumetric attributes (e.g., PBR colors) to the active voxels, typically by sampling or averaging the mesh attributes at the dual vertex position or voxel center [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def solve_o_voxel(mesh_vertices, mesh_faces, grid_size, face_weight=1.0, boundary_weight=1.0, regularization_weight=0.1):
    """
    Convert a mesh to O-Voxel representation.
    
    Args:
        mesh_vertices: Tensor of shape (V, 3)
        mesh_faces: Tensor of shape (F, 3)
        grid_size: int, resolution R
        face_weight: float
        boundary_weight: float
        regularization_weight: float
        
    Returns:
        voxel_indices: Tensor of shape (N, 3)
        dual_vertices: Tensor of shape (N, 3) in [0, 1]
        intersected: Tensor of shape (N, 3) as uint8 (bit-packed flags)
        attributes: Dict of optional attributes
    """
    # Step 1: Determine active voxels and intersection flags
    # This step is implemented in the parent method [Flexible Dual Grid Mesh Voxelization]
    # It returns active voxel indices and which corners are intersected.
    voxel_indices, intersected_flags = flexible_dual_grid_mesh_voxelization(mesh_vertices, mesh_faces, grid_size)
    
    N = voxel_indices.shape[0]
    if N == 0:
        return voxel_indices, torch.empty((0, 3)), intersected_flags, {}
    
    # Step 2: Accumulate QEF for each active voxel
    # For each active voxel, find the mesh faces and edges that intersect it.
    # Accumulate the QEF matrix A (3x3) and vector b (3) for each voxel.
    # A = sum(face_weight * n_i * n_i^T) + sum(boundary_weight * e_j * e_j^T) + regularization_weight * I
    # b = sum(face_weight * (n_i * d_i) * n_i) + sum(boundary_weight * ...)
    
    # Initialize QEF accumulators
    A = torch.zeros((N, 3, 3), device=mesh_vertices.device)
    b = torch.zeros((N, 3), device=mesh_vertices.device)
    
    # Add regularization term
    A += regularization_weight * torch.eye(3, device=mesh_vertices.device).unsqueeze(0)
    
    # Find intersecting faces and edges for each active voxel
    # This requires spatial indexing (e.g., BVH or grid-based) to efficiently find
    # which faces/edges intersect which voxels.
    # For each active voxel k:
    #   For each intersecting face i:
    #     Compute plane normal n_i and distance d_i
    #     A[k] += face_weight * (n_i @ n_i.T)
    #     b[k] += face_weight * (n_i * d_i)
    #   For each intersecting edge j:
    #     Compute edge direction e_j
    #     A[k] += boundary_weight * (e_j @ e_j.T)
    #     b[k] += boundary_weight * ... # Boundary term vector accumulation
    
    # Note: The exact accumulation logic for boundary terms is detailed in the 
    # Flexible Dual Grid Mesh Voxelization parent article.
    
    # Step 3: Solve QEF for each voxel
    # x = A^-1 * b
    # Use torch.linalg.solve for stability
    dual_vertices = torch.linalg.solve(A, b)
    
    # Step 4: Clamp to [0, 1]
    dual_vertices = torch.clamp(dual_vertices, 0.0, 1.0)
    
    # Step 5: Pack dual vertices to uint8 if needed for storage
    # dual_vertices_uint8 = (dual_vertices * 255).type(torch.uint8)
    
    # Step 6: Assign attributes (e.g., PBR colors)
    # This is typically done by sampling the mesh attributes at the dual vertex position
    # or voxel center.
    attributes = {}
    # attributes['base_color'] = sample_mesh_attributes(mesh_vertices, mesh_faces, dual_vertices)
    
    return voxel_indices, dual_vertices, intersected_flags, attributes
```

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** Nearly degenerate triangles can produce tiny non-zero normals, leading to numerical instability in QEF accumulation. The CUDA implementation addresses this by handling FMA contraction issues [S1, unofficial].
*   **Open Surfaces:** O-Voxel is designed to handle open surfaces and non-manifold geometry, which traditional watertight mesh methods struggle with. The boundary term in the QEF helps capture open boundaries accurately [S1, unofficial].
*   **Precision:** Dual vertex positions are stored as `uint8` (0-255) in the compressed format, which may introduce quantization error. For high-precision applications, float storage should be used [S1, unofficial].

## Subtopics

### Flexible Dual Grid Occupancy
Determines the active sparse voxels and solves the QEF to determine dual vertex positions within voxels based on mesh-voxel grid intersections. It focuses on the occupancy and QEF solving aspects. [Flexible Dual Grid Occupancy](3522-flexible-dual-grid-occupancy.md)

## Operating characteristics
*   **Time Complexity:** The conversion time depends on the number of mesh faces and the grid resolution. On an NVIDIA GeForce RTX 4090, converting a mesh with 268k vertices and 280k faces at grid size 512 takes approximately 27.1 ms [S1, unofficial]. For a subdivided mesh with 3.6M vertices and 10.3M faces, it takes approximately 82.1 ms [S1, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of active voxels and the attributes stored per voxel. For the 268k vertex mesh, warm peak memory is 939.7 MB [S1, unofficial]. For the 3.6M vertex mesh, it is 1836.8 MB [S1, unofficial].
*   **Strengths:** Instant bidirectional conversion between meshes and O-Voxels without expensive SDF evaluation or iterative optimization [S1, unofficial]. Native support for PBR attributes and arbitrary topology [S1, unofficial].
*   **Limitations:** Requires a CUDA implementation for optimal performance [S1, unofficial]. Quantization error in `uint8` storage may affect precision [S1, unofficial].
*   **When to Use:** High-quality 3D generation and reconstruction where PBR attributes and arbitrary topology are important.
*   **When Not to Use:** When strict watertight mesh representation is required, or when CPU-only performance is critical.

## Background
O-Voxel was introduced by quantaji in the `o-voxel-gpu` repository [S1, unofficial]. It is designed for high-quality 3D generation and reconstruction, leveraging the Flexible Dual Grid formulation to handle complex geometries. The implementation provides efficient CUDA kernels for mesh-to-O-Voxel conversion, aiming for parity with CPU implementations while offering significant speedups [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
