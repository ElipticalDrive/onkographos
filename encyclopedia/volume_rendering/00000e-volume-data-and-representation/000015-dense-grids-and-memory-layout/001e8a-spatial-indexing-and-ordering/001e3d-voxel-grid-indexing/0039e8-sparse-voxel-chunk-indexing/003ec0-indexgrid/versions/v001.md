# IndexGrid

Also called: IndexGrid VDB, Sparse IndexGrid, VDB index grid

## Overview
IndexGrid is a sparse data structure derived from NanoVDB, designed to aggressively accelerate stencil-centric operations (e.g., convolution) while serving as an effective acceleration structure for ray marching and other spatial operations [S1]. It solves the problem of memory inefficiency and code complexity in standard VDB implementations by separating topology (sparsity pattern) from values (features) [S1]. Unlike hash tables, which lack spatial coherence, or octrees, which suffer from $O(\log n)$ access times, IndexGrid uses a fixed-depth shallow tree to maintain $O(1)$ amortized access while preserving spatial coherence [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D integer coordinates $(i, j, k)$ and a reference to an IndexGrid leaf node.
*   **Output:** A linear index into an external "sidecar" value array. If the voxel at $(i, j, k)$ is inactive (background), the output is 0 [S1].

**Data Structures:**
The IndexGrid leaf node encodes only topology. It does not store data values.
*   **Leaf Node Layout:**
    *   `uint64_t mOffset`: The base offset into the external value array for the first active voxel in this leaf [S1].
    *   `uint64_t mPrefixSum`: Encodes the cumulative count of active voxels in the preceding 64-bit words of the bitmask [S1].
    *   `uint64_t mBitMask[8]`: An array of 8 64-bit words representing the occupancy of 512 dense leaf voxels ($8^3$) [S1].
    *   **Total Size:** 80 bytes [S1]. This is a >50x reduction compared to a standard NanoVDB LeafNode which stores values and exceeds 4KB [S1].

**Index Computation Logic:**
The index is computed from coordinates $(i, j, k)$ using the following steps [S1]:

1.  **Calculate Linear Index:** Compute the linear index $m$ inside the leaf node from coordinates $i, j, k$. The leaf node is $8 \times 8 \times 8$.
    $$ m = ((i \ \& \ 7) \ll 6) \ | \ ((j \ \& \ 7) \ll 3) \ | \ (k \ \& \ 7) $$
    Here, $m \in [0, 511]$ [S1].

2.  **Determine Word Index:** Determine which 64-bit word $n$ in `mBitMask` contains the bit for $m$.
    $$ n = m \gg 6 $$
    Here, $n \in [0, 7]$ [S1].

3.  **Create Mask:** Create a bit mask for the specific bit position within the word.
    $$ \text{mask} = 1 \ll (m \ \& \ 63) $$

4.  **Check Activity:** If the bit is not set in the word, the voxel is inactive.
    ```python
    w = mBitMask[n]
    if (w & mask) == 0:
        return 0  # Background index
    ```

5.  **Extract Prefix Sum:** If active, extract the prefix sum of active values from preceding words. The `mPrefixSum` field stores 7 $\times$ 9-bit values (one for each of the first 7 words).
    ```python
    if n > 0:
        # Extract the 9-bit prefix sum for word n-1
        # The bits for word k are stored at offset 9*k
        sum = (mPrefixSum >> (9 * n)) & 511
    else:
        sum = 0
    ```
    *Note: The source code snippet uses `mPrefixSum >> (9*n) & 511` to extract the count of active bits in all words preceding word `n` [S1].*

6.  **Count Preceding Active Bits:** Count the number of active bits in the current word `w` that precede the current bit `mask`.
    ```python
    # Count bits set in w that are strictly less significant than mask
    # This is equivalent to counting bits in (w & (mask - 1))
    count = countOn(w & (mask - 1))
    ```

7.  **Compute Final Index:** Add the base offset, the prefix sum, and the count of preceding active bits in the current word.
    $$ \text{index} = \text{sum} + \text{mOffset} + \text{count} $$

**Node Offsets:**
For traversal in upper and lower nodes, specific offset functions map coordinates to node indices [S1]:

*   **Lower Node Offset (Leaf Nodes):** Maps coordinates to indices $0 \dots 16^3 - 1$.
    ```python
    def lower_off(i, j, k):
        # Helper to extract top 5 bits of 8-bit coordinate (0-127 range)
        # Actually, coordinates are typically larger, but this function
        # assumes a specific coordinate space or normalization.
        # Based on source: (n & 127) >> 3
        a = lambda n: (n & 127) >> 3
        return (a(i) << 8) | (a(j) << 4) | a(k)
    ```

*   **Upper Node Offset (Upper Nodes):** Maps coordinates to indices $0 \dots 32^3 - 1$.
    ```python
    def upper_off(i, j, k):
        # Helper to extract top 5 bits of 12-bit coordinate (0-4095 range)
        # Based on source: (n & 4095) >> 7
        a = lambda n: (n & 4095) >> 7
        return (a(i) << 10) | (a(j) << 5) | a(k)
    ```

**Parameters:**
*   **Leaf Node Size:** $8^3 = 512$ voxels [S1].
*   **Bitmask Size:** 8 x 64-bit words [S1].
*   **Tree Depth:** 4 levels (top-down fanout factors: 32, 16, 8) [S1].

**Edge Cases and Pitfalls:**
*   **Inactive Voxels:** Always return 0. This is a unique background index [S1].
*   **Precision:** The implementation relies on bitwise operations and intrinsic functions (e.g., `countOn`) for speed. Care must be taken with bit shifts and masks to ensure correct alignment with the 64-bit words [S1].
*   **Memory Layout:** The sidecar arrays must be allocated separately and accessed via the computed indices. The IndexGrid itself does not hold the data [S1].

## Subtopics

### GridBatch
A class in fVDB representing a set of NanoVDB index grids (one per item in a batch), stored as a contiguous block of IndexGrids with metadata for quick access [S1]. Internally, it is a contiguous block of NanoVDB IndexGrids stored one after the other [S1]. It is used to represent a minibatch of sparse voxel grids in fVDB [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ amortized reads and writes due to the fixed-depth, shallow VDB tree [S1].
*   **Memory Complexity:** The leaf node size is 80 bytes, compared to >4KB in standard NanoVDB LeafNode, achieving a >50x memory reduction [S1].
*   **Performance:** Designed for bandwidth-limited GPU computations by limiting random-access operations and maximizing data reuse [S1].
*   **Strengths:** Preserves spatial coherence (unlike hash tables), supports arbitrary data types and dimensions via sidecars (unlike standard VDB), and enables efficient ray marching (unlike hash tables) [S1].
*   **Limitations:** Requires external management of value arrays (sidecars) [S1].
*   **When to Use:** Deep learning tasks involving sparse 3D tensors, such as convolution, pooling, attention, ray-tracing, and meshing [S1].
*   **When Not to Use:** When dense data storage is required or when dynamic sparsity patterns require frequent structural changes without careful construction optimization [S1].

## Background
IndexGrid was introduced in the fVDB framework by Williams et al. (2024) [S1]. It is derived from NanoVDB [Museth 2021] [S1]. The motivation was to address the limitations of previous sparse 3D learning frameworks that relied on hash tables (lacking spatial coherence) or octrees (suffering from $O(\log n)$ access and deep trees) [S1]. Standard VDB encodes data and topology together, requiring template specializations and being memory inefficient for shared topology [S1]. IndexGrid separates these concerns, allowing a single topology to be reused with multiple data types [S1].

## Sources
[S1] fVDB: A Deep-Learning Framework for Sparse, Large-Scale, and High-Performance Spatial Intelligence, https://research.nvidia.com/labs/prl/publication/williams2024fvdb/, fetched 2026-10-30
