# Textured Mesh to Volumetric Attribute Voxelization
## Overview
This method samples texture maps into voxel space to produce volumetric attributes, converting a textured mesh asset into a structured voxel grid containing material properties such as base color, metallic, and roughness [S1, unofficial].
## How it works
**Inputs:**
- `asset`: A mesh with associated texture maps [S1, unofficial].
- `grid_size`: The resolution of the output voxel grid $(N_x, N_y, N_z)$ [S1, unofficial].
- `aabb`: Axis-Aligned Bounding Box defined by minimum and maximum coordinates, typically $[-0.5, -0.5, -0.5]$ to $[0.5, 0.5, 0.5]$ [S1, unofficial].
- `voxel_size`: The physical size of a single voxel, specified as a 1D array of 3 floats $[s_x, s_y, s_z]$ [S1, unofficial].

**Outputs:**
- `voxel_indices_mat`: A tensor of voxel indices corresponding to occupied voxels [S1, unofficial].
- `attributes`: A dictionary mapping attribute names (e.g., `'base_color'`, `'metallic'`, `'roughness'`) to tensors of values aligned with `voxel_indices_mat` [S1, unofficial].

**Steps:**
1. **Validate Inputs:** Ensure `voxel_size` is a 1D tensor/array of 3 floats on the target device [S1, unofficial].
2. **Compute Grid Resolution:** If `grid_size` is not provided, compute it from `aabb` and `voxel_size`: $N_i = \text{round}((max_i - min_i) / s_i)$ for $i \in \{x, y, z\}$ [S1, unofficial].
3. **Transform Geometry:** Translate vertices so the AABB origin is at $(0,0,0)$: $V' = V - aabb_{min}$, then construct triangle tensors [S1, unofficial].
4. **Compute Occupancy:** Perform geometric intersection between the transformed triangles and the voxel grid defined by `voxel_size` and `grid_size` to produce a set of occupied voxel indices [S1, unofficial].
5. **Sort and Align Data:** To ensure attributes align with voxel indices, compute a sequence ID for each occupied voxel: $vid\_mat = \text{serialize\_encode\_seq}(voxel\_indices\_mat)$. Compute the sorting permutation: $mapping\_mat = \text{argsort}(vid\_mat)$. Apply this mapping to all attribute tensors: $attributes = \{k : v[mapping\_mat] \mid k, v \in attributes\}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def textured_mesh_to_volumetric_attr(mesh, aabb_min, aabb_max, voxel_size, grid_size=None, attributes=None):
    # 1. Validate voxel_size
    assert len(voxel_size) == 3
    voxel_size = [float(x) for x in voxel_size]

    # 2. Determine grid_size if not provided
    if grid_size is None:
        grid_size = [round((aabb_max[i] - aabb_min[i]) / voxel_size[i]) for i in range(3)]
    else:
        if isinstance(grid_size, int):
            grid_size = [grid_size, grid_size, grid_size]
        grid_size = [int(x) for x in grid_size]

    # 3. Transform vertices to local AABB space
    vertices = [[v[i] - aabb_min[i] for i in range(3)] for v in mesh.vertices]
    triangles = [[vertices[f[i]] for i in range(3)] for f in mesh.faces]

    # 4. Compute occupied voxel indices via geometric intersection [G]
    # The sources specify a backend call (_C.intersect_occ_cuda or _C.intersect_occ_cpu) for this step.
    voxel_indices_mat = [] # Computed via backend intersection primitive

    # 5. Sort and align attributes
    vid_mat = [hash(tuple(voxel_indices_mat[i])) for i in range(len(voxel_indices_mat))]
    mapping_mat = sorted(range(len(vid_mat)), key=lambda i: vid_mat[i])
    attributes = {k: [v[m] for m in mapping_mat] for k, v in attributes.items()}

    return voxel_indices_mat, attributes
```

**Edge Cases and Pitfalls:**
- `voxel_size` must be on CPU for metadata operations to avoid implicit CUDA synchronization [S1, unofficial].
- If `grid_size` is derived from `aabb` and `voxel_size`, rounding errors may cause slight mismatches between the logical grid and physical bounds [S1, unofficial].
- Attribute alignment relies on `serialize.encode_seq` and `argsort`; incorrect serialization of voxel indices will lead to misaligned material properties [S1, unofficial].
## Subtopics

### [Transparent Illustration via Voxelization](4782-transparent-illustration-via-voxelization.md)
Transparent Illustration via Voxelization is a technique for rendering complex, deformable geometric models semi-transparently within volumetric scenes. It solves the visualization problem by storing multivalued data (normals, texture coordinates, material IDs) in a voxel worksheet and integrating this data into a slicing-based volume rendering pipeline. The method builds on [Textured Mesh to Volumetric Attribute Voxelization](2760-textured-mesh-to-volumetric-attribute-voxelization.md) by extending volumetric attribute formation to support multivalued storage for transparent illustration.

### [Trimesh Mesh Voxelization & Texture Color Mapping](6284-trimesh-mesh-voxelization-texture-color-mapping.md)
This method converts a polygonal triangle mesh into a 3D volumetric density field by marking voxels whose centers are in close proximity to the mesh surface, while simultaneously mapping vertex texture colors to those voxels. It differs from standard binary voxelization by integrating attribute creation (color/normal) directly during the voxelization process using barycentric projection or proximity queries, avoiding the need for post-hoc interpolation. Unlike [Surface-aligned Texture Field (TexF)](10067-surface-aligned-texture-field-texf.md) which aligns texture fields to surface normals, this method relies on geometric proximity and barycentric coordinate projection to assign attributes.

### [Surface-aligned Texture Field (TexF)](10067-surface-aligned-texture-field-texf.md)
Surface-aligned Texture Field (TexF) is a UV-free volumetric representation that associates texture attributes with voxels derived from a mesh surface, preserving spatial locality without UV parameterization. It solves the problem of transferring 2D texture data to 3D geometry by embedding attributes directly in a surface-aligned 3D field, allowing direct queries at mesh surface positions during rendering. Unlike UV-based pipelines that use chart-based unfolding and require UV coordinates/indices, TexF eliminates intermediate parameterization overhead. It builds on [Textured Mesh to Volumetric Attribute Voxelization](2760-textured-mesh-to-volumetric-attribute-voxelization.md) by specifically targeting surface-aligned sparse voxelization for texture storage and compression.

### Fire Profile Texture Mapping
A method to create a 3D cylindrical unit fire volume from a 2D artist-created texture, controlling color, shape, and intensity [S2, unofficial]. The mapping uses the formula $\tilde{uv} = (\sqrt{tex_x^2 + tex_y^2}, tex_z)$ to convert a 3D location $tex$ in the unit fire volume to a 2D positional vector sampled in the fire profile texture, then computes $color = \text{FireProfileTexture}(\tilde{uv})$ [S2, unofficial]. The unit fire volume has coordinates in the range $[-1, 1]$ for the horizontal axes $x$ and $y$, and $[0, 1]$ for the vertical axis $z$ [S2, unofficial].

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] PDF Real-time Procedural Volumetric Fire - UC Davis, https://web.cs.ucdavis.edu/~hamann/FullerKrishnanMahrousHamannJoyFirePaperFor_I3D2007AsSubmitted11012006.pdf, fetched 2026-10-27 (unofficial)
