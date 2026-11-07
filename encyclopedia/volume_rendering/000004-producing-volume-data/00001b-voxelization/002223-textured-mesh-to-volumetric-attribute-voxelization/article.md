# Textured Mesh to Volumetric Attribute Voxelization


## Overview
This method samples texture maps and material properties from a textured 3D mesh into a discrete voxel grid to produce volumetric attributes such as base color, metallic, and roughness [S1, unofficial]. It solves the problem of transferring continuous surface appearance data to a volumetric representation for subsequent volume rendering or analysis. It differs from standard occupancy voxelization by focusing on attribute mapping rather than binary occupancy, and from proximity-based methods by utilizing rasterization or intersection-based sampling to align attributes with the voxel grid established by the parent [Volumetric Attribute and Density Formation](6304-volumetric-attribute-and-density-formation.md).

## How it works
**Inputs:**
- `asset`: A mesh object containing vertices, faces, and associated texture maps/material properties.
- `grid_size`: The resolution of the output voxel grid, specified as an integer (cubical) or a list/tuple/array of three integers $(N_x, N_y, N_z)$ [S1, unofficial].
- `aabb`: The Axis-Aligned Bounding Box defining the spatial extent of the voxelization, specified as two 3D vectors $[min_x, min_y, min_z]$ and $[max_x, max_y, max_z]$ [S1, unofficial].

**Outputs:**
- `voxel_indices_mat`: A tensor or array of voxel indices corresponding to occupied voxels.
- `attributes`: A dictionary mapping attribute names to tensors/arrays. Common keys include `'base_color'`, `'metallic'`, and `'roughness'` [S1, unofficial].

**Data Structures:**
- The output attributes are aligned with `voxel_indices_mat`. Each attribute tensor has a shape compatible with the number of occupied voxels, indexed by a sequence ID [S1, unofficial].
- For proximity-based variants, the color array layout is shape $(X, Y, Z, 4)$, where $X, Y, Z$ are the grid dimensions and 4 represents RGBA components [S2, unofficial].

**Steps:**
1.  **Normalize and Prepare Mesh Data**:
    -   Ensure `voxel_size` is a 1D tensor of length 3 on the CPU [S1, unofficial].
    -   Compute `grid_size` from `aabb` and `voxel_size` if not explicitly provided:
        $$ grid\_size = \text{round}\left(\frac{aabb[1] - aabb[0]}{voxel\_size}\right) $$
        [S1, unofficial].
    -   Translate mesh vertices relative to the AABB origin:
        $$ vertices_{local} = vertices - aabb[0] $$
        [S1, unofficial].
    -   Construct triangles from local vertices and face indices [S1, unofficial].

2.  **Compute Occupancy and Initial Indices**:
    -   Perform mesh-voxel intersection to determine occupied voxels. This can be done via CUDA or CPU kernels depending on device availability [S1, unofficial].
    -   Generate `voxel_indices_mat` containing the indices of all occupied voxels [S1, unofficial].

3.  **Serialize and Sort for Alignment**:
    -   To ensure strict alignment between occupancy masks and attribute data, serialize the voxel indices into a sequence ID:
        $$ vid\_mat = \text{serialize.encode\_seq}(voxel\_indices\_mat) $$
        [S1, unofficial].
    -   Compute the sorting permutation that orders these sequence IDs:
        $$ mapping\_mat = \text{argsort}(vid\_mat) $$
        [S1, unofficial].

4.  **Map Attributes**:
    -   Sample or extract attributes (e.g., base color, metallic, roughness) for each occupied voxel. The specific sampling method (rasterization vs. proximity) depends on the implementation variant [S1, S2, unofficial].
    -   Apply the sorting mapping to all attribute tensors to match the sorted `voxel_indices_mat`:
        $$ attributes = \{ k : v[mapping\_mat] \text{ for } k, v \text{ in attributes.items()} \} $$
        [S1, unofficial].

5.  **Output**:
    -   Return `voxel_indices_mat` and the sorted `attributes` dictionary [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def textured_mesh_to_volumetric_attr(mesh, grid_size, aabb, voxel_size):
    # 1. Prepare inputs
    # Ensure voxel_size is 1D float32 tensor on CPU
    voxel_size_cpu = voxel_size.to(dtype=torch.float32).contiguous()
    assert voxel_size_cpu.dim() == 1 and voxel_size_cpu.size(0) == 3
    
    # Compute grid_size from aabb if needed
    if grid_size is None:
        aabb_cpu = aabb.detach()
        grid_size_cpu = ((aabb_cpu[1] - aabb_cpu[0]) / voxel_size_cpu).round().to(dtype=torch.int32)
        grid_size = grid_size_cpu
    else:
        # Handle int/list/tuple conversion to tensor
        if isinstance(grid_size, int):
            grid_size = [grid_size, grid_size, grid_size]
        grid_size_cpu = torch.tensor(grid_size, dtype=torch.int32)
        grid_size = grid_size_cpu

    # 2. Transform mesh vertices
    vertices = mesh.vertices - aabb[0].reshape(1, 3)
    triangles = vertices[mesh.faces.to(dtype=torch.long)]
    
    # 3. Compute occupancy
    # Use CUDA if available, else CPU
    if vertices.is_cuda:
        voxel_indices_mat = intersect_occ_cuda(triangles, voxel_size_cpu, grid_size)
    else:
        voxel_indices_mat = intersect_occ_cpu(triangles, voxel_size_cpu, grid_size)
        
    # 4. Serialize and Sort
    # Serialize voxel indices to a unique sequence ID
    vid_mat = serialize_encode_seq(voxel_indices_mat)
    # Compute sorting indices
    mapping_mat = torch.argsort(vid_mat)
    
    # 5. Extract and Map Attributes
    # Note: Specific attribute extraction (color, metallic, etc.) 
    # depends on the mesh's texture/material data.
    # This step assumes attributes are extracted into a dict of tensors 
    # aligned with the raw voxel_indices_mat order.
    raw_attributes = extract_attributes_from_mesh(mesh, voxel_indices_mat)
    
    # Apply mapping to align attributes with sorted indices
    sorted_attributes = {
        k: v[mapping_mat] for k, v in raw_attributes.items()
    }
    
    return voxel_indices_mat, sorted_attributes
```

**Edge Cases and Pitfalls:**
-   **Non-Watertight Meshes**: If the input mesh is not perfectly watertight (e.g., due to Structure from Motion reconstruction artifacts), the voxelized mesh may exhibit holes [S2, unofficial].
-   **Precision**: `voxel_size` metadata must be on CPU to avoid implicit CUDA synchronization; pass Python values or ensure CPU placement [S1, unofficial].
-   **Alignment**: Failure to sort `voxel_indices_mat` and apply the same permutation to attributes will result in misaligned data [S1, unofficial].

## Subtopics

### Trimesh Mesh Voxelization & Texture Color Mapping
This variant uses proximity queries to map vertex colors to a hollow voxel grid. It voxelizes the mesh into a hollow grid and maps the mesh's vertex texture colors to voxels by finding the closest mesh vertex for each voxel point [S2, unofficial]. It is chosen when a simple, library-based (Trimesh) approach is sufficient and the mesh is expected to be relatively clean. It differs from the primary method by using distance-based nearest-neighbor lookup rather than rasterization/intersection sampling for attribute assignment [S2, unofficial]. The color array layout is $(X, Y, Z, 4)$ [S2, unofficial]. Pitfalls include potential holes in the voxelized mesh if the input is not watertight [S2, unofficial].

## Operating characteristics
-   **Time Complexity**: Dominated by the mesh-voxel intersection step and the sorting operation. Intersection is $O(N_v \cdot N_t)$ where $N_v$ is voxel count and $N_t$ is triangle count, depending on the spatial acceleration structure. Sorting is $O(N \log N)$ where $N$ is the number of occupied voxels [S1, unofficial].
-   **Memory Complexity**: Proportional to the number of occupied voxels times the number of attributes. The attribute tensors must store values for each occupied voxel [S1, unofficial].
-   **Strengths**: Produces dense, aligned volumetric attributes suitable for volume rendering pipelines. Handles multiple material properties (color, metallic, roughness) [S1, unofficial].
-   **Limitations**: Requires watertight meshes for optimal results in proximity-based variants. Sorting step adds overhead [S1, unofficial].
-   **When to Use**: When volumetric rendering of textured meshes is required and the rendering engine supports attribute-based volume shading.
-   **When Not to Use**: For simple binary occupancy visualization or when mesh quality is poor and holes are unacceptable without post-processing.

## Background
The method originates from GPU-accelerated voxelization libraries such as `o-voxel-gpu`, which provide efficient CUDA-based intersection and attribute mapping routines [S1, unofficial]. It is used in modern volume rendering pipelines that require high-fidelity texture mapping onto volumetric data structures, particularly in real-time or near-real-time applications where rasterization-based attribute sampling is preferred over point-cloud-based proximity methods [S1, S2, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
