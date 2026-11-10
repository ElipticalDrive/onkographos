# Global Accumulator + Compaction

Also called: FGatherVisibleCountsCS, FPrefixSumVisibleCountsCS

## Overview
A two-pass GPU compute system that aggregates per-proxy visible splat counts into a global array and computes exclusive prefix sums to generate indirect dispatch and draw arguments for global Phase-3 rendering passes [S1, unofficial]. It differs from sibling primitives by focusing on indirect argument generation for batched splat rendering rather than general-purpose sorting or vertex indexing, specifically optimizing the transition from per-proxy visibility data to global draw calls [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `PerProxyVisibleCount` (StructuredBuffer<uint>), `ProxyIndex` (uint32), `ProxyCount` (uint32), `MaxRenderBudget` (uint32).
*   **Output:** `GlobalBaseOffsets` (RWStructuredBuffer<uint>), `GlobalCalcDistIndirectArgs` (RWStructuredBuffer<uint>), `GlobalSortIndirectArgs` (RWStructuredBuffer<uint>), `GlobalSortParams` (RWStructuredBuffer<uint>), `GlobalDrawIndirectArgs` (RWStructuredBuffer<uint>).

**Data Structures:**
1.  `GlobalVisibleCountArray`: RWStructuredBuffer<uint> of size `ProxyCount`. Stores the total visible splat count for each proxy.
2.  `GlobalBaseOffsets`: RWStructuredBuffer<uint> of size `ProxyCount`. Stores the starting index (offset) of each proxy's splats in the global flattened list.
3.  `GlobalCalcDistIndirectArgs`: RWStructuredBuffer<uint> of size 3. Stores `numGroupsX`, `numGroupsY`, `numGroupsZ` for the distance calculation dispatch.
4.  `GlobalSortIndirectArgs`: RWStructuredBuffer<uint> of size 4. Stores `NumTiles`, `1`, `1` (and padding/unused) for the sort dispatch.
5.  `GlobalSortParams`: RWStructuredBuffer<uint> of size 2. Stores `SortCount` (total visible splats) and `NumTiles`.
6.  `GlobalDrawIndirectArgs`: RWStructuredBuffer<uint> of size 5. Standard indirect draw arguments (VertexCount, InstanceCount, StartVertexLocation, StartInstanceLocation, RootArgumentIndex).

**Steps:**

1.  **Gather Visible Counts:**
    *   Dispatch a compute shader (`FGatherVisibleCountsCS`) with 1 thread per proxy.
    *   Each thread `i` reads `PerProxyVisibleCount[ProxyIndex]` and writes it to `GlobalVisibleCountArray[i]`.
    *   *Note:* The source specifies copying from `GPUResources->VisibleSplatCountBuffer[0]` into `GlobalVisibleCountArray[ProxyIndex]` [S1, unofficial].

2.  **Compute Prefix Sums:**
    *   Dispatch a compute shader (`FPrefixSumVisibleCountsCS`) with 1 thread total (or a minimal group, but the source implies a single-threaded or highly serialized approach for the global accumulation logic given the small data size relative to proxy count, though typically this would be a parallel scan; the source explicitly states "1 thread total computes prefix sums" [S1, unofficial]).
    *   The thread iterates through `GlobalVisibleCountArray` to compute exclusive prefix sums.
    *   Let $C[i]$ be the count for proxy $i$.
    *   $Offset[0] = 0$.
    *   $Offset[i] = Offset[i-1] + C[i-1]$ for $i > 0$.
    *   Write $Offset[i]$ to `GlobalBaseOffsets[i]`.
    *   Calculate total visible splats: $Total = Offset[ProxyCount-1] + C[ProxyCount-1]$.
    *   Apply clamping: If `MaxRenderBudget` > 0, $Total = \min(Total, MaxRenderBudget)$. [S1, unofficial].

3.  **Prepare Indirect Arguments:**
    *   Write $Total$ to `GlobalSortParams[0]` (`SortCount`).
    *   Determine `NumTiles` based on $Total$ and tile size (implementation detail not fully specified in source, but implied by `GlobalSortIndirectArgs` layout) [S1, unofficial].
    *   Write `NumTiles` to `GlobalSortParams[1]`.
    *   Write `NumTiles` to `GlobalSortIndirectArgs[0]`.
    *   Write `1` to `GlobalSortIndirectArgs[1]` and `GlobalSortIndirectArgs[2]`.
    *   Configure `GlobalCalcDistIndirectArgs` based on $Total$ and thread group size `ComputeThreadGroupSize` [S1, unofficial].
    *   Configure `GlobalDrawIndirectArgs` with vertex count $Total$, instance count 1, and appropriate root arguments [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# FGatherVisibleCountsCS
# Thread ID = ProxyIndex
def gather_visible_counts(PerProxyVisibleCount, GlobalVisibleCountArray, ProxyIndex):
    count = PerProxyVisibleCount[ProxyIndex]
    GlobalVisibleCountArray[ProxyIndex] = count

# FPrefixSumVisibleCountsCS
# Single thread execution context
def prefix_sum_and_compact(GlobalVisibleCountArray, GlobalBaseOffsets, 
                           GlobalCalcDistIndirectArgs, GlobalSortIndirectArgs, 
                           GlobalSortParams, GlobalDrawIndirectArgs, 
                           ProxyCount, MaxRenderBudget, ComputeThreadGroupSize):
    
    # 1. Compute Exclusive Prefix Sums
    current_offset = 0
    GlobalBaseOffsets[0] = 0
    
    for i in range(1, ProxyCount):
        count = GlobalVisibleCountArray[i-1]
        current_offset += count
        GlobalBaseOffsets[i] = current_offset
        
    # 2. Calculate Total Visible Splats
    last_count = GlobalVisibleCountArray[ProxyCount - 1]
    total_splats = current_offset + last_count
    
    # 3. Apply MaxRenderBudget Cap
    if MaxRenderBudget > 0:
        if total_splats > MaxRenderBudget:
            total_splats = MaxRenderBudget
            
    # 4. Determine Tile Count for Sorting
    # Assuming a standard tile size, e.g., 256 splats per tile.
    # Source does not specify exact tile size, so we assume a constant or parameter.
    TILE_SIZE = 256 
    num_tiles = (total_splats + TILE_SIZE - 1) // TILE_SIZE
    
    # 5. Write Sort Params
    GlobalSortParams[0] = total_splats
    GlobalSortParams[1] = num_tiles
    
    # 6. Write Sort Indirect Args (NumTiles, 1, 1)
    GlobalSortIndirectArgs[0] = num_tiles
    GlobalSortIndirectArgs[1] = 1
    GlobalSortIndirectArgs[2] = 1
    GlobalSortIndirectArgs[3] = 0 # Padding/Unused
    
    # 7. Write Calc Dist Indirect Args
    # Assuming 1D dispatch for distance calculation based on total splats
    threads_per_group = ComputeThreadGroupSize
    num_groups_x = (total_splats + threads_per_group - 1) // threads_per_group
    GlobalCalcDistIndirectArgs[0] = num_groups_x
    GlobalCalcDistIndirectArgs[1] = 1
    GlobalCalcDistIndirectArgs[2] = 1
    
    # 8. Write Draw Indirect Args
    # Standard indirect draw: VertexCount, InstanceCount, StartVertexLocation, StartInstanceLocation, RootArgumentIndex
    GlobalDrawIndirectArgs[0] = total_splats
    GlobalDrawIndirectArgs[1] = 1
    GlobalDrawIndirectArgs[2] = 0
    GlobalDrawIndirectArgs[3] = 0
    GlobalDrawIndirectArgs[4] = 0 # Root argument index placeholder

```

**Parameters:**
*   `MaxRenderBudget`: uint32. Cap on total visible splats. 0 means no cap [S1, unofficial].
*   `ComputeThreadGroupSize`: uint32. Thread group size for compute shaders [S1, unofficial].

**Edge Cases:**
*   `MaxRenderBudget` clamping may truncate the scene, requiring subsequent logic to handle partial rendering or culling [S1, unofficial].
*   If `ProxyCount` is 0, the prefix sum loop is skipped, and totals are 0 [G].

## Subtopics

### Indirect Argument Preparation
A GPU compute shader variant that prepares indirect dispatch and draw arguments from the visible splat count. It outputs `IndirectDispatchArgs` (uint3: numGroupsX, Y, Z), `IndirectDrawArgs` (5 uints), `SortIndirectArgs` (uint3: NumTiles, 1, 1), and `SortParamsBuffer` (uint2: SortCount, NumTiles) [S1, unofficial]. It is chosen when explicit control over dispatch dimensions and sort parameters is required for the rendering pipeline [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(P)$ for gathering counts, where $P$ is the number of proxies. The prefix sum is described as a single-threaded operation in the source, implying $O(P)$ sequential time, which is negligible for typical proxy counts but may be a bottleneck if $P$ is very large [S1, unofficial].
*   **Memory Complexity:** $O(P)$ for `GlobalVisibleCountArray` and `GlobalBaseOffsets`.
*   **Performance:** The method is part of a pipeline that achieved ~60 FPS to +60% improvement on an RTX 4090 for a 57-million-splat scene when combined with sort interval optimization [S1, unofficial].
*   **Strengths:** Centralizes visibility data, enabling efficient global sorting and drawing.
*   **Limitations:** The single-threaded prefix sum description may not scale to extremely high proxy counts without optimization [S1, unofficial].

## Background
Originating from the OpenSplat4D plugin for Unreal Engine, this method is part of the Gaussian Splatting performance optimization suite [S1, unofficial]. It addresses the need to efficiently transition from per-proxy visibility culling to global batched rendering passes for splat distance calculation, sorting, and drawing [S1, unofficial].

## Sources
[S1] YuanBaoSMadLab/OpenSplat4D, https://github.com/YuanBaoSMadLab/OpenSplat4D, fetched 2026-10-27 (unofficial)
