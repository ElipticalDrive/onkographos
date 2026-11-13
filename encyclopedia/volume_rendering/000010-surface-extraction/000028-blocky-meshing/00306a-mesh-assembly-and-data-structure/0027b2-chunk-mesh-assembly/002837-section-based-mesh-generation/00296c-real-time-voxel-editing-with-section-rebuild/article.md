# Real-time Voxel Editing with Section Rebuild

Also called: Incremental Section Mesh Update, Live Voxel Editing & Re-meshing, Section Rebuild

## Overview
This method enables immediate visual feedback for voxel placement and removal by regenerating mesh geometry only for the affected 16×16×16 section and any adjacent sections if the edit occurs on a section boundary [S1, unofficial]. It solves the latency issue of full-chunk regeneration by limiting updates to the immediate neighborhood of a block change, ensuring real-time responsiveness during voxel editing [S1, unofficial]. Unlike full rebuilds, it relies on boundary detection to determine which neighboring sections require recalculation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The 3D coordinates $(i, j, k)$ of the modified voxel within the chunk, the new `BlockType` (e.g., AIR, STONE), and the chunk's internal voxel array `mBlocks` [S1, unofficial].
*   **Output:** Updated vertex and index buffers for the affected mesh sections, uploaded to the GPU via `CreateMeshSection_LinearColor` [S1, unofficial].

**Data Structures:**
*   `mBlocks`: A 1D linear array storing voxel types, sized to $(mSectionSide \times mSectionSide \times mSectionSide) + \text{boundary padding}$ to handle inter-chunk rendering [S2, unofficial].
*   `mSectionSide`: The side length of a section in voxels (typically 16) [S2, unofficial].
*   `mSectionCount`: The total number of sections stacked vertically in the chunk [S1, unofficial].

**Numbered Steps:**
1.  **Validate Edit:** Calculate the linear index `positionInTArray` for $(i, j, k)$. If `mBlocks[positionInTArray]` already equals `blockType`, return immediately [S1, unofficial].
2.  **Update Voxel Data:** Set `mBlocks[positionInTArray] = blockType` [S1, unofficial].
3.  **Identify Affected Sections:**
    *   Calculate the primary section index: $section = \lfloor i / mSectionSide \rfloor$ [S1, unofficial].
    *   Check for boundary conditions: If $i \pmod{mSectionSide} == 0$ or $(i + 1) \pmod{mSectionSide} == 0$, the edit lies on a section boundary [S1, unofficial].
    *   If on boundary, identify the adjacent section:
        *   If at the bottom edge ($i \pmod{mSectionSide} == 0$), include $section - 1$ [S1, unofficial].
        *   If at the top edge ($(i + 1) \pmod{mSectionSide} == 0$), include $section + 1$ [S1, unofficial].
4.  **Clear Existing Data:** For each affected section index, clear the existing mesh data using `ClearMeshSection` [S1, unofficial].
5.  **Regenerate Mesh:** For each affected section, call `GetMeshData` to generate vertices and indices, then upload via `CreateMeshSection_LinearColor` [S1, unofficial].
6.  **Handle Inter-Chunk Updates:** If the edit occurs on a chunk boundary ($j==0$ or $k==0$), notify the adjacent chunk to update its edge voxels [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def UpdateVoxel(chunk, i, j, k, blockType):
    # 1. Validate and Update
    pos = GetPositionInTArray(i, j, k, chunk.mSectionSide)
    if chunk.mBlocks[pos] == blockType:
        return

    chunk.mBlocks[pos] = blockType

    # 2. Identify Sections
    section = i // chunk.mSectionSide
    sections_to_rebuild = [section]

    # Check boundary conditions
    if i % chunk.mSectionSide == 0:
        if section > 0:
            sections_to_rebuild.append(section - 1)
    if (i + 1) % chunk.mSectionSide == 0:
        if section < chunk.mSectionCount - 1:
            sections_to_rebuild.append(section + 1)

    # 3. Rebuild Sections
    for sec in sections_to_rebuild:
        chunk.ClearMeshSection(sec)
        mesh_data = chunk.GetMeshData(sec)
        chunk.CreateMeshSection_LinearColor(sec, mesh_data)

    # 4. Inter-Chunk Notification
    if j == 0 or k == 0:
        NotifyAdjacentChunk(chunk, i, j, k, blockType)

def GetPositionInTArray(i, j, k, sectionSide):
    # Maps 3D coordinates to the linear block array [G]
    return i + j * sectionSide + k * sectionSide * sectionSide

def NotifyAdjacentChunk(chunk, i, j, k, blockType):
    # Triggers mesh update on adjacent chunk if j==0 or k==0 [G]
    neighbor = chunk.GetNeighborChunk(j, k)
    if neighbor:
        neighbor.UpdateEdgeVoxels(i, j, k, blockType)
```

**Parameters:**
*   `mSectionSide`: Typically 16. This value determines the granularity of rebuilds [S2, unofficial].
*   `mSectionCount`: Total number of sections in the chunk (e.g., 16 for a 256-high chunk with section size 16) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Checks:** The code must handle indices carefully. The chunk includes extra columns/rows for inter-chunk rendering, so `newI`, `newJ`, `newK` checks must account for padding [S1, unofficial].
*   **Duplicate Generation:** In-flight chunks are tracked in a `TMap<int32, TFuture<...>*>` to prevent duplicate generation tasks if multiple edits occur rapidly [S1, unofficial].
*   **Precision:** Integer division and modulo operations are used for section indexing; ensure no off-by-one errors occur at exact boundaries [G].

## Subtopics

### [Adjacency-Based Mesh Rebuilding](4817-adjacency-based-mesh-rebuilding.md)
An optimization that regenerates mesh geometry only for chunks adjacent to a modified voxel, further limiting updates to the immediate neighborhood of a block change. It distinguishes itself from static meshing by its incremental nature and from full-scene rebuilds by its localized scope. Choose it when minimizing visual artifacts at chunk boundaries is critical without triggering global regeneration.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Provides immediate visual feedback for voxel editing. Performance degrades only if edits occur frequently on section boundaries, requiring multiple section rebuilds [S1, unofficial].
*   **Strengths:** Real-time responsiveness, low latency for user input.
*   **Limitations:** Does not handle inter-chunk boundary updates automatically without explicit notification logic. Requires careful management of in-flight tasks to avoid race conditions [S1, unofficial].

## Background
This method is implemented in the `Manolo94/UE5MinecraftClone` repository [S1, unofficial]. It is part of a larger architecture that includes blueprint-driven world generation and thread-pooled chunk generation [S1, unofficial]. The approach is designed for real-time voxel editing in a game context, prioritizing user interaction speed over global optimization [S1, unofficial]. A sparse session edit overlay is reapplied after regeneration to support persistence [S2, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
[S2] Phase 9: Voxel Surface Exploration, https://github.com/LundstedtAdam/SolarSystem/pull/4, fetched 2026-10-28 (unofficial)
