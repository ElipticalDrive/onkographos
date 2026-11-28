# Tree LOD Chain with Instancing

Also called: Canonical crowns, Tree LOD stages

## Overview
A multi-stage Level of Detail (LOD) system for volumetric trees, chosen by projected size per tree or chunk, ranging from full voxels to baked heightfields, using instancing for distant stages to reduce CPU overhead [S1, unofficial]. It sets itself apart from discrete mesh variants by unifying LOD selection into a scalar function of camera distance and employing canonical crowns for mid-range instancing, ensuring consistent visual transitions between window trees and horizon instances [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Tree voxel data (DF bounds, species, seed), camera position, camera FOV, window resolution.
- Output: Rendered tree geometry (meshes or heightfield contributions) with appropriate LOD stage.

**Data Structures:**
- `TreeEntity`: Contains `species_id`, `seed`, `df_bounds` (min/max coordinates), `visibility_range` (tuple of distances for stage transitions).
- `SpeciesData`: Contains list of `canonical_crowns` (meshes) and `variants` (per-voxel-stage variants).
- `LODStage`: Enum or integer identifying the current detail level (1-5).

**Parameters:**
- `DWARF_EYE_LOD_NEAR`: Override for the near band edge distance [S1, unofficial].
- `DWARF_EYE_OCCLUSION`: Flag to disable GPU occlusion culling on the camera (0 to disable) [S1, unofficial].
- `FOV`: Horizontal field of view in radians.
- `WindowHeight`: Height of the viewport in pixels.
- `TileSize`: Size of a voxel chunk/tile in world units.
- `BANDS`: Number of LOD bands (typically 3-5) [S1, unofficial].

**Steps:**
1. **Calculate Projected Size:** For each tree, calculate the projected size of its bounding box or chunk on the screen. The band is chosen by projected tile size on screen (pixels per tile at that distance), not raw distance [S1, unofficial].
   - Formula for near band edge $N$ (in blocks/tiles):
     $$ N = \frac{h}{2 \cdot 2 \cdot \text{px} \cdot \tan(\text{fov}/2)} \cdot 0.25 \cdot \text{tile} $$
     Where $h$ is window height, $\text{px}$ is pixels per unit (derived from FOV and width), and $\text{tile}$ is tile size [S1, unofficial].
   - If `DWARF_EYE_LOD_NEAR` is set, use it to override the near band edge calculation [S1, unofficial].

2. **Determine LOD Stage:** Based on the projected size or distance, assign one of 5 stages:
   - Stage 1: Full voxels with cutout leaves.
   - Stage 2: Quarter-resolution voxels with opaque leaves, greedy-merged into one mesh per chunk.
   - Stage 3: One canonical crown per species, a few dozen triangles, instanced.
   - Stage 4: A green box, twelve triangles, instanced.
   - Stage 5: Nothing per tree; canopy colour and height baked into the coarse heightfield.

3. **Mesh Generation/Selection:**
   - For Stages 1 and 2: Generate or retrieve voxel meshes. Stage 2 meshes are quarter-resolution and opaque, merged per chunk [S1, unofficial].
   - For Stage 3: Select a canonical crown from `SpeciesData` based on species. Scale it to the tree's DF height and radius [S1, unofficial].
   - For Stage 4: Use a simple green box mesh (12 triangles) [S1, unofficial].
   - For Stage 5: No mesh is generated for the tree; its contribution is handled by the heightfield [S1, unofficial].

4. **Instancing:**
   - For Stages 3 and 4, use GPU instancing. Replace per-tree rasters with a few dozen variants per species, instanced and clipped to each tree's DF bounds [S1, unofficial].
   - Identical mesh+material entities batch in the engine (e.g., Bevy) [S1, unofficial].

5. **Crossfading:**
   - Use `VisibilityRange` per entity to manage transitions between stages. Apply dithered crossfade between stages to avoid visible popping artifacts [S1, unofficial].
   - Band edges are one map over `BANDS` to allow for smooth transitions [S1, unofficial].

6. **Occlusion Culling:**
   - Use engine-side GPU occlusion culling (e.g., Bevy's `OcclusionCulling` with `DepthPrepass`) if `DWARF_EYE_OCCLUSION` is not set to disable it [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_tree(tree: TreeEntity, camera: Camera, window: Window) -> RenderCommand:
    # Step 1: Calculate projected size and determine LOD stage
    fov = camera.fov
    h = window.height
    px_per_unit = h / (2 * tan(fov / 2)) # Simplified projection factor
    tile_size = tree.df_bounds.width # Assuming square tiles for simplicity
    
    # Calculate projected tile size in pixels
    projected_tile_size = tile_size * px_per_unit / tree.distance
    
    # Determine LOD stage based on projected size
    # Note: Exact thresholds depend on implementation, using N as reference
    N = (h / (2 * 2 * px_per_unit * tan(fov / 2))) * 0.25 * tile_size # In blocks
    
    if tree.distance < N:
        stage = 1
    elif tree.distance < 2 * N:
        stage = 2
    elif tree.distance < 4 * N:
        stage = 3
    elif tree.distance < 8 * N:
        stage = 4
    else:
        stage = 5
        
    # Override if DWARF_EYE_LOD_NEAR is set
    if DWARF_EYE_LOD_NEAR is not None:
        if tree.distance < DWARF_EYE_LOD_NEAR:
            stage = 1
            
    # Step 2 & 3: Get or generate mesh
    if stage == 1:
        mesh = get_full_voxel_mesh(tree.species_id, tree.seed)
        material = cutout_material()
    elif stage == 2:
        mesh = get_quarter_voxel_mesh(tree.species_id, tree.seed)
        material = opaque_material()
    elif stage == 3:
        # Select canonical crown
        crown = get_canonical_crown(tree.species_id)
        # Scale to DF bounds
        scale = tree.df_bounds.height / crown.base_height
        mesh = scale_mesh(crown, scale)
        material = species_material(tree.species_id)
    elif stage == 4:
        mesh = green_box_mesh()
        material = green_material()
    else: # Stage 5
        return HeightfieldContribution(tree.df_bounds, tree.species_id)
        
    # Step 4: Instancing for stages 3 and 4
    if stage in [3, 4]:
        return InstancedRenderCommand(mesh, material, tree.df_bounds.center, tree.df_bounds.scale)
    else:
        return MeshRenderCommand(mesh, material, tree.df_bounds.center, tree.df_bounds.scale)
```

**Edge Cases and Pitfalls:**
- Instancing at full detail saves worker CPU, upload, and VRAM, but not frame time; GPU cost is per triangle and pixel, and near trees are already merged per chunk. Frame time is cut by reducing cutout overdraw and triangle count [S1, unofficial].
- Transparency holds to about 13.6 blocks at 720p [S1, unofficial].
- Quake-style precomputed visibility does not fit an open, growing world [S1, unofficial].

## Subtopics

### [L-System Detail Scalar LOD](20525-l-system-detail-scalar-lod.md)
Replaces discrete stages with a continuous detail scalar for smooth crossfading.

## Operating characteristics
- **Time Complexity:** Mesh generation is on-demand from disk tiles, so it depends on tile access time. Instancing reduces CPU overhead for distant trees.
- **Memory Complexity:** Held triangles increase with more bands (e.g., ~40% for extra band), but drawn triangles decrease significantly [S1, unofficial].
- **Performance:** Measured on a 3000-chunk forest at 720p: 24.5 M triangles drawn down to 4.2 M; FPS unchanged because 720p is not geometry bound [S1, unofficial]. User measurement at their window size: 120+ fps with near/mid bands, up from 19 fps with 32.6 M triangles [S1, unofficial].
- **Strengths:** Reduces triangle count and overdraw for distant trees. Instancing improves CPU efficiency for mid-range trees.
- **Limitations:** Geometry bound scenes may not see FPS improvements if the bottleneck is pixel shading. Transparency handling has limits (approx. 13.6 blocks at 720p) [S1, unofficial].

## Background
Originated from discussions and implementations in the Dwarf Eye engine, specifically addressing tree rendering optimization in large open worlds [S1, unofficial]. The approach was refined through user feedback and engine-side development, integrating with Bevy's rendering capabilities [S1, unofficial].

## Sources
[S1] Simplify cached fine chunks far from the camera into a mid-detail surface, https://github.com/aaronsb/dwarf-eye/issues/10, fetched 2026-10-26 (unofficial)
