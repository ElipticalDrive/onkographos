# Cascaded 2-way Merge for Global Splat Sorting


## Overview
Cascaded 2-way Merge is a GPU-based algorithm that merges pre-sorted splat lists from multiple independent renderers into a single globally depth-sorted order buffer using a cascaded binary merge tree [S1, unofficial]. It solves the problem of incorrect depth ordering and excessive overdraw when rendering multiple active Gaussian Splat scenes in a unified viewport [S1, unofficial]. Unlike per-renderer rendering, where each renderer handles its own depth sorting and draw calls independently, this method delegates sorting execution to a global sorter and rendering to a global renderer [S1, unofficial].

## How it works
The method operates on $K$ active renderers, each providing a list of splats sorted by depth [S1, unofficial]. The goal is to produce a single list of $N = \sum_{k=0}^{K-1} \text{cnt}_k$ splats sorted by depth [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** $K$ renderers. Renderer $k$ has `RemainingCount` ($\text{cnt}_k$) splats [S1, unofficial].
- **Outputs:** `m_globalOrderBuffer` containing the final sorted global order indices [S1, unofficial].

**Data Structures:**
The algorithm uses four scratch buffers and one global output buffer, all of size `m_totalSplatCount` [S1, unofficial]:
1. `m_packScratchOrders`: `GraphicsBuffer` of type `uint`, name 'Gsplat.PackScratchOrders' [S1, unofficial].
2. `m_packScratchDepths`: `GraphicsBuffer` of type `float`, name 'Gsplat.PackScratchDepths' [S1, unofficial].
3. `m_mergeScratchOrders`: `GraphicsBuffer` of type `uint`, name 'Gsplat.MergeScratchOrders' [S1, unofficial].
4. `m_mergeScratchDepths`: `GraphicsBuffer` of type `float`, name 'Gsplat.MergeScratchDepths' [S1, unofficial].
5. `m_globalOrderBuffer`: `GraphicsBuffer` holding the final sorted global order, bound via `k_globalOrderBuffer` property [S1, unofficial].

**Parameters:**
- `k_rendererIdx`: Index of the current renderer being packed [S1, unofficial].
- `k_srcCount`: Number of splats in the current renderer's range [S1, unofficial].
- `k_dstOffset`: Offset in the destination scratch buffer for the current renderer's packed data [S1, unofficial].
- `k_offsetA`: Offset of Source A (merged result) in the destination buffer [S1, unofficial].
- `k_offsetB`: Offset of Source B (current renderer) in the destination buffer [S1, unofficial].
- `k_countA`: Number of elements in Source A [S1, unofficial].
- `k_countB`: Number of elements in Source B [S1, unofficial].

**Algorithm Steps:**
1. **Packing Phase:** For each renderer $k$ from $0$ to $K-1$, if $\text{cnt}_k > 0$, dispatch `m_kernelPackOrders` with parameters `k_rendererIdx = k`, `k_srcCount = cnt_k`, `k_dstOffset = off_k`. The kernel reads raw orders/depths from `res.OrderBuffer`/`res.InputKeys` and writes to `m_packScratchOrders`/`m_packScratchDepths` at offset $off_k$ [S1, unofficial].
2. **Initialization:** Set `mergedCount = activeGsplats[0].RemainingCount`, `mergedOffset = 0`, `mergedInPack = true` [S1, unofficial].
3. **Cascaded Merge Loop:** Iterate $k$ from $1$ to $K-1$. Calculate `total = mergedCount + cnt_k`. Select Source A: if `mergedInPack` is true, use `m_packScratchOrders`/`Depths`; otherwise use `m_mergeScratchOrders`/`Depths`. Select Source B: always `m_packScratchOrders`/`Depths` at offset $off_k$. Select Destination: if $k == K-1$ (final pass), use `m_globalOrderBuffer` with `dstDepths = null`; otherwise use the buffer not currently holding the merged result. Dispatch merge kernel: use `m_kernelMergeTwo` if final, otherwise `m_kernelMergeTwoWithDepth`. Set parameters `k_offsetA = mergedOffset`, `k_offsetB = off_k`, `k_countA = mergedCount`, `k_countB = cnt_k`, `k_outputOffset = 0`. Bind sources and destination. Dispatch with thread count $\lceil \text{total} / 256 \rceil$ [S1, unofficial].
4. **Finalization:** After the loop, set `m_totalRemainingCount = mergedCount` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Type aliases for GPU buffer types
uint = int
float = float

class GraphicsBuffer:
    def __init__(self, dtype, size, name):
        self.dtype = dtype
        self.size = size
        self.name = name
        self.data = [None] * size

def dispatch_pack_orders(renderer_idx, src_count, dst_offset, src_orders, src_depths, dst_orders, dst_depths):
    # Simulates GPU kernel: reads raw orders/depths from src buffers, writes to dst buffers at dst_offset
    pass

def dispatch_merge_two(offset_a, offset_b, count_a, count_b, output_offset, src_a_orders, src_b_orders, dst_orders):
    # Simulates GPU kernel: merges two sorted order lists without depth output
    pass

def dispatch_merge_two_with_depth(offset_a, offset_b, count_a, count_b, output_offset, src_a_orders, src_a_depths, src_b_orders, src_b_depths, dst_orders, dst_depths):
    # Simulates GPU kernel: merges two sorted order lists with depth output
    pass

def cascaded_2_way_merge(active_gsplats, m_total_splat_count, m_global_order_buffer):
    K = len(active_gsplats)
    if K < 2 or m_global_order_buffer is None:
        return

    pack_orders = GraphicsBuffer(uint, m_total_splat_count, "Gsplat.PackScratchOrders")
    pack_depths = GraphicsBuffer(float, m_total_splat_count, "Gsplat.PackScratchDepths")
    merge_orders = GraphicsBuffer(uint, m_total_splat_count, "Gsplat.MergeScratchOrders")
    merge_depths = GraphicsBuffer(float, m_total_splat_count, "Gsplat.MergeScratchDepths")

    # Step 1: Packing Phase
    offset = 0
    for k in range(K):
        cnt_k = active_gsplats[k].RemainingCount
        if cnt_k == 0:
            continue
        dispatch_pack_orders(k, cnt_k, offset, active_gsplats[k].OrderBuffer, active_gsplats[k].InputKeys, pack_orders, pack_depths)
        offset += cnt_k

    # Step 2: Initialization
    merged_count = active_gsplats[0].RemainingCount
    merged_offset = 0
    merged_in_pack = True

    # Step 3: Cascaded Merge Loop
    for k in range(1, K):
        cnt_k = active_gsplats[k].RemainingCount
        if cnt_k == 0:
            continue
        total = merged_count + cnt_k
        is_final = (k == K - 1)

        # Select Source A (Current Merged Result)
        if merged_in_pack:
            src_a_orders, src_a_depths = pack_orders, pack_depths
        else:
            src_a_orders, src_a_depths = merge_orders, merge_depths

        # Select Source B (Renderer k's Data)
        src_b_orders, src_b_depths = pack_orders, pack_depths

        # Select Destination
        if is_final:
            dst_orders = m_global_order_buffer
            dst_depths = None
        else:
            if merged_in_pack:
                dst_orders, dst_depths = merge_orders, merge_depths
            else:
                dst_orders, dst_depths = pack_orders, pack_depths

        # Dispatch Merge Kernel
        k_offset_a = merged_offset
        k_offset_b = offset - cnt_k
        k_count_a = merged_count
        k_count_b = cnt_k
        k_output_offset = 0
        thread_count = (total + 255) // 256

        if is_final:
            dispatch_merge_two(k_offset_a, k_offset_b, k_count_a, k_count_b, k_output_offset, src_a_orders, src_b_orders, dst_orders)
        else:
            dispatch_merge_two_with_depth(k_offset_a, k_offset_b, k_count_a, k_count_b, k_output_offset, src_a_orders, src_a_depths, src_b_orders, src_b_depths, dst_orders, dst_depths)

        # Update State
        merged_count = total
        merged_offset = 0
        merged_in_pack = not merged_in_pack

    # Step 4: Finalization
    m_total_remaining_count = merged_count
```

**Edge Cases and Pitfalls:**
- If $K < 2$ or `m_globalOrderBuffer` is null, the merge process returns immediately without performing any merges [S1, unofficial].
- If a renderer's count is $0$, it is skipped during the packing phase [S1, unofficial].
- The algorithm requires GPU subgroups/wave operations, which are only available in D3D12, Metal, or Vulkan graphics APIs [S1, unofficial]. WebGPU supports subgroups but Unity has not implemented it for this feature [S1, unofficial].
- Global sort is all-or-nothing: presence of any uncompressed renderer forces fallback to per-renderer sorting [S1, unofficial].

## Subtopics

### [Global Splat Merge and Rendering](7691-global-splat-merge-and-rendering.md)
Global Splat Merge and Rendering is a technique that merges multiple independent Gaussian Splat renderers into a single global draw call by packing their splat data, spherical harmonic (SH) coefficients, and depth/order information into shared global buffers, then performing a cascaded merge of depth-sorted orders. It merges two or more active Gaussian splat renderers into a single globally depth-sorted draw call to reduce overdraw and improve sorting accuracy across scenes. Unlike per-renderer rendering, where each renderer handles its own depth sorting and draw calls independently, this method delegates sorting execution to a global sorter and rendering to a global renderer.

### Global Depth Sorting for 3DGS
Global Depth Sorting for 3DGS sorts Gaussians across multiple renderers globally using GPU subgroups/waves to maintain correct depth ordering in a unified scene [S1, unofficial]. It is activated by the `EnableGlobalSort` boolean setting, which requires all active renderers to use compressed assets [S1, unofficial]. A key pitfall is that global sort is all-or-nothing: the presence of any uncompressed renderer forces a fallback to per-renderer sorting [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Performance depends on the GPU's subgroup/wave support [S1, unofficial].
- **Scaling:** Not given in the sources.
- **Strengths:** Reduces overdraw, improves sorting accuracy across scenes, enables unified rendering of multiple scenes [S1, unofficial].
- **Limitations:** Requires compressed assets for all renderers, limited to specific graphics APIs, all-or-nothing activation [S1, unofficial].
- **When to use:** When rendering multiple Gaussian Splat scenes in a single viewport with overlapping depth ranges [S1, unofficial].
- **When not to use:** When renderers use uncompressed assets, or when the target platform does not support GPU subgroups (e.g., older APIs, WebGPU in Unity) [S1, unofficial].

## Background
The method originates from the `wuyize25/gsplat-unity` project [S1, unofficial]. It builds on the `b0nes164/GPUSorting` library for GPU-based sorting primitives [S1, unofficial]. The technique was developed to address the limitations of per-renderer sorting in multi-scene environments, providing a more efficient and accurate way to composite multiple Gaussian Splat scenes [S1, unofficial].

## Sources
[S1] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
