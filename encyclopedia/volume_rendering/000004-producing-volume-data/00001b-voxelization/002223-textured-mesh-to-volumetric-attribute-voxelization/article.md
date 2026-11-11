# Textured Mesh to Volumetric Attribute Voxelization


## Overview
This method samples texture maps into voxel space to produce volumetric attributes, converting a textured mesh asset into a structured voxel grid containing material properties such as base color, metallic, and roughness [S1, unofficial]. It differs from standard binary voxelization by explicitly preserving and discretizing per-vertex or per-face texture attributes into the volumetric domain, enabling direct attribute queries during volume rendering without post-hoc interpolation [S1, unofficial].

## How it works
**Inputs:**
- `asset`: A mesh defined by vertices $V$ and faces $F$, along with associated texture maps (e.g., base color, metallic, roughness).
- `grid_size`: The resolution of the output voxel grid $(N_x, N_y, N_z)$. Can be an integer (uniform), list/tuple, or tensor.
- `aabb`: Axis-Aligned Bounding Box defined by minimum $[min_x, min_y, min_z]$ and maximum $[max_x, max_y, max_z]$ coordinates. Default is $[-0.5, -0.5, -0.5]$ to $[0.5, 0.5, 0.5]$.
- `voxel_size`: The physical size of a single voxel. Must be a 1D tensor/array of 3 floats $[s_x, s_y, s_z]$.

**Outputs:**
- `voxel_indices_mat`: A tensor of voxel indices corresponding to occupied voxels.
- `attributes`: A dictionary mapping attribute names (e.g., `'base_color'`, `'metallic'`, `'roughness'`) to tensors of values aligned with `voxel_indices_mat`.

**Data Structures:**
- `voxel_indices_mat`: 1D integer tensor. Each element represents a unique voxel index in the flattened grid.
- `attributes`: Dictionary where each value is a 1D tensor of floats, indexed identically to `voxel_indices_mat`.

**Steps:**
1. **Normalize and Validate Inputs:**
   Ensure `voxel_size` is a 1D tensor of 3 elements on the target device. If `grid_size` is not provided, compute it from `aabb` and `voxel_size`:
   $$ N_i = \text{round}\left(\frac{max_i - min_i}{s_i}\right) $$
   for $i \in \{x, y, z\}$. [S1, unofficial]

2. **Transform Mesh Geometry:**
   Translate vertices so the AABB origin is at $(0,0,0)$:
   $$ V' = V - aabb_{min} $$
   Construct triangle tensors from transformed vertices and face indices. [S1, unofficial]

3. **Compute Occupancy:**
   Perform geometric intersection between the transformed triangles and the voxel grid defined by `voxel_size` and `grid_range` (derived from `grid_size`). This produces a set of occupied voxel indices. [S1, unofficial]

4. **Sort and Align Data:**
   To ensure attributes align with voxel indices, compute a sequence ID for each occupied voxel:
   $$ vid\_mat = \text{serialize\_encode\_seq}(voxel\_indices\_mat) $$
   Compute the sorting permutation:
   $$ mapping\_mat = \text{argsort}(vid\_mat) $$
   Apply this mapping to all attribute tensors to ensure they are ordered consistently with the voxel indices:
   $$ attributes = \{k : v[mapping\_mat] \mid k, v \in attributes\} $$
   [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def textured_mesh_to_volumetric_attr(mesh, aabb, voxel_size, grid_size=None):
    # 1. Validate and prepare voxel_size
    assert voxel_size.dim() == 1 and voxel_size.size(0) == 3
    voxel_size_cpu = voxel_size.to(dtype=torch.float32).contiguous()
    
    # 2. Determine grid_size if not provided
    if grid_size is None:
        aabb_cpu = aabb.detach()
        grid_size_cpu = ((aabb_cpu[1] - aabb_cpu[0]) / voxel_size_cpu).round().to(dtype=torch.int32)
        grid_size = grid_size_cpu
    else:
        # Handle int/list/tuple conversion to tensor
        if isinstance(grid_size, int):
            grid_size = [grid_size, grid_size, grid_size]
        if isinstance(grid_size, (list, tuple)):
            grid_size = np.array(grid_size)
        grid_size_cpu = torch.tensor(grid_size, dtype=torch.int32)
        grid_size = grid_size_cpu

    # 3. Transform vertices to local AABB space
    vertices = mesh.vertices - aabb[0].reshape(1, 3)
    triangles = vertices[mesh.faces].contiguous()

    # 4. Compute occupied voxel indices
    # Implementation depends on backend (CPU/GPU)
    voxel_indices_mat = intersect_occ(triangles, voxel_size_cpu, grid_size_cpu)

    # 5. Extract and align attributes
    # Assume mesh has pre-computed per-vertex or per-face attributes
    # This step assumes attributes are already sampled into the occupied voxels
    # The core alignment logic from sources:
    vid_mat = serialize.encode_seq(voxel_indices_mat)
    mapping_mat = torch.argsort(vid_mat)
    
    attributes = {}
    for k, v in mesh.attributes.items():
        attributes[k] = v[mapping_mat]
        
    return voxel_indices_mat, attributes
```

**Edge Cases and Pitfalls:**
- `voxel_size` must be on CPU for metadata operations to avoid implicit CUDA synchronization [S1, unofficial].
- If `grid_size` is derived from `aabb` and `voxel_size`, rounding errors may cause slight mismatches between the logical grid and physical bounds [S1, unofficial].
- Attribute alignment relies on `serialize.encode_seq` and `argsort`; incorrect serialization of voxel indices will lead to misaligned material properties [S1, unofficial].

## Subtopics

### [Transparent Illustration via Voxelization](4782-transparent-illustration-via-voxelization.md)
Extends this method to store multivalued data (normals, texture coordinates, material IDs) in a voxel worksheet for slicing-based volume rendering.

### [Trimesh Mesh Voxelization & Texture Color Mapping](6284-trimesh-mesh-voxelization-texture-color-mapping.md)
Focuses on mapping vertex texture colors to voxels using barycentric projection or proximity queries during voxelization.

### [Surface-aligned Texture Field (TexF)](10067-surface-aligned-texture-field-texf.md)
A UV-free representation that embeds attributes directly in a surface-aligned 3D field, reducing discretization errors.

## Operating characteristics
- **Time Complexity:** Dominated by the geometric intersection step between triangles and voxels. Sorting and mapping are $O(N \log N)$ where $N$ is the number of occupied voxels.
- **Memory Complexity:** Linear in the number of occupied voxels and the number of attributes stored per voxel.
- **Strengths:** Preserves high-frequency texture details in the volumetric domain; enables direct attribute access during rendering.
- **Limitations:** Requires explicit texture sampling during voxelization; sensitive to AABB alignment and voxel size discretization.

## Background
Originated in the context of GPU-accelerated voxelization pipelines for real-time rendering, specifically within the `o-voxel-gpu` library [S1, unofficial]. It addresses the need for efficient transfer of PBR (Physically Based Rendering) attributes from mesh assets to volumetric representations.

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
