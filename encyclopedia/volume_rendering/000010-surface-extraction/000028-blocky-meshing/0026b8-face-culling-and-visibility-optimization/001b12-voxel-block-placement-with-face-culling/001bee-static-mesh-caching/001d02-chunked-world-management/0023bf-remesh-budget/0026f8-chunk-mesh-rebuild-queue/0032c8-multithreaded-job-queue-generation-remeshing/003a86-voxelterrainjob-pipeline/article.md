# VoxelTerrainJob Pipeline

Also called: Chunk processing phases, Voxel chunk processing job system, VoxelPropJob execution

## Overview
The VoxelTerrainJob Pipeline is a multi-phase, resumable job system used in the Voxelman engine module to process voxel chunks [S1, unofficial]. It handles mesh generation, collider creation, lighting baking, and physics assignment by executing a sequence of defined phases [S1, unofficial]. It distinguishes itself from simpler meshing routines by its explicit support for yielding back to the main thread between phases (via `should_return()` checks) to prevent frame drops, and its integration with Godot's `PhysicsServer` and material caching systems [S1, unofficial].

## How it works
The pipeline processes a single `VoxelChunk` through a series of phases [S1, unofficial]. Each phase may be interrupted and resumed later using metadata stored on the chunk to track loop indices [S1, unofficial].

### Inputs and Outputs
*   **Input**: A `VoxelChunk` object containing voxel data, build flags (`BUILD_FLAG_CREATE_COLLIDER`, `BUILD_FLAG_USE_LIGHTING`), and a list of `VoxelMesher` and `VoxelLiquidMesher` instances [S1, unofficial].
*   **Output**: Updated chunk meshes, physics colliders (RID references), and baked vertex colors if lighting is enabled [S1, unofficial].

### Data Structures
*   **VoxelChunk**: Contains build flags, mesh data resources, and metadata keys for resumption [S1, unofficial].
    *   Metadata keys (strings):
        *   `tms_m`: Terrain mesh setup index.
        *   `tms_lm`: Liquid mesh setup index.
        *   `bpc_aa`: Collider build index (terrain).
        *   `bpc_laa`: Collider build index (liquid).
        *   `bptm_ulm`: Unlit terrain mesh index.
        *   `bptm_ullm`: Unlit liquid mesh index.
        *   `bptm_mm`: Merged terrain mesh index.
        *   `bptm_lmm`: Merged liquid mesh index.
*   **VoxelMesher**: Base class for meshing algorithms that follows a `SurfaceTool`-like interface (sets colors, UVs, calls `add_vertex`) [S1, unofficial].
*   **VoxelMesherJobStep**: Defines specific processing steps (e.g., `TYPE_NORMAL`, `TYPE_SIMPLIFY_MESH`) [S1, unofficial].

### Numbered Steps
The pipeline executes phases in the following order [S1, unofficial]. If `should_return()` is true at any point, the current phase saves its state to chunk metadata and returns control to the caller [S1, unofficial]. The next call resumes from the saved state [S1, unofficial].

1.  **Phase: Library Setup**
    *   Check `should_return()` [S1, unofficial].
    *   Get the `VoxelLibrary` from the chunk [S1, unofficial].
    *   If `lib->supports_caching()` is true and the chunk lacks a material cache key (`!_chunk->material_cache_key_has()`), generate the key via `lib->material_cache_get_key(_chunk)` [S1, unofficial].
    *   If a cache exists, retrieve it [S1, unofficial].
    *   While `!cache->get_initialized()`, yield by calling `OS::get_singleton()->delay_usec(100)` and checking `should_return()` again [S1, unofficial].
    *   Iterate over mesh data resources and update their UV rectangles using `cache->additional_texture_get_uv_rect(tex)` [S1, unofficial].
    *   Call `next_phase()` [S1, unofficial].

2.  **Phase: Terrain Mesh Setup**
    *   Resume index from meta `tms_m` (default 0) [S1, unofficial].
    *   Loop over `_meshers`:
        *   Call `mesher->add_chunk(_chunk)` [S1, unofficial].
        *   If `should_return()`, save index to `tms_m` and return [S1, unofficial].
    *   Repeat for `_liquid_meshers` using meta `tms_lm` [S1, unofficial].
    *   Remove metas `tms_m` and `tms_lm` [S1, unofficial].
    *   Call `next_phase()` [S1, unofficial].

3.  **Phase: Collider Build**
    *   If `(chunk->get_build_flags() & VoxelChunkDefault::BUILD_FLAG_CREATE_COLLIDER) == 0`, skip to next phase [S1, unofficial].
    *   Resume index from meta `bpc_aa` (default 0) [S1, unofficial].
    *   Loop over `_meshers`:
        *   Call `mesher->build_collider()` [S1, unofficial].
        *   Append result to `temp_arr_collider` [S1, unofficial].
        *   If `should_return()`, save index to `bpc_aa` and return [S1, unofficial].
    *   If `is_editor_hint()`:
        *   Resume index from meta `bpc_laa` (default 0) [S1, unofficial].
        *   Loop over `_liquid_meshers`:
            *   Append to `temp_arr_collider_liquid` [S1, unofficial].
            *   If `should_return()`, save index to `bpc_laa` and return [S1, unofficial].
    *   Remove metas `bpc_aa` and `bpc_laa` [S1, unofficial].
    *   If both `temp_arr_collider` and `temp_arr_collider_liquid` are empty, call `reset_stages()` and skip to next phase [S1, unofficial].
    *   Set build phase type to `BUILD_PHASE_TYPE_PHYSICS_PROCESS` [S1, unofficial].
    *   Call `next_phase()` [S1, unofficial].

4.  **Phase: Physics Process**
    *   If `temp_arr_collider` is not empty:
        *   If `!chunk->meshes_has(VoxelChunkDefault::MESH_INDEX_TERRAIN, VoxelChunkDefault::MESH_TYPE_INDEX_BODY)`, call `chunk->colliders_create(VoxelChunkDefault::MESH_INDEX_TERRAIN)` [S1, unofficial].
        *   Call `PhysicsServer::get_singleton()->shape_set_data(chunk->mesh_rid_get(VoxelChunkDefault::MESH_INDEX_TERRAIN, VoxelChunkDefault::MESH_TYPE_INDEX_SHAPE), temp_arr_collider)` [S1, unofficial].
        *   Clear `temp_arr_collider` [S1, unofficial].
    *   If in editor and `temp_arr_collider_liquid` is not empty, repeat the above for liquid colliders [S1, unofficial].
    *   Set build phase type to `BUILD_PHASE_TYPE_NORMAL` [S1, unofficial].
    *   Call `next_phase()` [S1, unofficial].

5.  **Phase: Terrain Mesh (Lighting & Merge)**
    *   If `(chunk->get_build_flags() & VoxelChunkDefault::BUILD_FLAG_USE_LIGHTING) != 0`:
        *   Resume index from meta `bptm_ulm` [S1, unofficial].
        *   Loop over meshers:
            *   Call `mesher->bake_colors(_chunk)` [S1, unofficial].
            *   If `should_return()`, save index to `bptm_ulm` and return [S1, unofficial].
        *   Resume index from meta `bptm_ullm` for liquid meshers [S1, unofficial].
        *   Resume index from meta `bptm_mm` [S1, unofficial].
        *   Merge meshers into a single mesher using `add_mesher`, assigning material index 0 [S1, unofficial].
        *   Repeat for liquids with meta `bptm_lmm` [S1, unofficial].
    *   If both merged meshers have 0 vertices:
        *   Clear metas [S1, unofficial].
        *   Call `reset_stages()` and skip to next phase [S1, unofficial].
    *   Call `next_phase()` [S1, unofficial].

6.  **Phase: Job Steps (Prop/Advanced Processing)**
    *   Validate prop mesher [S1, unofficial].
    *   If vertex count is 0, reset stages and complete [S1, unofficial].
    *   Count required meshes by iterating `_job_steps` and summing based on job type:
        *   `NORMAL`, `NORMAL_LOD`, `DROP_UV2`, `MERGE_VERTS`, `BAKE_TEXTURE`: add 1 [S1, unofficial].
        *   `SIMPLIFY_MESH`: add `step->get_simplification_steps()` [S1, unofficial].
    *   Allocate meshes via `chunk->meshes_create(...)` or clear existing ones [S1, unofficial].
    *   Loop through `job_steps`:
        *   Switch on `step->get_job_type()`:
            *   `TYPE_NORMAL`: Standard meshing [S1, unofficial].
            *   `TYPE_DROP_UV2`: Get mesh RID. Clear UV2 array (`temp_mesh_arr[VisualServer::ARRAY_TEX_UV2] = Variant()`). Add surface from arrays. Retrieve and set material [S1, unofficial].
            *   `TYPE_SIMPLIFY_MESH`: Requires `MESH_UTILS_PRESENT` macro [S1, unofficial].
            *   `TYPE_NORMAL_LOD`: Explicitly non-functional for `VoxelPropJob`; prints error [S1, unofficial].
        *   Increment `_current_job_step` [S1, unofficial].
        *   Check `should_return()` to yield [S1, unofficial].
    *   Reset stages and complete [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants and helper classes to make pseudocode self-contained
class VoxelChunkDefault:
    BUILD_FLAG_CREATE_COLLIDER = 1
    BUILD_FLAG_USE_LIGHTING = 2
    MESH_INDEX_TERRAIN = 0
    MESH_TYPE_INDEX_BODY = 0
    MESH_TYPE_INDEX_SHAPE = 1

class VoxelMesherJobStep:
    TYPE_NORMAL = 0
    TYPE_NORMAL_LOD = 1
    TYPE_DROP_UV2 = 2
    TYPE_MERGE_VERTS = 3
    TYPE_BAKE_TEXTURE = 4
    TYPE_SIMPLIFY_MESH = 5

class Mesher:
    def __init__(self):
        self.vertex_count = 0
        self.arrays = {}
        self.material_index = 0
    def add_chunk(self, chunk): pass
    def build_collider(self): return []
    def bake_colors(self, chunk): pass
    def add_mesher(self, other): self.vertex_count += other.vertex_count
    def set_material_index(self, idx): self.material_index = idx
    def get_vertex_count(self): return self.vertex_count

class OS:
    @staticmethod
    def delay_usec(us): pass
    @staticmethod
    def is_editor_hint(): return False

class PhysicsServer:
    @staticmethod
    def shape_set_data(rid, data): pass

class VisualServer:
    ARRAY_TEX_UV2 = 1
class Variant:
    pass

def process_chunk(chunk, job):
    phase = job.get_current_phase()

    # Phase 0: Library Setup
    if phase == 0:
        if not job.should_return():
            lib = chunk.get_library()
            if lib.supports_caching() and not chunk.has_material_cache_key():
                chunk.set_material_cache_key(lib.get_material_cache_key(chunk))
            cache = lib.get_material_cache(chunk)
            while not cache.is_initialized():
                if job.should_return():
                    return False
                OS.delay_usec(100)
            for tex in chunk.get_mesh_data_resources():
                cache.additional_texture_get_uv_rect(tex)
        job.next_phase()

    # Phase 1: Terrain Mesh Setup
    elif phase == 1:
        idx = chunk.get_meta('tms_m', 0)
        for i in range(idx, len(job.meshers)):
            job.meshers[i].add_chunk(chunk)
            if job.should_return():
                chunk.set_meta('tms_m', i)
                return False
        idx = chunk.get_meta('tms_lm', 0)
        for i in range(idx, len(job.liquid_meshers)):
            job.liquid_meshers[i].add_chunk(chunk)
            if job.should_return():
                chunk.set_meta('tms_lm', i)
                return False
        chunk.remove_meta('tms_m')
        chunk.remove_meta('tms_lm')
        job.next_phase()

    # Phase 2: Collider Build
    elif phase == 2:
        if not (chunk.get_build_flags() & VoxelChunkDefault.BUILD_FLAG_CREATE_COLLIDER):
            job.next_phase()
            return True
        job.temp_collider = []
        job.temp_liquid_collider = []
        idx = chunk.get_meta('bpc_aa', 0)
        for i in range(idx, len(job.meshers)):
            collider = job.meshers[i].build_collider()
            job.temp_collider.append(collider)
            if job.should_return():
                chunk.set_meta('bpc_aa', i)
                return False
        if OS.is_editor_hint():
            idx = chunk.get_meta('bpc_laa', 0)
            for i in range(idx, len(job.liquid_meshers)):
                collider = job.liquid_meshers[i].build_collider()
                job.temp_liquid_collider.append(collider)
                if job.should_return():
                    chunk.set_meta('bpc_laa', i)
                    return False
        chunk.remove_meta('bpc_aa')
        chunk.remove_meta('bpc_laa')
        if not job.temp_collider and not job.temp_liquid_collider:
            job.reset_stages()
            job.next_phase()
            return True
        job.set_build_phase_type("BUILD_PHASE_TYPE_PHYSICS_PROCESS")
        job.next_phase()

    # Phase 3: Physics Process
    elif phase == 3:
        if job.temp_collider:
            if not chunk.meshes_has(VoxelChunkDefault.MESH_INDEX_TERRAIN, VoxelChunkDefault.MESH_TYPE_INDEX_BODY):
                chunk.colliders_create(VoxelChunkDefault.MESH_INDEX_TERRAIN)
            rid = chunk.mesh_rid_get(VoxelChunkDefault.MESH_INDEX_TERRAIN, VoxelChunkDefault.MESH_TYPE_INDEX_SHAPE)
            PhysicsServer.shape_set_data(rid, job.temp_collider)
            job.temp_collider.clear()
        if OS.is_editor_hint() and job.temp_liquid_collider:
            if not chunk.meshes_has(VoxelChunkDefault.MESH_INDEX_TERRAIN, VoxelChunkDefault.MESH_TYPE_INDEX_BODY):
                chunk.colliders_create(VoxelChunkDefault.MESH_INDEX_TERRAIN)
            rid = chunk.mesh_rid_get(VoxelChunkDefault.MESH_INDEX_TERRAIN, VoxelChunkDefault.MESH_TYPE_INDEX_SHAPE)
            PhysicsServer.shape_set_data(rid, job.temp_liquid_collider)
            job.temp_liquid_collider.clear()
        job.set_build_phase_type("BUILD_PHASE_TYPE_NORMAL")
        job.next_phase()

    # Phase 4: Terrain Mesh (Lighting & Merge)
    elif phase == 4:
        merged_terrain_mesh = Mesher()
        merged_liquid_mesh = Mesher()
        if chunk.get_build_flags() & VoxelChunkDefault.BUILD_FLAG_USE_LIGHTING:
            idx = chunk.get_meta('bptm_ulm', 0)
            for i in range(idx, len(job.meshers)):
                job.meshers[i].bake_colors(chunk)
                if job.should_return():
                    chunk.set_meta('bptm_ulm', i)
                    return False
            idx = chunk.get_meta('bptm_ullm', 0)
            for i in range(idx, len(job.liquid_meshers)):
                job.liquid_meshers[i].bake_colors(chunk)
                if job.should_return():
                    chunk.set_meta('bptm_ullm', i)
                    return False
            idx = chunk.get_meta('bptm_mm', 0)
            for i in range(idx, len(job.meshers)):
                merged_terrain_mesh.add_mesher(job.meshers[i])
            merged_terrain_mesh.set_material_index(0)
            idx = chunk.get_meta('bptm_lmm', 0)
            for i in range(idx, len(job.liquid_meshers)):
                merged_liquid_mesh.add_mesher(job.liquid_meshers[i])
            merged_liquid_mesh.set_material_index(0)
        if merged_terrain_mesh.get_vertex_count() == 0 and merged_liquid_mesh.get_vertex_count() == 0:
            chunk.clear_metas()
            job.reset_stages()
            job.next_phase()
            return True
        job.next_phase()

    # Phase 5: Job Steps
    elif phase == 5:
        prop_mesher = job.get_prop_mesher()
        if prop_mesher and prop_mesher.get_vertex_count() == 0:
            job.reset_stages()
            return True
        total_steps = 0
        for step in job.job_steps:
            jtype = step.get_job_type()
            if jtype in [VoxelMesherJobStep.TYPE_NORMAL, VoxelMesherJobStep.TYPE_NORMAL_LOD,
                         VoxelMesherJobStep.TYPE_DROP_UV2, VoxelMesherJobStep.TYPE_MERGE_VERTS,
                         VoxelMesherJobStep.TYPE_BAKE_TEXTURE]:
                total_steps += 1
            elif jtype == VoxelMesherJobStep.TYPE_SIMPLIFY_MESH:
                total_steps += step.get_simplification_steps()
        chunk.meshes_create(total_steps)
        current_step_idx = 0
        for step in job.job_steps:
            jtype = step.get_job_type()
            if jtype == VoxelMesherJobStep.TYPE_NORMAL:
                pass # Standard meshing logic handled by mesher
            elif jtype == VoxelMesherJobStep.TYPE_DROP_UV2:
                temp_mesh_arr = step.get_arrays()
                temp_mesh_arr[VisualServer.ARRAY_TEX_UV2] = Variant()
                step.add_surface_from_arrays(temp_mesh_arr)
                step.set_material(step.get_material())
            elif jtype == VoxelMesherJobStep.TYPE_SIMPLIFY_MESH:
                pass # Requires MESH_UTILS_PRESENT macro
            elif jtype == VoxelMesherJobStep.TYPE_NORMAL_LOD:
                print("TYPE_NORMAL_LOD not supported for VoxelPropJob")
            current_step_idx += 1
            if job.should_return():
                job.set_current_step(current_step_idx)
                return False
        job.reset_stages()
        return True
    return True
```

### Parameters
*   `BUILD_FLAG_CREATE_COLLIDER`: Bit flag on `VoxelChunkDefault`. Controls whether collider generation is enabled [S1, unofficial].
*   `BUILD_FLAG_USE_LIGHTING`: Bit flag on `VoxelChunkDefault`. Controls whether vertex color baking is performed [S1, unofficial].
*   `MESH_UTILS_PRESENT`: Preprocessor macro. Enables `TYPE_SIMPLIFY_MESH` job step support and includes `fast_quadratic_mesh_simplifier.h` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Threaded Cache Initialization**: If the material cache is being merged on another thread, the job must wait. This is handled by polling `cache->get_initialized()` with a 100µs delay loop [S1, unofficial].
*   **Empty Mesh Generation**: If both terrain and liquid merged meshers have 0 vertices, the pipeline skips mesh allocation and physics setup by clearing metas, resetting stages, and advancing phases [S1, unofficial].
*   **Job Resumption**: Long-running phases use meta keys to store loop indices. On return, the index is saved; on restart, it's loaded. Metas are removed after completion [S1, unofficial].
*   **Non-functional LOD Step**: `step_type_normal_lod` is explicitly non-functional for `VoxelPropJob` and prints an error instead of processing [S1, unofficial].
*   **Physics API Differences**: Godot 3 uses `body_create(PhysicsServer::BODY_MODE_STATIC)`, while Godot 4 requires `body_create()` followed by `body_set_mode(body, PhysicsServer::BODY_MODE_STATIC)` [S1, unofficial].

## Subtopics

### [VoxelMesher](52-voxel-mesher.md)
Base class for meshing algorithms.

### [VoxelChunk](53-voxel-chunk.md)
Storage for terrain data, prop data, and mesh generation jobs.

### [VoxelLibrary](54-voxel-library.md)
Manages material caching and texture UV rectangles.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Not given in the sources.
*   **Strengths**: Prevents frame drops by yielding between phases; integrates tightly with Godot's physics and rendering servers; supports complex post-processing steps (simplification, UV2 baking) [S1, unofficial].
*   **Limitations**: Complex state management via metadata; Godot version-specific API calls; `TYPE_NORMAL_LOD` is non-functional [S1, unofficial].
*   **When to use**: When building voxel terrain in a real-time engine (Godot) where frame rate stability is critical and physics integration is required [S1, unofficial].
*   **When not to use**: For offline mesh generation where yieldability is unnecessary, or for non-Godot engines [S1, unofficial].

## Background
The VoxelTerrainJob Pipeline is part of the Voxelman engine module for Godot, developed by Péter Magyar (Relintai) [S1, unofficial]. It was designed to provide editor integration, gameplay-related features, and extendability from GDScript without sacrificing speed [S1, unofficial]. It relies on the `thread_pool` module for multi-threaded chunk generation [S1, unofficial].

## Sources
[S1] Relintai/voxelman, https://github.com/Relintai/voxelman, fetched 2026-10-26 (unofficial)
