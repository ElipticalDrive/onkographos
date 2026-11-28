# BVH with 8-bit Quantized Bounds and Indirect Interior Storage

Also called: bvh8-q8-ci, bvh8i, qbvh8i

## Overview
This method defines an 8-ary Bounding Volume Hierarchy (BVH) layout optimized for memory efficiency and traversal speed by storing interior node bounds in a separate indirect array and quantizing child bounding boxes to 8-bit integers. It solves the problem of high memory bandwidth consumption and cache pressure during ray traversal by reducing the footprint of interior nodes and leveraging SIMD-friendly integer arithmetic for intersection tests. It differs from its sibling [BVH for Gaussian Surfels](17858-bvh-for-gaussian-surfels.md) by targeting triangle primitives with a specific 8-ary topology, and from [Heuristic BVH Construction (Top-Down with Plane-Sweep)](18008-heuristic-bvh-construction-top-down-with-plane-sweep.md) by defining the storage layout and quantization scheme rather than the construction heuristic. It extends [Hierarchical Bounding Volumes (HBV)](13475-hierarchical-bounding-volumes-hbv.md) by introducing indirect interior storage and 8-bit quantization.

## How it works
**Inputs:** A set of geometric primitives (specifically triangles) and a target BVH topology (8-ary).
**Outputs:** A serialized BVH structure containing a primitive array and an indirect interior array, along with a root index.

### Data Structures
The layout is defined by two main groups: `Interiors` and the main indexed group.

1.  **Global Header:**
    *   `primitive_count`: `u32`
    *   `primitives`: Array of `Triangle` of size `primitive_count`.
    *   `interior_count`: `u32`

2.  **Indirect Interior Group (`Interiors`):**
    *   Size: `interior_count`
    *   Fields per node:
        *   `mlo`: `f32x3` (Merged Low bounds of children)
        *   `mex`: `f32x3` (Merged Extent of children, $mhi - mlo$)
        *   `child_bounds`: `qbox3x8` (Quantized bounds for 8 children)
        *   `children`: `u32x8` (Pointers/indices to children)
    *   Derived fields (for traversal logic):
        *   $lo = \text{dequantize\_bounds\_lo}(mlo, mex, child\_bounds)$
        *   $hi = \text{dequantize\_bounds\_hi}(mlo, mex, child\_bounds)$

3.  **Main Indexed Group (indexed by `I`):**
    *   Default root index `I` is 1.
    *   The index `I` is a 32-bit integer split as follows:
        *   Bit 0: Tag bit. `1` indicates an Interior node; `0` indicates a Leaf node.
        *   Bits 1-31: Payload.
            *   If Tag is `1` (Interior): Bits 2-31 store the index into the `Interiors` array. Bit 1 is unused/reserved.
            *   If Tag is `0` (Leaf): Bits 7-31 store the primitive offset `O`. Bits 2-6 store `nprims - 1`.

4.  **Quantized Box (`qbox3`):**
    *   Represents a 3D bounding box quantized to 8 bits per axis.
    *   Fields: `lo`: `u8x3`, `hi`: `u8x3`.
    *   Values are clamped to $[0, 255]$.

5.  **Quantized Box Array (`qbox3x8`):**
    *   Array of 8 `qbox3` structures, one for each child of an interior node.

### Algorithms and Formulas

**1. Quantization of Child Bounds**
For each interior node with 8 children, the bounds are quantized relative to the merged bounds of all children.

*   **Merged Bounds:**
    $$mlo = \min_i(low_i)$$
    $$mhi = \max_i(high_i)$$
    $$mex = mhi - mlo$$
    Where $i \in [0, 7]$ indexes the children.

*   **Reciprocal:**
    $$rcp = \frac{255.0}{mex}$$

*   **Quantization Function:**
    For each child $i$ and axis $d \in \{x, y, z\}$:
    $$q_{lo}^{(i,d)} = \text{tfloor}((low_i^{(d)} - mlo^{(d)}) \times rcp^{(d)})$$
    $$q_{hi}^{(i,d)} = \text{tceil}((high_i^{(d)} - mlo^{(d)}) \times rcp^{(d)})$$
    
    Where $\text{tfloor}(v)$ and $\text{tceil}(v)$ clamp the result to $[0, 255]$:
    $$\text{tfloor}(v) = \text{clamp}(\lfloor v \rfloor, 0, 255)$$
    $$\text{tceil}(v) = \text{clamp}(\lceil v \rceil, 0, 255)$$

*   **Dequantization (for intersection tests):**
    $$lo^{(i,d)} = mlo^{(d)} + (q_{lo}^{(i,d)} \times rcp^{(d)}) \times mex^{(d)}$$
    $$hi^{(i,d)} = mlo^{(d)} + (q_{hi}^{(i,d)} \times rcp^{(d)}) \times mex^{(d)}$$
    Note: The source specifies dequantization uses the stored `mlo`, `mex`, and quantized bounds. The formula in the notes is $\text{lo} = \text{mlo} + (\text{bound.lo} \times \text{rcp}) \times \text{mex}$. This implies the stored `bound.lo` is the integer index, and it is scaled back to float space.

**2. Index Packing**
*   **Interior Node Index:**
    $$I = (\text{interior\_array\_index} \ll 2) \mid 1$$
*   **Leaf Node Index:**
    $$I = (\text{primitive\_offset} \ll 7) \mid ((\text{nprims} - 1) \ll 2) \mid 0$$

**3. Build Procedure**
The build process constructs the tree bottom-up or via a specified topology, generating the packed indices.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper functions for quantization
def tfloor(f: f32x3) -> u8x3:
    """Floor and clamp to [0, 255]"""
    f1 = floor(f)
    f2 = max(0.0, min(f1, 255.0))
    return f2 as u8x3

def tceil(f: f32x3) -> u8x3:
    """Ceil and clamp to [0, 255]"""
    f1 = ceil(f)
    f2 = max(0.0, min(f1, 255.0))
    return f2 as u8x3

def compute_merged_low(low: f32x3x8) -> f32x3:
    """Compute element-wise minimum across 8 children's low bounds."""
    mlo = low[0]
    for i in range(1, 8):
        mlo = min(mlo, low[i])
    return mlo

def compute_merged_extent(low: f32x3x8, high: f32x3x8) -> f32x3:
    """Compute max(high) - min(low) across 8 children."""
    mlo = compute_merged_low(low)
    mhi = high[0]
    for i in range(1, 8):
        mhi = max(mhi, high[i])
    return mhi - mlo

def quantize_bounds(low: f32x3x8, high: f32x3x8) -> qbox3x8:
    """Quantize 8 children's bounds to 8-bit integers."""
    mlo = compute_merged_low(low)
    mex = compute_merged_extent(low, high)
    rcp = (1.0 / mex) * 255.0
    
    child_bounds = qbox3x8()
    for i in range(8):
        # Quantize each axis for child i
        q_lo = tfloor((low[i] - mlo) * rcp)
        q_hi = tceil((high[i] - mlo) * rcp)
        child_bounds[i] = qbox3(lo=q_lo, hi=q_hi)
    return child_bounds

# BVH Build Functions
def build_interior_node(children: BVH[8], lo: f32x3x8, hi: f32x3x8) -> u32:
    """
    Build an interior node.
    Returns the packed index for this node.
    """
    # 1. Compute merged bounds
    mlo = compute_merged_low(lo)
    mex = compute_merged_extent(lo, hi)
    
    # 2. Quantize child bounds
    child_bounds = quantize_bounds(lo, hi)
    
    # 3. Store in Interiors array
    # The index in the Interiors array is determined by the build order/allocator.
    # Let's assume 'interior_index' is the next available slot in Interiors.
    interior_index = allocate_interior_slot()
    
    Interiors[interior_index].mlo = mlo
    Interiors[interior_index].mex = mex
    Interiors[interior_index].child_bounds = child_bounds
    Interiors[interior_index].children = [child.index for child in children]
    
    # 4. Return packed index: shift by 2, set bit 0 to 1
    return (interior_index << 2) | 1

def build_leaf_node(nprims: u8, data: Triangle[nprims]) -> u32:
    """
    Build a leaf node.
    Returns the packed index for this node.
    """
    # 1. Append primitives to global array
    primitive_offset = append_primitives(data, nprims)
    
    # 2. Return packed index:
    # Bits 7-31: primitive_offset
    # Bits 2-6: nprims - 1
    # Bit 0: 0 (Leaf tag)
    # Bit 1: 0 (reserved)
    return (primitive_offset << 7) | ((nprims - 1) << 2) | 0
```

**4. Intersection Method**
For the quantized layout, the Möller-Trumbore algorithm is used for ray-triangle intersection [S1]. The dequantized bounds are used for the BVH traversal step (AABB intersection), and the quantized bounds may be used for early rejection or SIMD optimization if the hardware supports integer AABB tests.

### Edge Cases and Pitfalls
*   **Zero Extent:** If `mex` is zero (degenerate bounding box), division by zero occurs in `rcp`. The implementation must handle this, likely by treating the extent as a small epsilon or skipping quantization. Not given in the sources.
*   **Quantization Error:** Quantization to 8 bits introduces error. The dequantized bounds are an approximation. This may lead to false negatives in intersection tests if the approximation is too coarse, though the source notes that Möller-Trumbore is used for the final primitive test, mitigating this risk for correctness.
*   **Index Overflow:** The leaf index packs `nprims - 1` into 5 bits (bits 2-6). This limits `nprims` to $2^5 = 32$ primitives per leaf. If a leaf has more than 32 primitives, the layout cannot represent it directly. Not given in the sources if this is a hard limit or if larger leaves are split.
*   **Interior Index Overflow:** The interior index is stored in 30 bits (bits 2-31), allowing for $2^{30}$ interior nodes.

## Operating characteristics
*   **Memory Footprint:** Significantly reduced compared to unquantized layouts (e.g., `bvh8i`) because child bounds are stored as 8-bit integers instead of 32-bit floats, and interior nodes are stored in a separate compact array. The source notes that `qbvh8i` uses less memory than `bvh8i` [S1].
*   **Traversal Speed:** The use of 8-bit quantization and integer arithmetic can improve SIMD performance. The source notes that `qbvh8i` has lower latency per ray than `bvh8i` in some configurations [S1].
*   **Construction Time:** The build procedure involves additional floating-point operations for quantization and dequantization setup. The source notes that fully replicating Embree's construction strategy is impractical, implying construction is complex [S1].
*   **Scalability:** The 8-ary tree structure reduces tree height, potentially improving cache locality. The indirect interior storage allows for better memory layout optimization.
*   **Limitations:** Limited to 32 primitives per leaf. Quantization error may affect precision, though the final primitive test is accurate.

## Background
This layout is part of Intel Embree's 8-ary BVH layouts, specifically `qbvh8i` for quantized representations [S1]. It represents over a decade of production engineering effort by domain experts [S1]. It is used in production rendering systems for its balance of memory efficiency and traversal speed. The source "Decoupling Data Layouts from Bounding Volume Hierarchies" compares this layout with other BVH implementations [S1].

## Sources
[S1] Decoupling Data Layouts from Bounding Volume Hierarchies, https://arxiv.org/pdf/2511.15028, fetched 2026-10-29
