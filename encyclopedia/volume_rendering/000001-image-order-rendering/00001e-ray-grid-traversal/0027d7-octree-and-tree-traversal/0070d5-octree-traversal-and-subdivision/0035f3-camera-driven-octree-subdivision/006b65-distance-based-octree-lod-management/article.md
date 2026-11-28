# Distance-Based Octree LOD Management

Also called: Chunk Splitting and Merging

## Overview
Distance-Based Octree LOD Management optimizes rendering load by dynamically subdividing or merging octree chunks based on the camera's distance to the chunk center [S1, unofficial]. It splits chunks into eight children when the camera is close and merges them back when the camera is far, serving as the core subdivision logic for [Camera-Driven Octree Subdivision](7636-camera-driven-octree-subdivision.md). Unlike [Octree Texture Subdivision](13613-octree-texture-subdivision.md), which manages LOD via texture mipmaps, this method explicitly creates and destroys geometric child nodes.

## How it works
**Inputs and Outputs:**
*   **Input:** Camera position, chunk center position, chunk edge size, current LOD level, and a global semaphore state.
*   **Output:** Creation of 8 child chunks (splitting) or restoration of the parent chunk (merging).

**Data Structures:**
*   **Chunk:** Contains `edgeSize` (float), `lodLevel` (int), `isSplitting` (bool), `isRestoringParent` (bool), `mesh` (reference), and `childrenReady` (int, count of active children).
*   **DCManager:** Contains `greenLight` (bool semaphore), `maxLodLevel` (int), and `diagonal` (float, derived from edge size).
*   **Child Naming:** Suffixes `|DBR`, `|DBL`, `|DTR`, `|DTL`, `|UBR`, `|UBL`, `|UTR`, `|UTL` are appended to chunk names to identify orientation (Down/Up, Back/Front, Right/Left).

**Formulas:**
*   **Diagonal:** The face diagonal length of the chunk, used as the primary distance threshold.
    $$ \text{diagonal} = \sqrt{2 \cdot (\text{edgeSize}^2)} $$
    [S1, unofficial]

**Parameters:**
*   **maxLodLevel:** The maximum depth of subdivision allowed. [S1, unofficial]
*   **Split Threshold:** `distance < diagonal` [S1, unofficial]
*   **Merge Threshold:** `distance > diagonal * 3.0` [S1, unofficial]

**Steps:**
1.  **Calculate Distance:** Compute the Euclidean distance from the camera position to the chunk center.
2.  **Evaluate Split Condition:**
    *   If `distance < diagonal` AND `lodLevel < maxLodLevel` AND `childrenReady < 8` AND `!isSplitting` AND `!isRestoringParent` AND `mesh != null`:
        *   Check global semaphore `DCManager.greenLight`. If false, abort.
        *   Set `DCManager.greenLight = false`.
        *   Set `isSplitting = true`.
        *   Call `splitCube()`.
3.  **Evaluate Merge Condition:**
    *   If `lodLevel > 0` AND `distance > diagonal * 3.0` AND `mesh != null` AND `!parent.isRestoringParent`:
        *   If the chunk name ends with `|DBR` (indicating it is a specific child type requiring parent restoration):
            *   Set `parent.isRestoringParent = true`.
            *   Call `restoreParentCube()`.
4.  **Split Cube (`splitCube`):**
    *   Calculate `delta = edgeSize / 4`.
    *   Create 8 child chunks.
    *   Position each child at the parent's position offset by `±delta` in x, y, and z axes.
    *   Assign appropriate suffixes (`|DBR`, etc.) to child names.
    *   Increment `childrenReady` on the parent.
5.  **Restore Parent Cube (`restoreParentCube`):**
    *   Calculate `delta = edgeSize / 2`.
    *   Create the parent cube at `transform.position + delta` with size `edgeSize * 2` and `lodLevel - 1`.
    *   Clear `edgeDictionary` and `edgeDictionaryOrphans` on the parent.
    *   Reset `isRestoringParent` to false.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_lod(chunk, camera_pos, manager):
    # Calculate distance from chunk center to camera
    distance = vector_length(camera_pos - chunk.position)
    
    # Calculate diagonal threshold
    diagonal = math.sqrt(2 * (chunk.edgeSize ** 2))
    
    # Check for splitting
    if (distance < diagonal and 
        chunk.lodLevel < manager.max_lod_level and 
        chunk.children_ready < 8 and 
        not chunk.is_splitting and 
        not chunk.is_restoring_parent and 
        chunk.mesh is not None):
        
        # Serialize split operations
        if not manager.green_light:
            return
        
        manager.green_light = False
        chunk.is_splitting = True
        split_cube(chunk)

    # Check for merging
    elif (chunk.lodLevel > 0 and 
          distance > diagonal * 3.0 and 
          chunk.mesh is not None and 
          chunk.parent is not None and 
          not chunk.parent.is_restoring_parent):
        
        # Only specific children trigger parent restoration
        if chunk.name.endswith("|DBR"):
            chunk.parent.is_restoring_parent = True
            restore_parent_cube(chunk.parent)

def split_cube(chunk):
    delta = chunk.edgeSize / 4.0
    offsets = [
        (-delta, -delta, -delta), (delta, -delta, -delta),
        (-delta, delta, -delta), (delta, delta, -delta),
        (-delta, -delta, delta), (delta, -delta, delta),
        (-delta, delta, delta), (delta, delta, delta)
    ]
    suffixes = ["|DBR", "|DBL", "|DTR", "|DTL", "|UBR", "|UBL", "|UTR", "|UTL"]
    
    for i, offset in enumerate(offsets):
        new_pos = chunk.position + vector(offset)
        new_chunk = create_chunk(
            position=new_pos,
            size=chunk.edgeSize / 2.0,
            lod_level=chunk.lodLevel + 1,
            name=chunk.name + suffixes[i],
            manager=chunk.manager
        )
        chunk.children.append(new_chunk)
    
    chunk.children_ready = 8
    chunk.is_splitting = False
    chunk.manager.green_light = True

def restore_parent_cube(parent_chunk):
    delta = parent_chunk.edgeSize / 2.0
    new_pos = parent_chunk.transform.position + delta
    new_size = parent_chunk.edgeSize * 2.0
    new_lod = parent_chunk.lod_level - 1
    
    restored_chunk = create_chunk(
        position=new_pos,
        size=new_size,
        lod_level=new_lod,
        name=parent_chunk.name,
        manager=parent_chunk.manager
    )
    
    # Clear edge data structures
    restored_chunk.edge_dictionary.clear()
    restored_chunk.edge_dictionary_orphans.clear()
    
    parent_chunk.is_restoring_parent = False
```

**Edge Cases and Pitfalls:**
*   **Race Conditions:** Splitting and merging are blocked by `isSplitting` and `isRestoringParent` flags to prevent concurrent modification [S1, unofficial].
*   **Serialization:** A global `DCManager.greenLight` semaphore serializes split operations across all chunks to prevent resource contention [S1, unofficial].
*   **Merge Trigger:** Merging is only triggered by chunks ending in `|DBR`, implying a specific traversal or ownership model for parent restoration [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Splitting and merging are $O(1)$ relative to the chunk count, but involve $O(1)$ object creation/destruction. The global semaphore introduces potential contention under high concurrency.
*   **Memory Complexity:** Memory usage scales with the number of visible chunks. Splitting increases memory by 8x for the affected region; merging decreases it.
*   **Strengths:** Simple distance-based heuristic; effective for reducing draw calls in distant areas.
*   **Limitations:** The global semaphore can bottleneck performance in highly parallel scenes. The merge condition is rigid (`diagonal * 3.0`) and may cause popping if not smoothed.
*   **When to use:** Suitable for static or slowly changing scenes where camera distance is the primary LOD factor.
*   **When not to use:** High-frequency camera movement may cause excessive split/merge churn.

## Background
This method is implemented in the `proceduralplanet` project by paolorussian, which uses it for dynamic terrain generation and rendering [S1, unofficial]. It is part of a camera-driven subdivision system where geometric chunks are explicitly managed rather than using texture-based LOD.

## Sources
[S1] paolorussian/proceduralplanet, https://github.com/paolorussian/proceduralplanet, fetched 2026-10-26 (unofficial)
