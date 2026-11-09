# Chunk Mesh Generation
Also called: Chunk meshing, Greedy Mesher, Chunk Mesh Extraction, Exterior Face Mesh Generation

## Overview
Chunk Mesh Generation is a surface extraction technique that constructs a vertex buffer for a voxel container or chunk by iterating over its voxels and determining face visibility based on the existence and properties of neighboring voxels [S1, S4, unofficial]. It solves the problem of excessive polygon counts in voxel worlds by skipping interior faces, generating geometry only for exterior (exposed) faces where the adjacent voxel is transparent or non-existent [S4, unofficial]. Unlike static greedy meshing, this method often incorporates dynamic zone management, recalculating merge boundaries based on player movement and configurable distance thresholds to avoid unnecessary rebuilds [S3, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A voxel container or chunk defined by dimensions ($W, H, D$) and a block size.
- A voxel data structure where each voxel has properties such as `IsTransparent` (boolean) and `IsActive` (boolean) [S1, S4, unofficial].
- Player position (for dynamic zone updates) [S3, unofficial].
- Configuration parameters: `minMeshDistance` (minimum distance from player to trigger optimization; 0 disables) [S3, unofficial].

**Outputs:**
- A vertex buffer containing positions, colors, normals, and visibility states [S1, unofficial].
- A visibility index map linking voxels to their vertex ranges [S1, unofficial].

### Data Structures
The implementation relies on specific buffers and structures:
- **Vertex Buffers:**
  - `_vertices`: `std::vector<GLfloat>` for vertex positions [S1, unofficial].
  - `_colors`: `std::vector<GLfloat>` for vertex colors [S1, unofficial].
  - `_normals`: `std::vector<GLfloat>` for vertex normals [S1, unofficial].
  - `_visibilities`: `std::vector<GLfloat>` for per-vertex active states (1.0f or 0.0f) [S1, unofficial].
- **Indexing:**
  - `_visibilityIndexes`: A map from `std::shared_ptr<IVoxel>` to a struct containing `start` and `end` indices, tracking the range of vertices for each voxel in the `_visibilities` buffer [S1, unofficial].
- **Chunk/Section State (Dynamic Greedy Meshing):**
  - `sectionStates`: `BlockState[CELLS]` where $CELLS = 16 \times 16 \times 16 = 4096$ [S3, unofficial].
  - `faceMergeKeys`: `long[6][CELLS]` for merge keys per face direction [S3, unofficial].
  - `scratchCorners`: `float[12]` [S3, unofficial].
  - `scratchLighting`: `GreedyLighting.Scratch` [S3, unofficial].
  - `scratchTintPos`: `BlockPos.MutableBlockPos` [S3, unofficial].
  - `scratchCornerBlocks`: `int[4][3]` [S3, unofficial].
- **Face Structure:**
  - A `Face` structure containing three `Vertex` references (`v1`, `v2`, `v3`) [S2, unofficial].

### Algorithm Steps
1. **Initialization and Zone Update (Dynamic Mode):**
   - Retrieve `minMeshDistance` from configuration [S3, unofficial].
   - Calculate player chunk coordinates $(playerX, playerZ)$ by shifting block coordinates right by 4 bits ($>> 4$) [S3, unofficial].
   - Store old zone coordinates $(oldX, oldZ, oldMinDistance)$ [S3, unofficial].
   - Update zone coordinates to current player position and distance [S3, unofficial].
   - If the zone is uninitialized, distance is zero, distance has not changed, or the player hasn't moved chunks, return early without marking sections dirty [S3, unofficial].
   - Calculate `wasNear` and `isNear` using Chebyshev distance (max of absolute x/z differences) [S3, unofficial].
   - Skip processing if the status hasn't flipped or if the section is outside the effective render distance [S3, unofficial].

2. **Iterative Passes (Dynamic Mode):**
   - Perform two passes:
     - Pass 0 centers on $(oldX, oldZ)$.
     - Pass 1 centers on $(playerX, playerZ)$.
   - For each pass, loop $x$ and $z$ from $center - minDistance$ to $center + minDistance$ [S3, unofficial].
   - On Pass 1, skip coordinates that fall within the $minDistance$ radius of the old position to avoid double-processing overlapping regions [S3, unofficial].

3. **World Offset Calculation:**
   - Compute world-space offsets for the chunk:
     $$ xOffset = chunkX \times IChunk::Width $$
     $$ yOffset = chunkY \times IChunk::Height $$
     $$ zOffset = chunkZ \times IChunk::Depth $$
   [S1, unofficial].

4. **Voxel Iteration and Neighbor Check:**
   - Iterate over each voxel in the container using `forEachVoxel`, yielding local coordinates $(x, y, z)$ [S1, unofficial].
   - Calculate world coordinates for the current voxel:
     $$ worldX = x + xOffset $$
     $$ worldY = y + yOffset $$
     $$ worldZ = z + zOffset $$
   [S1, unofficial].
   - Check the six neighboring voxels (left, right, top, bottom, front, rear) [S1, unofficial].
   - Determine neighbor validity:
     - If a neighbor's world coordinate is at the boundary (0 or max-1), set it to `nullptr` [S1, unofficial].
     - Otherwise, if the local coordinate is at the container boundary, fetch from `voxelWorld`; else fetch from `voxelContainer` [S1, unofficial].
   - Determine render flag: `needToRender` is true if any of the six neighbors is `nullptr` [S1, unofficial].
   - If a neighbor voxel has `IsTransparent` set to false, do not create a face for the current voxel in the direction of that neighbor [S4, unofficial].
   - Generate mesh data only for faces where the adjacent voxel is transparent or non-existent (exterior) [S4, unofficial].

5. **Mesh Construction:**
   - If `needToRender` is true, call `VoxelGeometry::AddVerticesAndColors` with the voxel and world coordinates to populate vertices, colors, and normals vectors [S1, unofficial].
   - Calculate the number of newly added vertices:
     $$ numberOfNewVertices = (newSize - priorSize) / 3 $$
   [S1, unofficial].
   - Push visibility values to the `_visibilities` buffer: for each new vertex, push `voxel->getIsActive() ? 1.0f : 0.0f` twice [S1, unofficial].
   - Store the visibility index range for the voxel in `_visibilityIndexes[voxel]` using `priorVisibilitySize` and `visibilites.size() - 1` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk_mesh(chunk, player_pos, config):
    # 1. Zone Update Logic (Dynamic Greedy Meshing)
    min_mesh_distance = config.minMeshDistance
    player_chunk_x = player_pos.block_x >> 4
    player_chunk_z = player_pos.block_z >> 4
    
    old_x, old_z, old_min_dist = get_old_zone_coords()
    
    # Check if zone update is needed
    if zone_uninitialized or distance_zero or distance_unchanged or player_not_moved_chunks:
        return # Early exit, no rebuild
    
    # Calculate Chebyshev distance for zone status
    was_near = chebyshev_distance(old_x, old_z, player_chunk_x, player_chunk_z) <= min_mesh_distance
    is_near = chebyshev_distance(player_chunk_x, player_chunk_z, player_chunk_x, player_chunk_z) <= min_mesh_distance # Simplified logic based on notes
    
    if status_not_flipped or section_outside_render_distance:
        return

    # 2. World Offset Calculation
    width = chunk.width
    height = chunk.height
    depth = chunk.depth
    chunk_x, chunk_y, chunk_z = chunk.coordinates
    
    x_offset = chunk_x * width
    y_offset = chunk_y * height
    z_offset = chunk_z * depth
    
    # Initialize buffers
    vertices = []
    colors = []
    normals = []
    visibilities = []
    visibility_indexes = {}
    
    # 3. Iterative Passes (if dynamic)
    passes = []
    if dynamic_mode:
        passes.append({'center_x': old_x, 'center_z': old_z})
        passes.append({'center_x': player_chunk_x, 'center_z': player_chunk_z})
    else:
        passes.append({'center_x': player_chunk_x, 'center_z': player_chunk_z})

    for pass_idx, pass_info in enumerate(passes):
        center_x = pass_info['center_x']
        center_z = pass_info['center_z']
        
        # Loop range based on minMeshDistance
        start_x = center_x - min_mesh_distance
        end_x = center_x + min_mesh_distance
        start_z = center_z - min_mesh_distance
        end_z = center_z + min_mesh_distance
        
        for x in range(start_x, end_x + 1):
            for z in range(start_z, end_z + 1):
                # Skip overlap on pass 1
                if pass_idx == 1 and chebyshev_distance(x, z, old_x, old_z) <= min_mesh_distance:
                    continue
                    
                # 4. Voxel Iteration and Neighbor Check
                # Note: The notes describe iterating voxels in the chunk. 
                # The pass logic applies to which chunks/sections are processed.
                # Here we assume the loop above selects the chunks, and we iterate voxels within.
                
                for local_x, local_y, local_z in chunk.iterate_voxels():
                    world_x = local_x + x_offset
                    world_y = local_y + y_offset
                    world_z = local_z + z_offset
                    
                    # Check 6 neighbors
                    need_to_render = False
                    for direction in [LEFT, RIGHT, TOP, BOTTOM, FRONT, REAR]:
                        neighbor = get_neighbor(world_x, world_y, world_z, direction)
                        
                        # Boundary check
                        if neighbor is None or neighbor.is_at_boundary():
                            neighbor = None # Treat as empty/transparent
                        
                        if neighbor is None:
                            need_to_render = True
                        elif not neighbor.is_transparent:
                            # Neighbor is opaque, face is hidden
                            continue
                        else:
                            # Neighbor is transparent, face is visible
                            need_to_render = True
                    
                    # 5. Mesh Construction
                    if need_to_render:
                        # Add vertices for the exposed face(s)
                        prior_size = len(vertices)
                        add_vertices_and_colors(vertices, colors, normals, local_x, local_y, local_z, world_x, world_y, world_z)
                        new_size = len(vertices)
                        
                        # Calculate new vertices count
                        # Assuming 4 vertices per quad face, but notes say /3 for triangles?
                        # Notes say: numberOfNewVertices = (newSize - priorSize) / 3
                        # This implies vertices are stored as floats, 3 per vertex.
                        num_new_vertices = (new_size - prior_size) / 3
                        
                        # Push visibility
                        active_state = 1.0 if voxel.is_active() else 0.0
                        for _ in range(num_new_vertices):
                            visibilities.append(active_state)
                            visibilities.append(active_state) # Push twice per vertex [S1, unofficial]
                        
                        # Store index range
                        start_idx = len(visibilities) - num_new_vertices * 2 # Adjust based on push logic
                        end_idx = len(visibilities) - 1
                        visibility_indexes[voxel] = {'start': start_idx, 'end': end_idx}

    return vertices, colors, normals, visibilities, visibility_indexes

def get_neighbor(world_x, world_y, world_z, direction):
    # Implementation depends on world/voxel container access
    # If at boundary, return None
    # Else fetch from voxelWorld or voxelContainer
    pass
```

### Parameters
- `minMeshDistance`: Configurable minimum distance from player to trigger greedy meshing. Value 0 disables the optimization [S3, unofficial].
- `IsTransparent`: A boolean property of a voxel. If false, it blocks face generation for adjacent voxels [S4, unofficial].
- `prunePendingTicks`: Internal tick counter for periodic cleanup. Resets to 40 after each execution [S3, unofficial].

### Edge Cases and Pitfalls
- **Visibility Buffer Mismatch:** The visibility buffer pushes the same active state value twice per vertex (`visibilites.push_back` twice), which may indicate a layout mismatch or a bug in the vertex count calculation [S1, unofficial].
- **Render Distance Aliasing:** Sections outside render distance are not built; dirtying them would alias onto whichever loaded section currently occupies that render grid slot. This is handled by checking `Math.max(...) > renderDistance` and skipping [S3, unofficial].
- **Unloaded Sections:** No backend provides an unload callback. Sections scrolling out of render distance would otherwise linger in totals forever. This is handled by periodic pruning every 40 ticks with a radius of `renderDistance + 1` for edge slack [S3, unofficial].
- **Water Rendering Bugs:** Known issue where some merged water surfaces render black/missing faces on Sodium, likely due to CPU-side geometry being correct but draw data issues [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Zone update is $O(minDistance^2 \times verticalSections)$ per player movement event, bounded by render distance [S3, unofficial]. Pruning is $O(sections \ in \ radius)$ every 40 ticks [S3, unofficial].
- **Memory Complexity:** Depends on the number of exposed faces. The dynamic mode uses scratch buffers (`scratchCorners`, `scratchLighting`, etc.) and section arrays (`sectionStates`, `faceMergeKeys`) [S3, unofficial].
- **Strengths:** Reduces polygon count significantly by skipping interior faces [S4, unofficial]. Dynamic zone management avoids unnecessary rebuilds when the player moves within the same zone [S3, unofficial].
- **Limitations:** Potential for rendering artifacts with water or CTM blocks if not handled correctly [S3, unofficial]. The visibility buffer push logic may be buggy [S1, unofficial].
- **When to Use:** Suitable for voxel worlds where performance is critical and the player moves frequently, allowing for dynamic updates [S3, unofficial].
- **When Not to Use:** If static geometry is sufficient or if the overhead of dynamic zone management outweighs the benefits (e.g., very small worlds) [S3, unofficial].

## Background
Chunk Mesh Generation is a fundamental technique in voxel engine development, originating from the need to efficiently render large voxel worlds like those in Minecraft. It evolved from simple face culling to more sophisticated greedy meshing algorithms that merge adjacent faces into larger quads [S3, unofficial]. The method is implemented in various engines and mods, including the Fabric mod 'Greedy Meshing', which integrates with Minecraft's SectionCompiler and Sodium renderer [S3, unofficial]. It differs from static greedy meshing by dynamically recalculating the merge zone boundary based on player movement [S3, unofficial].

## Sources
[S1] JamesRandall/SimpleVoxelEngine, https://github.com/JamesRandall/SimpleVoxelEngine, fetched 2026-10-26 (unofficial)
[S2] nnarain/VoxelEngine, https://github.com/nnarain/VoxelEngine, fetched 2026-10-27 (unofficial)
[S3] Greedy Meshing - GitHub, https://github.com/programmer1o1/GreedyMeshingMod, fetched 2026-10-27 (unofficial)
[S4] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
