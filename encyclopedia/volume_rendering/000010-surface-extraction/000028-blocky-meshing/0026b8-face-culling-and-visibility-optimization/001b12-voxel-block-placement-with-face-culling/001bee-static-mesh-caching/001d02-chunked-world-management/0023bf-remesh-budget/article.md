# Remesh Budget

Also called: RemeshQueue, VoxelBodyRendererBudgetedRemeshing

## Overview
Remesh Budget is a rendering optimization that processes dirty chunks nearest the camera first within a strict time budget per frame [S1]. It prevents main-thread stalls by limiting the number of sub-chunks remeshed per frame and capping the total time spent on geometry generation [S2, unofficial]. Unlike immediate synchronous rebuilding, which causes frame drops during rapid block placement, this method decouples modification detection from geometry generation [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A set of "dirty" chunks (sub-chunks containing modified voxel data) and the current camera position.
- **Output:** Updated GPU mesh data for a subset of dirty chunks, marked as "clean" in the world state.

**Data Structures:**
- **RemeshQueue:** A collection of chunk indices ordered by distance to the camera (nearest first) [S1].
- **ChunkMeshData:** Contains `Vertices` (span of vertices), `Indices` (span of indices), `VertexCount`, and `IndexCount` [S2, unofficial].
- **GpuMesh:** Constructed from `data[i].Vertices` and `data[i].Indices` [S2, unofficial].
- **Frame State:**
  - `_rebuildsThisFrame`: Integer counter for chunks processed in the current frame [S2, unofficial].
  - `_frameStarted`: Timestamp recorded at the start of the frame using `System.Diagnostics.Stopwatch.GetTimestamp()` [S2, unofficial].
  - `MaxRebuildsPerFrame`: Constant integer limit (typically 24) [S2, unofficial].
  - `RebuildBudgetMs`: Constant double limit (typically 6.0 ms) [S2, unofficial].
  - `FrameBudget`: Time budget for remeshing per frame (typically 3 ms) [S1].

**Parameters:**
- `MaxRebuildsPerFrame`: 24 [S2, unofficial].
- `RebuildBudgetMs`: 6.0 ms [S2, unofficial].
- `FrameBudget`: 3 ms [S1].
- **Note:** One sub-chunk always goes through even if the budget is exceeded [S2, unofficial].

**Steps:**
1. **BeginFrame:** Reset `_rebuildsThisFrame` to 0 and record `_frameStarted` timestamp [S2, unofficial].
2. **Prepare:**
   - If a pending task exists and is not complete, return false [S2, unofficial].
   - If the pending task is complete, call `FinishPending` [S2, unofficial].
   - Collect indices of all dirty chunks [S2, unofficial].
   - If no dirty chunks exist, return true [S2, unofficial].
   - Create a task on a worker thread to build meshes for dirty chunks using a local `MeshBuilder` [S2, unofficial].
   - Return false [S2, unofficial].
3. **RebuildWithinBudget:**
   - Iterate through all chunk indices in the `RemeshQueue` (ordered nearest to camera first) [S1].
   - For each chunk:
     - If the chunk is dirty:
       - Check if `RebuildAllowed` is true.
       - If allowed:
         - Increment `_rebuildsThisFrame` [S2, unofficial].
         - Call `Rebuild` for the chunk [S2, unofficial].
         - Mark the chunk as clean [S2, unofficial].
       - If not allowed:
         - Set `complete` to false and continue [S2, unofficial].
     - If the chunk is not dirty, skip [S2, unofficial].
   - Return true if all dirty chunks were processed [S2, unofficial].
4. **RebuildAllowed:**
   - Returns true if `_rebuildsThisFrame` < `MaxRebuildsPerFrame` AND (`_rebuildsThisFrame` == 0 OR elapsed time since `_frameStarted` < `RebuildBudgetMs`) [S2, unofficial].
5. **FinishPending:**
   - If the pending task is not completed, return false [S2, unofficial].
   - If completed, clear the pending task [S2, unofficial].
   - If the pending mode differs from the current mode, return true (meshes are useless) [S2, unofficial].
   - Otherwise, upload each chunk's data and mark chunks clean [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class RemeshBudget:
    def __init__(self, max_rebuilds_per_frame=24, rebuild_budget_ms=6.0, frame_budget_ms=3.0):
        self.max_rebuilds_per_frame = max_rebuilds_per_frame
        self.rebuild_budget_ms = rebuild_budget_ms
        self.frame_budget_ms = frame_budget_ms
        self.rebuilds_this_frame = 0
        self.frame_started = 0
        self.pending_task = None
        self.dirty_chunks = []

    def begin_frame(self):
        self.rebuilds_this_frame = 0
        self.frame_started = get_timestamp()

    def is_rebuild_allowed(self):
        if self.rebuilds_this_frame >= self.max_rebuilds_per_frame:
            return False
        if self.rebuilds_this_frame > 0:
            elapsed_ms = get_elapsed_ms(self.frame_started)
            if elapsed_ms >= self.rebuild_budget_ms:
                return False
        return True

    def prepare(self):
        if self.pending_task is not None:
            if not self.pending_task.is_complete():
                return False
            self.finish_pending()
        
        self.dirty_chunks = collect_dirty_chunk_indices()
        if len(self.dirty_chunks) == 0:
            return True
        
        # Sort by distance to camera
        self.dirty_chunks.sort(key=lambda c: distance_to_camera(c))
        
        self.pending_task = create_worker_task(build_meshes, self.dirty_chunks)
        return False

    def rebuild_within_budget(self):
        if self.pending_task is None:
            return True
        
        if not self.pending_task.is_complete():
            return False
        
        self.finish_pending()
        return True

    def finish_pending(self):
        if self.pending_task is None:
            return
        
        if not self.pending_task.is_complete():
            return False
        
        # Check mode switch
        if self.pending_task.mode != current_mode:
            self.pending_task = None
            return True
        
        # Upload data
        for chunk_index in self.pending_task.chunk_indices:
            upload_chunk_data(chunk_index)
            mark_chunk_clean(chunk_index)
        
        self.pending_task = None
        return True

    def process_frame(self):
        self.begin_frame()
        
        # Always process at least one chunk if there are dirty chunks
        if len(self.dirty_chunks) > 0:
            self.prepare()
        
        # Process chunks within budget
        while len(self.dirty_chunks) > 0 and self.is_rebuild_allowed():
            chunk_index = self.dirty_chunks.pop(0)
            if is_chunk_dirty(chunk_index):
                rebuild_chunk(chunk_index)
                mark_chunk_clean(chunk_index)
                self.rebuilds_this_frame += 1
        
        return self.pending_task is None or self.pending_task.is_complete()
```

**Edge Cases and Pitfalls:**
- **Mode Switch:** Changing `TerrainMode` from Blocks to Smooth or vice versa marks all bodies dirty, triggering full remeshing over subsequent frames [S2, unofficial].
- **Precision:** Timestamps should use high-resolution counters (e.g., `Stopwatch.GetTimestamp()`) to accurately measure sub-millisecond budgets [S2, unofficial].
- **Boundaries:** The first chunk is always processed regardless of budget limits to ensure responsiveness [S2, unofficial].

## Subtopics

### [Chunk Mesh Rebuild Queue](3911-chunk-mesh-rebuild-queue.md)
A queue-based system to batch and process chunk mesh rebuilds asynchronously or on tick. It solves the problem of redundant mesh regeneration by ensuring each modified chunk is processed exactly once per frame or tick cycle. Unlike immediate synchronous rebuilding, which can cause frame drops during rapid block placement, this method decouples modification detection from geometry generation.

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of dirty chunks, but bounded by the time budget [S1].
- **Memory Complexity:** O(M) where M is the number of chunks in the remesh queue [S1].
- **Measured Performance:** In the Outpost scene (4 chunks, 10k triangles), the system handles 37 draws/200k triangles on Low, 114 draws/879k on Balanced, and 153 draws/1.87M on Ultra [S1].
- **Scaling:** Performance scales with the number of dirty chunks and the time budget [S1].
- **Strengths:** Prevents main-thread stalls, ensures smooth frame rates during rapid block placement [S2, unofficial].
- **Limitations:** May leave some dirty chunks unprocessed in a single frame if the budget is exceeded [S2, unofficial].
- **When to Use:** Ideal for dynamic voxel worlds with frequent block placement/removal [S2, unofficial].
- **When Not to Use:** Not suitable for static worlds where all meshes can be generated upfront [S2, unofficial].

## Background
Remesh Budget originated in the context of voxel-based sandbox games, specifically in projects like *Survivor Builder* and *VoxelTerraformer* [S1][S2, unofficial]. It was developed to address the performance issues associated with synchronous mesh generation in dynamic voxel environments [S1][S2, unofficial]. The method is used in games that feature real-time terrain modification and require consistent frame rates [S1][S2, unofficial].

## Sources
[S1] Draw the blocks: beveled chunk meshing, tier tiles, remesh budget (Phase 2c), https://github.com/austinbryanprice-droid/Claude-Survivor-Builder/pull/8, fetched 2026-10-26
[S2] GhostlyActive/VoxelTerraformer, https://github.com/GhostlyActive/VoxelTerraformer, fetched 2026-10-27 (unofficial)
