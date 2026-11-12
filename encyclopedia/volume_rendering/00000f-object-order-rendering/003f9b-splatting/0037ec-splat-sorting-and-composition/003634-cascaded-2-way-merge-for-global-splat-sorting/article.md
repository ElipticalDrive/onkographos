# Cascaded 2-way Merge for Global Splat Sorting

Also called: Global Splat Merge, Cascaded Binary Merge Tree for Splat Sorting

## Overview
A GPU-based algorithm that merges sorted splat lists from multiple active Gaussian Splat renderers into a single globally sorted order buffer using a cascaded binary merge tree [S1, unofficial]. It solves the problem of incorrect depth ordering and excessive overdraw when rendering multiple independent Gaussian Splat scenes in a unified view by delegating sorting execution to a global sorter and rendering to a global renderer [S1, unofficial]. Unlike per-renderer rendering, where each renderer handles its own depth sorting and draw calls independently, this method ensures correct cross-renderer depth compositing [S1, unofficial].

## How it works
The method operates on $K$ active renderers, each having already produced a locally depth-sorted list of splats [S1, unofficial]. The goal is to produce a single global list sorted by depth [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** $K$ active renderers. Renderer $k$ provides a sorted list of splats with count $cnt_k$ and an offset $off_k$ in a global scratch space [S1, unofficial].
- **Outputs:** A single global order buffer ($m\_globalOrderBuffer$) containing the indices of all splats sorted by depth [S1, unofficial].

**Data Structures:**
The implementation uses four scratch buffers and one final output buffer, all of size $m\_totalSplatCount$ (the sum of splat counts from all active renderers) [S1, unofficial]:
1.  $m\_packScratchOrders$: `GraphicsBuffer` of type `uint`, name 'Gsplat.PackScratchOrders' [S1, unofficial].
2.  $m\_packScratchDepths$: `GraphicsBuffer` of type `float`, name 'Gsplat.PackScratchDepths' [S1, unofficial].
3.  $m\_mergeScratchOrders$: `GraphicsBuffer` of type `uint`, name 'Gsplat.MergeScratchOrders' [S1, unofficial].
4.  $m\_mergeScratchDepths$: `GraphicsBuffer` of type `float`, name 'Gsplat.MergeScratchDepths' [S1, unofficial].
5.  $m\_globalOrderBuffer$: `GraphicsBuffer` holding the final sorted global order [S1, unofficial].

**Parameters:**
- $k\_rendererIdx$: Index of the current renderer being packed [S1, unofficial].
- $k\_srcCount$: Number of splats in the current renderer's range [S1, unofficial].
- $k\_dstOffset$: Offset in the destination scratch buffer for the current renderer's packed data [S1, unofficial].
- $k\_offsetA$: Offset of Source A (merged result) in the destination buffer [S1, unofficial].
- $k\_offsetB$: Offset of Source B (current renderer) in the destination buffer [S1, unofficial].
- $k\_countA$: Number of elements in Source A [S1, unofficial].
- $k\_countB$: Number of elements in Source B [S1, unofficial].

**Algorithm Steps:**

1.  **Pack Local Orders:** For each renderer $k$ from $0$ to $K-1$:
    - If $cnt_k == 0$, skip renderer $k$ [S1, unofficial].
    - Dispatch kernel `m_kernelPackOrders` to pack renderer $k$'s raw orders and depths from its local buffers (`res.OrderBuffer`, `res.InputKeys`) into the scratch buffers at offset $off_k$:
        - Write to $m\_packScratchOrders$ and $m\_packScratchDepths$ [S1, unofficial].
        - Parameters: $k\_rendererIdx=k$, $k\_srcCount=cnt_k$, $k\_dstOffset=off_k$ [S1, unofficial].

2.  **Initialize Merge State:**
    - Set $mergedCount = activeGsplats[0].RemainingCount$ [S1, unofficial].
    - Set $mergedOffset = 0$ [S1, unofficial].
    - Set $mergedInPack = \text{true}$, indicating the current merged result resides in $m\_packScratch$ buffers [S1, unofficial].

3.  **Cascaded Merge Loop:** Iterate $k$ from $1$ to $K-1$:
    - Calculate $total = mergedCount + cnt_k$ [S1, unofficial].
    - **Select Source A (Current Merged Result):**
        - If $mergedInPack$ is true, Source A is $m\_packScratchOrders$ and $m\_packScratchDepths$ [S1, unofficial].
        - Otherwise, Source A is $m\_mergeScratchOrders$ and $m\_mergeScratchDepths$ [S1, unofficial].
    - **Select Source B (Renderer $k$'s Data):**
        - Source B is always located in $m\_packScratchOrders$ and $m\_packScratchDepths$ at offset $off_k$ [S1, unofficial].
    - **Select Destination:**
        - If $k == K-1$ (final renderer), Destination Orders is $m\_globalOrderBuffer$, and Depth output is null [S1, unofficial].
        - Otherwise, Destination is the buffer *not* currently holding the merged result (swaps between $m\_mergeScratch$ and $m\_packScratch$) [S1, unofficial].
    - **Dispatch Merge Kernel:**
        - If $k == K-1$, use kernel `m_kernelMergeTwo` (no depth output) [S1, unofficial].
        - Otherwise, use kernel `m_kernelMergeTwoWithDepth` (outputs depths) [S1, unofficial].
        - Parameters: $k\_offsetA=mergedOffset$, $k\_offsetB=off_k$, $k\_countA=mergedCount$, $k\_countB=cnt_k$, $k\_outputOffset=0$ [S1, unofficial].
        - Bind Source A, Source B, Destination Orders, and Destination Depths (if not final) [S1, unofficial].
        - Dispatch with thread count $\lceil total / 256 \rceil$ [S1, unofficial].
    - **Update State:**
        - Set $mergedCount = total$ [S1, unofficial].
        - Update $mergedInPack$ based on which buffer was used as destination [S1, unofficial].

4.  **Finalize:**
    - Set $m\_totalRemainingCount = mergedCount$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cascaded_2_way_merge(active_renderers, m_packScratchOrders, m_packScratchDepths, 
                         m_mergeScratchOrders, m_mergeScratchDepths, m_globalOrderBuffer):
    K = len(active_renderers)
    
    # Edge case: No merges needed
    if K < 2 or m_globalOrderBuffer is None:
        return

    # Step 1: Pack local orders into scratch buffers
    current_offset = 0
    renderer_offsets = []
    for k in range(K):
        cnt_k = active_renderers[k].RemainingCount
        renderer_offsets.append(current_offset)
        
        if cnt_k == 0:
            continue
            
        # Dispatch m_kernelPackOrders
        # Copies raw orders/depths from active_renderers[k].OrderBuffer and .InputKeys
        # to scratch buffers at current_offset
        dispatch_pack_orders(active_renderers[k], m_packScratchOrders, m_packScratchDepths, 
                             current_offset, cnt_k, k)
        
        current_offset += cnt_k

    # Step 2: Initialize merge state
    merged_count = active_renderers[0].RemainingCount
    merged_offset = 0
    merged_in_pack = True  # Result is in packScratch buffers
    
    # Step 3: Cascaded merge loop
    for k in range(1, K):
        cnt_k = active_renderers[k].RemainingCount
        
        if cnt_k == 0:
            continue
            
        total = merged_count + cnt_k
        is_final = (k == K - 1)
        off_k = renderer_offsets[k]
        
        # Select Source A (merged result so far)
        if merged_in_pack:
            src_a_orders = m_packScratchOrders
            src_a_depths = m_packScratchDepths
        else:
            src_a_orders = m_mergeScratchOrders
            src_a_depths = m_mergeScratchDepths
            
        # Select Source B (current renderer's data)
        src_b_orders = m_packScratchOrders
        src_b_depths = m_packScratchDepths
        
        # Select Destination
        if is_final:
            dst_orders = m_globalOrderBuffer
            dst_depths = None
        else:
            # Swap buffers
            if merged_in_pack:
                dst_orders = m_mergeScratchOrders
                dst_depths = m_mergeScratchDepths
            else:
                dst_orders = m_packScratchOrders
                dst_depths = m_packScratchDepths
        
        # Dispatch Merge Kernel
        if is_final:
            # Final pass: no depth output
            dispatch_merge_two(src_a_orders, src_a_depths, src_b_orders, src_b_depths,
                               dst_orders, merged_offset, off_k, merged_count, cnt_k, 0)
        else:
            # Intermediate pass: depth output
            dispatch_merge_two_with_depth(src_a_orders, src_a_depths, src_b_orders, src_b_depths,
                                          dst_orders, dst_depths, merged_offset, off_k, merged_count, cnt_k, 0)
            
        # Update state
        merged_count = total
        merged_offset = 0  # Output is always written to offset 0 of the destination buffer
        merged_in_pack = (dst_orders is m_packScratchOrders)

    # Step 4: Finalize
    m_totalRemainingCount = merged_count

def dispatch_pack_orders(renderer, dst_orders, dst_depths, offset, count, renderer_idx):
    # Simulates dispatching m_kernelPackOrders to the GPU
    pass

def dispatch_merge_two(src_a_orders, src_a_depths, src_b_orders, src_b_depths,
                       dst_orders, offset_a, offset_b, count_a, count_b, output_offset):
    # Simulates dispatching m_kernelMergeTwo (no depth output)
    pass

def dispatch_merge_two_with_depth(src_a_orders, src_a_depths, src_b_orders, src_b_depths,
                                  dst_orders, dst_depths, offset_a, offset_b, count_a, count_b, output_offset):
    # Simulates dispatching m_kernelMergeTwoWithDepth (outputs depths)
    pass
```

## Subtopics

### Global Depth Sorting for 3DGS
Sorts Gaussians across multiple renderers globally using GPU subgroups/waves to maintain correct depth ordering in a unified scene [S1, unofficial]. It is activated by the `EnableGlobalSort` boolean setting, which requires all active renderers to use compressed assets [S1, unofficial]. A key pitfall is that global sort is all-or-nothing: the presence of any uncompressed renderer forces a fallback to per-renderer sorting [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The merge process is $O(NK)$, where $N$ is the total number of splats and $K$ is the number of active renderers, as it performs an iterative pairwise merge across $K-1$ stages [S1, unofficial]. Each merge step is linear in the number of elements being merged [S1, unofficial].
- **Memory Complexity:** Requires four scratch buffers and one output buffer, each of size $N$, totaling $5N$ elements for orders and depths [S1, unofficial].
- **Performance:** Relies on GPU hardware acceleration, specifically requiring wave/subgroup operations available in D3D12, Metal, or Vulkan [S1, unofficial]. WebGPU supports subgroups but may lack engine implementation (e.g., Unity) [S1, unofficial].
- **Strengths:** Reduces overdraw and improves sorting accuracy across scenes by merging multiple renderers into a single globally depth-sorted draw call [S1, unofficial].
- **Limitations:** Requires specific graphics APIs (no OpenGL/WebGL support due to subgroup requirements) [S1, unofficial]. Falls back to per-renderer sorting if uncompressed assets are present [S1, unofficial].
- **When to use:** When rendering multiple independent Gaussian Splat scenes in a unified view with correct depth compositing [S1, unofficial].
- **When not to use:** When only a single renderer is active, or when uncompressed assets are used and global sorting is required [S1, unofficial].

## Background
Originated from the `wuyize25/gsplat-unity` project, an unofficial implementation of Gaussian Splatting for Unity [S1, unofficial]. It builds on the `b0nes164/GPUSorting` library for GPU-based sorting [S1, unofficial]. The method is designed to handle global sorting for 3D Gaussian Splatting (3DGS) in real-time rendering pipelines like Unity's BiRP, URP, and HDRP [S1, unofficial].

## Sources
[S1] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
