# Radix Sort for Splat Depth

Also called: 16-bit Radix Sort for Splatting, Depth Sorting via Radix Sort, Tsplat depth sort

## Overview
Radix Sort for Splat Depth sorts projected 3D Gaussian splats by their depth to establish a front-to-back rendering order, which is critical for correct alpha blending in object-order splatting pipelines. It sets itself apart from [Splat Depth Pass](1078-splat-depth-pass.md) by focusing on the sorting algorithm itself rather than depth calculation, and from [Tile Binning for Splatting](1214-tile-binning-for-splatting.md) by performing a global sort rather than spatial binning. It leverages the monotonic property of IEEE 754 float bit patterns to enable efficient O(n) sorting without comparison-based overhead.

## How it works
**Inputs:**
- `projected`: An array of $N$ projected splat structures. Each structure contains at least a `depth` field (f32) and an index or ID.
- `scratch`: A buffer for intermediate storage during the sort.

**Outputs:**
- `projected`: The same array, reordered such that splats are sorted by `depth` in ascending order (front-to-back).

**Data Structures:**
- `counts_lo`: Array of 65,536 `u32` integers, initialized to zero.
- `counts_hi`: Array of 65,536 `u32` integers, initialized to zero.
- `offsets`: Array of 65,536 `u32` integers, used for prefix sums.
- `aux`: Temporary array of $N$ splat structures, same size as `projected`.

**Steps:**

1. **Edge Case Check:** If the number of splats $N \le 1$, return immediately. [S1, unofficial]
2. **Bit Pattern Extraction:** For each splat $i$ in `projected`:
   - Let $d = \text{projected}[i].\text{depth}$.
   - Convert $d$ to its IEEE 754 bit pattern as an unsigned 32-bit integer $k$. This preserves the ordering of positive floats. [S1, unofficial]
   - Extract low 16 bits: $k_{lo} = k \ \& \ 0xFFFF$.
   - Extract high 16 bits: $k_{hi} = k \gg 16$.
   - Increment `counts_lo[k_lo]` and `counts_hi[k_hi]`. [S1, unofficial]
3. **Pass 1 (Low 16 bits):**
   - Compute prefix sums of `counts_lo` into `offsets`. `offsets[x]` represents the starting index for bucket $x$. [S1, unofficial]
   - Scatter splats from `projected` to `aux` based on $k_{lo}$. For each splat $i$:
     - $idx = \text{offsets}[k_{lo}]$.
     - `aux[idx] = projected[i]`.
     - `offsets[k_lo]++`. [S1, unofficial]
4. **Pass 2 (High 16 bits):**
   - Reset `offsets` or recompute prefix sums for `counts_hi`. [S1, unofficial]
   - Scatter splats from `aux` back to `projected` based on $k_{hi}$. For each splat $i$ (iterating through `aux`):
     - $idx = \text{offsets}[k_{hi}]$.
     - `projected[idx] = aux[i]`.
     - `offsets[k_hi]++`. [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def radix_sort_splat_depth(projected: list[Splat], scratch: ScratchBuffers) -> None:
    n = len(projected)
    if n <= 1:
        return

    # Initialize histograms
    counts_lo = [0] * 65536
    counts_hi = [0] * 65536

    # Pass 0: Count frequencies
    for i in range(n):
        # Reinterpret f32 bits as u32. 
        # Note: In a real implementation, this requires bit-casting.
        # Assumes positive depth values where bit pattern order matches float order.
        k = float_to_bits(projected[i].depth)
        k_lo = k & 0xFFFF
        k_hi = k >> 16
        counts_lo[k_lo] += 1
        counts_hi[k_hi] += 1

    # Pass 1: Low 16 bits
    offsets = prefix_sum(counts_lo)
    aux = [None] * n
    
    for i in range(n):
        k = float_to_bits(projected[i].depth)
        k_lo = k & 0xFFFF
        idx = offsets[k_lo]
        aux[idx] = projected[i]
        offsets[k_lo] += 1

    # Pass 2: High 16 bits
    offsets = prefix_sum(counts_hi)
    
    for i in range(n):
        k = float_to_bits(aux[i].depth)
        k_hi = k >> 16
        idx = offsets[k_hi]
        projected[idx] = aux[i]
        offsets[k_hi] += 1

def float_to_bits(f: float) -> int:
    """Converts a positive f32 to its u32 bit representation."""
    # Implementation detail: pack float into 4 bytes, interpret as little-endian u32
    # [G] Standard bit-casting operation.
    pass

def prefix_sum(arr: list[int]) -> list[int]:
    """Computes exclusive prefix sum."""
    n = len(arr)
    result = [0] * n
    current_sum = 0
    for i in range(n):
        result[i] = current_sum
        current_sum += arr[i]
    return result
```

**Parameters:**
- **Parallel Sort Threshold:** If the number of splats is less than 50,000 or no thread pool is available, use the sequential radix sort described above. Otherwise, use a parallel unstable sort by key (e.g., `par_sort_unstable_by_key`) on the depth bits. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Negative Depths:** The method relies on positive float bit patterns preserving order. If depths are negative, the bit pattern order is reversed or inverted. Ensure depths are transformed to positive values (e.g., using absolute value or a monotonic mapping) before bit extraction. [S1, unofficial]
- **Precision:** Using 16-bit radix sort on 32-bit floats discards the lower 16 bits of the mantissa/exponent in the sorting key. This is acceptable for depth sorting as long as the precision loss does not cause incorrect ordering of splats that are visually distinct. [S1, unofficial]

## Subtopics

### [Gaussian Sorting](2845-gaussian-sorting.md)
Gaussian Sorting is the process of ordering 3D Gaussians by their distance from the camera to ensure correct alpha blending during rasterization. It solves the problem of rendering transparency correctly by establishing a front-to-back draw order, which is critical because Gaussians with non-zero opacity accumulate color based on their depth relative to the viewer. It sets itself apart from its parent [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) by focusing on the specific application of 3D Gaussian Splatting, where the sorting is performed on a global index buffer of Gaussian attributes rather than just projected splat depths, and it differs from [Fast Splat Sorting](2953-fast-splat-sorting.md) by relying on explicit compute-shader-based radix sorting of indices rather than optimized spatial partitioning heuristics.

### [Fast Splat Sorting](2953-fast-splat-sorting.md)
Fast Splat Sorting is an optimization for volume rendering pipelines that replaces the single 48-bit combined tileID+depth radix sort with two sequential, smaller-key sorts: a 32-bit depth sort of visible splats followed by a 16-bit tileID sort of generated fragments. It sets itself apart from its parent [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) by decoupling depth ordering from tile assignment, and from [Gaussian Sorting](2845-gaussian-sorting.md) by relying on explicit fragment duplication rather than implicit spatial partitioning. This separation reduces VRAM usage and computational complexity by leveraging the fact that the number of visible splats ($n$) is typically smaller than the number of fragments ($m$).

## Operating characteristics
- **Time Complexity:** O(n) for the radix sort, where n is the number of splats. The constant factor is small due to the fixed 16-bit passes. [S1, unofficial]
- **Memory Complexity:** O(n) for the auxiliary array `aux` and O(1) for the histogram arrays (fixed size 65,536 each). [S1, unofficial]
- **Performance:** Radix sort is consistently 2 times faster than comparison-based sorts for ~100k–200k elements. [S1, unofficial]
- **Strengths:** Efficient for large numbers of splats; leverages hardware-friendly memory access patterns in parallel implementations. [S1, unofficial]
- **Limitations:** Requires additional memory for the auxiliary array; sensitive to the distribution of depth values if not properly normalized. [S1, unofficial]
- **When to Use:** When rendering scenes with a large number of splats (e.g., >50,000) where sorting overhead becomes significant. [S1, unofficial]
- **When Not to Use:** For very small numbers of splats (<50,000) where the overhead of radix sort setup may outweigh benefits, or when parallel processing is not available. [S1, unofficial]

## Background
The method originates from the Tsplat project, which implements Gaussian splatting in a terminal environment. It was designed to provide efficient sorting for real-time rendering of large-scale Gaussian splat scenes. The approach leverages the properties of IEEE 754 floating-point numbers to avoid expensive comparison operations. [S1, unofficial]

## Sources
[S1] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S2] ReconWorldLab/godot-gaussian-splatting, https://github.com/ReconWorldLab/godot-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S3] Explore: Gaussian splatting, OpenUSD, AI world generation, volumetric video — study, https://github.com/sceneview/sceneview/issues/4023, fetched 2026-10-27 (unofficial)
[S4] VkSplat: High-Performance 3DGS Training in Vulkan Compute, https://doi.org/10.2312/egs.20261024, fetched 2026-10-09 (unofficial)
