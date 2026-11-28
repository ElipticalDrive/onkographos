# Surface Nets Mesh Generation

Also called: Fast Surface Nets, Marching Cubes Variant, Signed Distance Field Meshing via Surface Nets, Smooth Mesh from Density Field

## Overview
Surface Nets generates a smooth triangle mesh from a binary density field or Signed Distance Field (SDF) by computing vertex positions at voxel centers based on local gradients, producing significantly fewer vertices and smoother geometry than blocky voxel-based methods [S1, S2, unofficial]. It is implemented as a drop-in dependency (`fast-surface-nets` v0.x) for extracting meshes from voxel SDFs, chosen for its simplicity and performance [S1, unofficial].

## How it works
**Inputs:**
1. Density field $d$ (3D array of scalar values) [S2, unofficial].
2. Iso-value threshold (typically $0.5$ for binary density, $0.0$ for SDF) [S2, unofficial].
3. Optional bounding box lower corner `bounding_box.lo` for world-space translation [S2, unofficial].
4. Optional material data and chunk dimensions for post-processing [S3, unofficial].

**Outputs:**
1. Triangular mesh with vertex positions, indices, and optionally normals [S2, unofficial].
2. Post-processed mesh attributes including material blending, face culling, bounds calculation, and collider octants [S3, unofficial].

**Data Structures:**
- Density Field: 3D array where each cell $(i, j, k)$ holds a scalar value [S2, unofficial].
- Mesh: Standard triangle list format with vertex buffers (position, normal) and index buffers [S2, unofficial].

**Algorithm Steps:**

1. **Core Generation:**
   Call the generation routine with the density field and iso-value.
   $$ \text{mesh} = \text{SurfaceNets::GenerateMesh}(d, 0.5) $$
   Typical iso-value is $0.5$ for binary fields [S2, unofficial].

2. **Vertex Position Calculation (Core Logic):**
   For each voxel crossing the iso-surface, compute the vertex position at the voxel center by averaging the positions of the 8 neighboring grid points weighted by their distance to the iso-surface [G].

3. **World Space Translation:**
   Translate all generated mesh vertices to world space by adding the lower corner of the bounding box to each vertex position.
   $$ v_{\text{world}} = v_{\text{local}} + \text{bounding\_box.lo} $$
   This ensures the mesh is correctly positioned relative to the scene origin [S2, unofficial].

4. **Post-Processing (Optional but Common):**
   If the mesh contains indices, apply post-processing to handle material blending and collision detection.
   $$ \text{prepMesh}(\text{mesh}, \text{chunk\_coords}) $$
   This step handles:
   * Material blending based on adjacent voxel materials [S3, unofficial].
   * Face culling to remove internal faces [S3, unofficial].
   * Bounds calculation [S3, unofficial].
   * Generation of collider octants for physics engines [S3, unofficial].

5. **Data Transfer:**
   Transfer the generated buffers (vertices, indices, normals) and any auxiliary data (SDF, material buffers, octant buffers) back to the main thread or rendering pipeline [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_surface_nets_mesh(density_field, iso_value=0.5, bounding_box_lo=None, material_data=None):
    """
    Generates a smooth triangle mesh from a density field using Surface Nets.
    """
    # Step 1: Core mesh generation
    mesh = surface_nets_core_generate(density_field, iso_value)
    
    # Step 2: World space translation
    if bounding_box_lo is not None:
        for vertex in mesh['vertices']:
            vertex['position'] = (
                vertex['position'][0] + bounding_box_lo[0],
                vertex['position'][1] + bounding_box_lo[1],
                vertex['position'][2] + bounding_box_lo[2]
            )
            
    # Step 3: Post-processing (if indices and material data are present)
    if mesh['has_indices'] and material_data is not None:
        prep_mesh(mesh, material_data)
        
    # Step 4: Data transfer (returns processed mesh buffers)
    return mesh

def surface_nets_core_generate(density_field, iso_value):
    """
    Computes vertex positions at voxel centers weighted by distance to iso-surface [G].
    """
    vertices = []
    indices = []
    shape = density_field.shape
    for i in range(shape[0] - 1):
        for j in range(shape[1] - 1):
            for k in range(shape[2] - 1):
                corners = [
                    density_field[i, j, k], density_field[i+1, j, k],
                    density_field[i, j+1, k], density_field[i+1, j+1, k],
                    density_field[i, j, k+1], density_field[i+1, j, k+1],
                    density_field[i, j+1, k+1], density_field[i+1, j+1, k+1]
                ]
                if not (any(c < iso_value for c in corners) and any(c >= iso_value for c in corners)):
                    continue
                total_w = 0.0
                pos = [0.0, 0.0, 0.0]
                coords = [
                    (i, j, k), (i+1, j, k), (i, j+1, k), (i+1, j+1, k),
                    (i, j, k+1), (i+1, j, k+1), (i, j+1, k+1), (i+1, j+1, k+1)
                ]
                for coord, val in zip(coords, corners):
                    w = abs(val - iso_value)
                    total_w += w
                    pos[0] += coord[0] * w
                    pos[1] += coord[1] * w
                    pos[2] += coord[2] * w
                if total_w > 0:
                    pos[0] /= total_w
                    pos[1] /= total_w
                    pos[2] /= total_w
                    vertices.append({'position': pos})
                    indices.append(len(vertices) - 1)
    return {'vertices': vertices, 'indices': indices, 'has_indices': len(indices) > 0}

def prep_mesh(mesh, material_data):
    """
    Post-processes the mesh for material blending and collision.
    Handles material blending, face culling, bounds calculation, and collider octants [S3, unofficial].
    """
    pass
```

**Edge Cases and Pitfalls:**
* **Precision:** Floating-point precision in density field sampling can affect vertex placement accuracy [S1, unofficial].
* **Boundaries:** Ensure the density field is padded or handled correctly at grid boundaries to avoid artifacts [S2, unofficial].
* **Degenerate Input:** Empty or fully occupied fields should return empty meshes or full volumes appropriately [S1, unofficial].
* **Sharp Features:** The method cannot preserve sharp edges (perfect cliffs, voxel-style cubes); output is always smooth [S1, unofficial].

## Operating characteristics
* **Performance:** Targets ~20M triangles/second on a single core, accelerated by glam SIMD [S1, unofficial].
* **Memory Complexity:** Linear with respect to the number of vertices and triangles generated, which is typically less than voxel-based methods [S1, unofficial].
* **Strengths:** Efficiency, smoothness, and compatibility with SDF data [S1, S2, unofficial].
* **Limitations:** Loses sharp edges; output is always smooth [S1, unofficial].
* **When to Use:** Suitable for organic caves, smooth terrain, and organic destruction, covering 90% of the use case [S1, unofficial].

## Background
Surface Nets is implemented via the crate `fast-surface-nets` v0.x by bonsairobo, adopted as a drop-in dependency [S1, unofficial]. It is chosen as the primary path for extracting meshes from voxel SDFs due to its simplicity and performance [S1, unofficial]. A follow-up custom displacement correction (DC) path is planned for zones requiring sharp features like rocks or voxel-cube destruction [S1, unofficial].

## Sources
[S1] feat(world,mesh): Mesh extraction from voxel SDF — fast-surface-nets primary + custom DC follow-up, https://github.com/lobinuxsoft/kooch/issues/393, fetched 2026-10-26 (unofficial)
[S2] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S3] sh1ftmaker/voxel-wars, https://github.com/sh1ftmaker/voxel-wars, fetched 2026-10-28 (unofficial)
