# Asynchronous Chunk Generation Scheduling

Also called: LoadChunkFromList, chunk loading queue, pending chunk processor

## Overview
Asynchronous Chunk Generation Scheduling controls the rate at which pending voxel chunks are processed from a queue to prevent frame-rate drops during world generation. It solves the problem of synchronous mesh generation blocking the main thread by limiting the number of chunks generated per frame. It differs from its parent [Asynchronous Chunk Loading](4216-asynchronous-chynchronous-chunk-loading.md), which offloads work to a thread pool, by performing the generation logic directly on the calling thread (typically the main thread) but throttling the iteration count. It differs from siblings [Dependency Queue Gating](11597-dependency-queue-gating.md) and [Web Worker Pool for Meshing](14649-web-worker-pool-for-meshing.md) by using a simple percentage-based cap rather than dependency resolution or separate worker threads.

## How it works
The method processes a list of pending chunk positions, validating regions and generating voxel data or retrieving saved data. It supports both synchronous (all-at-once) and asynchronous (throttled) modes.

**Inputs and Outputs:**
- Input: `pendingChunks` (list of chunk identifiers), `asyncChunkGeneration` (boolean flag).
- Output: Updated `pendingChunks` list (processed items removed), generated chunk meshes/voxels added to the scene graph or data store.

**Data Structures:**
- `pendingChunks`: An array of strings representing chunk coordinates in the format `'x|y|z'` [S2, unofficial].
- `chunkDict`: A dictionary/map storing generated chunks, keyed by chunk position [S1, unofficial].
- `REGION_SIZE`: Integer constant defining the size of a region in chunks [S1, unofficial].
- `CHUNK_SIDE`: Integer constant defining the side length of a chunk in world units [S1, unofficial].

**Parameters:**
- `asyncChunkGeneration`: Boolean. If `false`, processes all pending chunks. If `true`, processes a fraction [S2, unofficial].
- `throttleFactor`: Fixed at `0.1` (10%) when `asyncChunkGeneration` is `true` [S2, unofficial].
- `minChunksPerFrame`: Fixed at `1` [S2, unofficial].
- `maxChunksPerFrame`: Equal to `pendingChunks.length` [S2, unofficial].

**Steps:**
1. If `pendingChunks` is empty, return immediately [S1, unofficial].
2. Determine the number of chunks to process (`count`):
   - If `asyncChunkGeneration` is `false`: `count = pendingChunks.length` [S2, unofficial].
   - If `asyncChunkGeneration` is `true`: `count = floor(pendingChunks.length * 0.1)` [S2, unofficial].
   - Clamp `count` such that `1 <= count <= pendingChunks.length` [S2, unofficial].
3. Iterate `i` from `0` to `count - 1`:
   a. Retrieve the chunk identifier string from `pendingChunks[i]` [S2, unofficial].
   b. Parse the string by splitting on `'|'` to obtain coordinate strings `x_str`, `y_str`, `z_str` [S2, unofficial].
   c. Convert coordinates to integers: `cx = int(x_str)`, `cy = int(y_str)`, `cz = int(z_str)` [S2, unofficial]. Note: Sources use bitwise OR `|0` for conversion, which is equivalent to truncation for positive integers [S2, unofficial].
   d. Calculate the region position:
      $$ regionX = \lfloor cx / REGION\_SIZE \rfloor $$
      $$ regionY = \lfloor cy / REGION\_SIZE \rfloor $$
      [S1, unofficial]
   e. Determine if the chunk is the first in its region (index 0):
      $$ chunkIndexInRegion = cx \pmod{REGION\_SIZE} $$
      [S1, unofficial]
   f. Generate or load chunk data:
      - If `chunkIndexInRegion != 0`: Load chunk data from the region's stored data [S1, unofficial].
      - If `chunkIndexInRegion == 0`: Generate new voxel data using `noiseManager.GenerateChunkData(key)` [S1, unofficial].
   g. Create the chunk object (e.g., a GameObject with MeshFilter and MeshRenderer) [S1, unofficial].
   h. Set the chunk's world position:
      $$ position.x = cx \times CHUNK\_SIDE $$
      $$ position.y = 0 $$
      $$ position.z = cy \times CHUNK\_SIDE $$
      [S1, unofficial]
   i. Add the chunk to `chunkDict` and remove it from `pendingChunks` [S1, unofficial].
   j. If the system is a client (`isClient`), call `showChunk(chunk)` to render it [S2, unofficial].
4. After the loop, remove the processed chunks from the front of `pendingChunks` using `splice(0, count)` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_chunk_queue(pending_chunks, async_chunk_generation, region_size, chunk_side, noise_manager, is_client):
    if not pending_chunks:
        return

    # Determine count based on async mode
    if not async_chunk_generation:
        count = len(pending_chunks)
    else:
        # 10% of pending, clamped between 1 and total length
        count = int(len(pending_chunks) * 0.1)
        count = max(1, min(count, len(pending_chunks)))

    # Process up to 'count' chunks
    for i in range(count):
        # Parse chunk position string 'x|y|z'
        chunk_str = pending_chunks[i]
        parts = chunk_str.split('|')
        
        # Convert to integers (bitwise OR 0 ensures int conversion in JS, equivalent here)
        cx = int(parts[0])
        cy = int(parts[1])
        cz = int(parts[2])

        # Calculate region position
        region_x = int(cx / region_size)
        region_y = int(cy / region_size)

        # Determine if this is the first chunk in the region
        chunk_index_in_region = cx % region_size

        # Generate or load data
        if chunk_index_in_region != 0:
            # Load from region data (implementation detail omitted in source, assumed available)
            chunk_data = load_from_region_data(region_x, region_y, cx, cy, cz)
        else:
            # Generate new data for the region start
            chunk_data = noise_manager.generate_chunk_data(cx, cy, cz)

        # Create chunk object
        chunk = create_chunk_object(chunk_data)

        # Set position
        chunk.position.x = cx * chunk_side
        chunk.position.y = 0
        chunk.position.z = cy * chunk_side

        # Add to dictionary and remove from queue
        chunk_dict[cx, cy, cz] = chunk
        pending_chunks.pop(0) # Remove processed chunk

        # Render if client
        if is_client:
            show_chunk(chunk)

    # Note: The loop above processes one by one and removes. 
    # The source [S2, unofficial] describes splicing after the loop. 
    # To match source logic exactly where splice(0, count) is called after loop:
    # We must assume the loop processes items but doesn't remove them individually, 
    # or the source implies the removal happens in batch. 
    # Re-reading S2: "If count > 0, remove processed chunks from the front... via splice(0, count)".
    # This implies the loop iterates but the removal is batched at the end.
    # However, S1 says "remove from queue" inside the step. 
    # Merging: The loop processes the chunks. The removal is explicitly batched in S2.
    # Let's adjust pseudocode to match S2's explicit splice instruction at the end.
    
    # Correction for strict adherence to S2's structure:
    # The loop processes chunks. The removal is done via splice at the end.
    # But S1 says "remove from queue" inside. 
    # Conflict: S1 removes inside, S2 removes via splice after.
    # Resolution: S2 is the more detailed algorithmic description for the queue management.
    # We will follow S2's batch removal but ensure the loop processes the correct items.
    
    # Re-writing loop to not remove individually, then splice.
    pass # See final consolidated pseudocode below

def process_chunk_queue_final(pending_chunks, async_chunk_generation, region_size, chunk_side, noise_manager, is_client):
    if not pending_chunks:
        return

    if not async_chunk_generation:
        count = len(pending_chunks)
    else:
        count = int(len(pending_chunks) * 0.1)
        count = max(1, min(count, len(pending_chunks)))

    # Process 'count' chunks
    for i in range(count):
        chunk_str = pending_chunks[i]
        parts = chunk_str.split('|')
        cx = int(parts[0])
        cy = int(parts[1])
        cz = int(parts[2])

        region_x = int(cx / region_size)
        region_y = int(cy / region_size)
        chunk_index_in_region = cx % region_size

        if chunk_index_in_region != 0:
            chunk_data = load_from_region_data(region_x, region_y, cx, cy, cz)
        else:
            chunk_data = noise_manager.generate_chunk_data(cx, cy, cz)

        chunk = create_chunk_object(chunk_data)
        chunk.position.x = cx * chunk_side
        chunk.position.y = 0
        chunk.position.z = cy * chunk_side

        chunk_dict[cx, cy, cz] = chunk

        if is_client:
            show_chunk(chunk)

    # Batch removal as per S2
    if count > 0:
        del pending_chunks[0:count]
```

**Edge Cases and Pitfalls:**
- If `count` is 0 (e.g., empty list or logic error), no chunks are processed, preventing UI freezes [S2, unofficial].
- Chunk positions are stored as strings; parsing must handle the `'|'` delimiter correctly [S2, unofficial].
- Integer conversion must be robust; using bitwise OR `|0` in JavaScript ensures integer truncation [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(k)$ per frame, where $k$ is the number of chunks processed per frame. $k$ is either $N$ (all pending) or $\lceil 0.1 N \rceil$ (throttled), where $N$ is the total pending chunks.
- **Memory Complexity:** $O(1)$ additional memory for the scheduling logic, excluding the memory required for the generated chunk meshes themselves.
- **Performance:** Prevents frame drops by limiting generation work per frame. When `asyncChunkGeneration` is true, it ensures smooth UI updates even with large backlogs [S2, unofficial].
- **Strengths:** Simple to implement; effective at decoupling generation rate from frame rate on the main thread.
- **Limitations:** Still runs on the main thread, so heavy generation can cause stutter if the per-frame cost of a single chunk is high. Does not handle dependencies between chunks (unlike Dependency Queue Gating).
- **When to use:** When main-thread generation is acceptable but needs rate-limiting to maintain responsiveness.
- **When not to use:** When generation is too heavy for the main thread (use Asynchronous Chunk Loading or Web Worker Pool) or when chunks depend on neighbors being generated first (use Dependency Queue Gating).

## Background
This method is derived from implementations in voxel engine frameworks, specifically noted in the Unity-based Marching Cubes implementation [S1, unofficial] and the JavaScript voxel-engine-stackgl project [S2, unofficial]. It represents a pragmatic approach to managing world generation load in real-time applications where full asynchronous offloading is not required or feasible.

## Sources
[S1] Javier-Garzo/Marching-cubes-on-Unity-3D, https://github.com/Javier-Garzo/Marching-cubes-on-Unity-3D, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
