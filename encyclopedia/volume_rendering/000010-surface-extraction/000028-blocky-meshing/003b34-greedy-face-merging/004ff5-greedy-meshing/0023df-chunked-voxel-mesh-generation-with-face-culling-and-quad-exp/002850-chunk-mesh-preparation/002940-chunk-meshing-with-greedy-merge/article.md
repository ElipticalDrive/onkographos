# Chunk Meshing with Greedy Merge

Also called: Greedy Mesher, meshChunk

## Overview
Chunk Meshing with Greedy Merge is a surface extraction method that generates optimized geometry for voxel chunks by merging adjacent coplanar faces of identical texture into maximal rectangles [S1, S2, unofficial]. It solves the problem of excessive vertex counts and draw calls inherent in naive per-block meshing by reducing geometry complexity through a greedy face-merging approach [S1, S2, unofficial]. Unlike sibling methods that may use quad expansion or simple culling, this method specifically targets maximal rectangular merging of opaque faces to minimize rendering overhead [S2, unofficial].

## How it works
**Inputs:** A `ChunkView` containing block data, fluid levels, and neighbor references; a `MeshConfig` defining block types (fluids, plants, transparent solids); and pre-processed chunk data from [Chunk Mesh Preparation](4265-chunk-mesh-preparation.md).
**Outputs:** Optimized `BufferGeometry` (or equivalent mesh structure) containing merged vertices and indices.

**Data Structures:**
*   `ChunkView`: Contains `blocks` (compact storage, e.g., `Uint16Array`), `fluid` (falling, levels, sources), and chunk coordinates [S1, S2, unofficial].
*   `MeshConfig`: Defines `fluidMaxLevels`, `waterBlockIds`, `transparentSolidBlockIds`, and `crossPlantBlockIds` [S2, unofficial].
*   `ChunkMeshContext`: Internal structure holding `fluids`, `layers`, `lookup`, `plants`, `push`, and `yLimit` [S2, unofficial].
*   Work Buffers: `mask` and `light masks` allocated based on chunk dimensions [S2, unofficial].

**Steps:**
1.  **Validate Chunk Views:** Ensure the input chunk data is valid and consistent [S2, unofficial].
2.  **Retrieve Block Lookup:** Access the special block lookup table required for identifying unique block behaviors [S2, unofficial].
3.  **Build Context:** Call `chunkMeshContextOf` to construct the `ChunkMeshContext`. This step internally builds lookups for layers, fluids, and cross-plants, and meshes special blocks and fluid surfaces [S2, unofficial].
4.  **Extract Context Data:** Retrieve `fluids`, `layers`, `lookup`, `plants`, `push`, and `yLimit` from the context [S2, unofficial].
5.  **Check Empty Ceiling:** If `yLimit` equals `EMPTY_CHUNK_CEILING`, return the empty layers immediately to skip unnecessary processing [S2, unofficial].
6.  **Calculate Buffer Size:** Determine the required buffer length using the formula:
    $$ \text{bufferLength} = \text{CHUNK\_SIZE} \times \max(\text{yLimit}, \text{CHUNK\_SIZE}) $$
    [S2, unofficial]
7.  **Allocate Buffers:** Use `meshBuffersOf` to allocate work buffers, including masks and light masks, sized according to the calculated buffer length [S2, unofficial].
8.  **Mesh All Faces:** Call `meshAllFaces` with the chunk, fluids, light, lookup, mask, neighbors, plants, special lookup, push, and yLimit. This function performs the core greedy merging algorithm [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunk_meshing_with_greedy_merge(chunk_view: ChunkView, config: MeshConfig) -> BufferGeometry:
    # Step 1: Validate chunk views
    if not is_valid_chunk(chunk_view):
        raise ValueError("Invalid chunk view")

    # Step 2: Retrieve special block lookup
    lookup = get_special_block_lookup(config)

    # Step 3: Build chunk mesh context
    context = chunk_mesh_context_of(chunk_view, config, lookup)

    # Step 4: Extract context data
    fluids = context.fluids
    layers = context.layers
    plants = context.plants
    push = context.push
    y_limit = context.y_limit

    # Step 5: Check for empty chunk ceiling
    if y_limit == EMPTY_CHUNK_CEILING:
        return layers  # Return empty layers immediately

    # Step 6: Calculate required buffer length
    buffer_length = CHUNK_SIZE * max(y_limit, CHUNK_SIZE)

    # Step 7: Allocate work buffers
    mask, light_masks = mesh_buffers_of(buffer_length)

    # Step 8: Mesh all faces
    # Note: meshAllFaces performs the greedy merge logic
    geometry = mesh_all_faces(
        chunk=chunk_view,
        fluids=fluids,
        light=chunk_view.light,
        lookup=lookup,
        mask=mask,
        neighbors=chunk_view.neighbors,
        plants=plants,
        special_lookup=lookup.special,
        push=push,
        y_limit=y_limit
    )

    return geometry
```

**Parameters:**
*   `CHUNK_SIZE`: Typically 16x16x16 or similar fixed dimensions [S1, unofficial].
*   `EMPTY_CHUNK_CEILING`: A constant indicating an empty chunk [S2, unofficial].
*   `fluidMaxLevels`: Defined in `MeshConfig`, e.g., 7 for water [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Chunks:** Handled by checking `yLimit` against `EMPTY_CHUNK_CEILING` [S2, unofficial].
*   **Fluid Levels:** Fluids require specific handling for levels and sources, which are processed in the context building phase [S2, unofficial].
*   **Precision:** Compact storage (e.g., `Uint16Array`) is used for block data, which may limit block ID ranges [S1, unofficial].

## Subtopics

### Chunk Mesh Preparation
Pre-processes chunk data, handling visibility culling against neighbors before meshing [4265-chunk-mesh-preparation.md].

### Greedy Meshing
The core algorithm that merges coplanar faces into maximal rectangles [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in sources, but implied to be more efficient than naive per-block meshing due to face merging [S2, unofficial].
*   **Memory Complexity:** Requires work buffers sized by `CHUNK_SIZE * max(yLimit, CHUNK_SIZE)` [S2, unofficial].
*   **Performance:** Benchmarks indicate significant reduction in vertex count and draw calls compared to naive approaches [S2, unofficial].
*   **Strengths:** Reduces rendering overhead by minimizing geometry complexity [S1, S2, unofficial].
*   **Limitations:** Relies on greedy merging, which may not always produce optimal geometry in all scenarios [S2, unofficial].

## Background
The method originates from implementations in voxel engine projects such as `blockbound` and `mc-meshing`, where it is used to optimize rendering performance by reducing the number of draw calls and vertices [S1, S2, unofficial]. It is commonly employed in browser-based voxel engines using Web Workers for off-thread meshing [S1, unofficial].

## Sources
[S1] Phase 2: voxel engine com chunks, meshing e edição, https://github.com/EduardoLavall/blockbound/issues/4, fetched 2026-10-26 (unofficial)
[S2] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
