# Random Access CSGV

Also called: CSGV-R, CSGV-R Random Access Decoding, Random Access Compressed Segmentation Volumes, Random Access Segmentation Volume Compression

## Overview
Random Access Compressed Segmentation Volumes (CSGV-R) is a lossless compression method for segmentation volumes that provides true per-voxel random access without brick-level caching [S1, unofficial]. It solves the memory overhead and latency issues inherent in standard Compressed Segmentation Volume (CSGV) decoding, which requires serial decoding of entire bricks to access individual voxels [S1, unofficial]. Unlike CSGV, which uses serial Asymmetric Numerical Systems (ANS) encoding, CSGV-R leverages Huffman-shaped wavelet trees to allow direct access to any node's operation and label lookup in constant time relative to the alphabet size, enabling rendering in the compression domain with optional voxel caching [S1, unofficial].

## How it works
**Inputs:** A compressed CSGV-R brick containing the wavelet matrix bit vector $W$, stop bit vector $S$, flat rank structure, palette of uncompressed labels (stored in reverse order), and LOD operation start indices $T_{\bar{l}}$ [S1, unofficial].
**Outputs:** The label value for a specific voxel identified by its Morton code $n$ and Level of Detail (LOD) $\bar{l}$ [S1, unofficial].

**Data Structures:**
1.  **Operation Stream ($T$):** Encoded implicitly via the Huffman-shaped wavelet matrix. The alphabet $\Sigma$ is constant with size $|\Sigma|=6$, consisting of operations: Parent ($\uparrow$), Neighbor X ($X$), Neighbor Y ($Y$), Neighbor Z ($Z$), Palette ($\mathcal{P}$), and Back Reference ($\delta=1$) [S1, unofficial]. The general back reference operation with distance $\delta > 1$ is omitted to reduce alphabet size [S1, unofficial].
2.  **Wavelet Matrix ($W$):** A bit vector encoding the operation stream. It is structured in levels $l=0 \dots 4$. The levels of the wavelet tree are distinct from the CSGV multi-resolution grid LODs; queries for all CSGV LODs start within the first wavelet tree level [S1, unofficial].
3.  **Stop Bit Vector ($S$):** A bit vector where $S[i]=1$ indicates that the region covered by node $i$ is missing in the operation stream on all finer levels [S1, unofficial].
4.  **LOD Starts ($T_{\bar{l}}$):** An array storing the operation stream index $i$ of the first grid node in each LOD $\bar{l}$ [S1, unofficial].
5.  **Palette:** A flat array of uncompressed uint32_t labels, stored in reverse order [S1, unofficial].

**Algorithm Steps:**

1.  **Compute Operation Stream Index:** Given a grid node's Morton code $n$ and LOD $\bar{l}$, compute the index $i$ in the operation stream $T$ using the function $idx(\mathbf{n}, \bar{l})$. This accounts for nodes removed by stop bits on coarser levels [S1, unofficial].
    $$i = idx(\mathbf{n}, \bar{l}) = T_{\bar{l}} + \mathbf{n} - \sum_{l < \bar{l}} rank_1^S(\mathbf{P}_l(\mathbf{n}, \bar{l})) \cdot 2^{3(\bar{l}-l)}$$
    Where:
    *   $T_{\bar{l}}$ is the stream index of the first grid node in LOD $\bar{l}$.
    *   $\mathbf{P}_l(\mathbf{n}, \bar{l})$ is the index of the (grand-)parent node of $n$ in LOD $l$.
    *   $rank_1^S(k)$ returns the number of 1 bits (stop bits) before index $k$ in vector $S$.
    *   The term $\mathbf{P}_l(\mathbf{n}, \bar{l})$ and the inner sum are computed iteratively to avoid recursion, running in $O(L)$ time where $L$ is the number of LOD levels [S1, unofficial].

2.  **Decode Operation:** Access the operation $\omega = T[i]$ from the wavelet matrix using Algorithm 2. The Huffman-shaped wavelet matrix uses Canonical Huffman Codes (CHC) where the operation number corresponds to the number of zeros before the first 1 bit [S1, unofficial].

3.  **Resolve Label:**
    *   If $\omega$ is a Palette operation ($\mathcal{P}$), the label is retrieved directly from the palette. The palette index is determined by a rank query on the operation stream for previous palette operations [S1, unofficial].
    *   If $\omega$ is a Parent ($\uparrow$) or Neighbor ($X, Y, Z$) operation, follow the indirection chain from the current grid node to the target node (parent or neighbor) until a Palette operation is reached. This involves recursively or iteratively computing the $idx$ for the target node and decoding its operation [S1, unofficial].
    *   If $\omega$ is a Back Reference ($\delta=1$), it refers to the immediately preceding node in the stream [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def access_wavelet_matrix(W, W_l_starts, Z_1, i):
    """
    Algorithm 2: Operation Stream Wavelet Matrix access.
    Input: operation stream index i, wavelet matrix bit vector W,
           W_l_starts: list of level starts W_l,
           Z_1: list where Z_1[l] = rank_W(W_l).
    Output: operation level l (0-4), where l=5 means 5 consecutive 0 bits.
    """
    for l in range(0, 5):
        # Access the bit at position W_l + i
        # access_W(pos) = (words[pos // 64] >> (pos & 63)) & 1
        bit = access_W(W[l] + i)
        
        if bit == 1:
            # Any 1 bit terminates the symbol
            # The level l corresponds to the number of 0 bits (CHC)
            return l
        
        # Count 1 bits before current position in this level
        ones_before = rank_W(W[l] + i) - Z_1[l]
        
        # Skip these 1 bits for the next level
        i = i - ones_before
        
    return 5 # Five consecutive 0 bits

def rank_wavelet_matrix(W, W_l_starts, Z_1, i, target_op_level):
    """
    Algorithm 3: Operation Stream Wavelet Matrix rank.
    Input: operation stream index i, wavelet matrix bit vector W,
           W_l_starts, Z_1.
    Output: Number of operations with level == target_op_level in T before i.
    """
    for l in range(0, 5):
        ones_before = rank_W(W[l] + i) - Z_1[l]
        i = i - ones_before
        
    # After traversal, i represents the number of 1 bits encountered
    # which corresponds to the count of symbols at the target level
    # Note: The source algorithm returns 'ones before' after the loop.
    # This logic assumes the target_op_level is implicitly handled by the traversal
    # or that the rank query is specific to the structure's final state.
    # Based on source, it returns the accumulated count.
    return ones_before

def get_label(n, l_bar, brick):
    """
    Retrieve label for voxel n at LOD l_bar.
    """
    # 1. Compute operation stream index
    i = idx(n, l_bar, brick)
    
    # 2. Decode operation
    op_level = access_wavelet_matrix(brick.W, brick.W_l_starts, brick.Z_1, i)
    
    # 3. Resolve label based on operation
    # op_level 0-4 correspond to specific operations (e.g., Parent, Neighbor, Palette)
    # The exact mapping of level to operation type depends on the Huffman code construction.
    # If it is a Palette operation, use rank query to find palette index.
    # If it is a Parent/Neighbor, compute idx for the target node and recurse.
    
    if is_palette_operation(op_level):
        # Rank query to find how many palette operations precede i
        palette_idx = rank_wavelet_matrix(brick.W, brick.W_l_starts, brick.Z_1, i, palette_op_level)
        return brick.palette[palette_idx]
    else:
        target_node = get_target_node(n, l_bar, op_level)
        return get_label(target_node.n, target_node.l, brick)

def idx(n, l_bar, brick):
    """
    Compute operation stream index for grid node n at LOD l_bar.
    """
    i = brick.T[l_bar] + n
    sum_offset = 0
    
    # Iterate over coarser levels l < l_bar
    for l in range(0, l_bar):
        # Compute parent index P_l(n, l_bar)
        # Parent of n in 3D is floor(n / 2^3) = floor(n / 8)
        # However, P_l is the index in LOD l.
        # The source states P_l(n, l_bar) is the index of the (grand-)parent.
        # For CSGV, the parent in the next coarser level is n // 8.
        # We need the index of this parent in LOD l.
        # This requires iterative computation or precomputed parent indices.
        
        # Simplified iterative approach for P_l:
        # P_l is effectively the Morton code of the parent, mapped to the stream index of LOD l.
        # But the formula uses P_l as an index into the stop bit vector S.
        # S is indexed by operation stream index.
        # So P_l(n, l_bar) must be the operation stream index of the parent node in LOD l.
        
        # Let's compute the parent's Morton code first.
        # In 3D, parent Morton code is n // 8.
        # But we need the index in LOD l.
        # The source says P_l(n, l_bar) is the index of the (grand-)parent node of n in LOD l.
        # This implies we need to map the spatial parent to the stream index of LOD l.
        
        # To avoid recursion, we can compute the parent's stream index iteratively.
        # However, the source formula uses P_l directly in rank_1^S.
        # rank_1^S expects an index into S, which is the operation stream index.
        
        # Let's assume we have a helper to get the stream index of a node at LOD l given its Morton code.
        # Or, we compute the parent's stream index from the current level up.
        
        # Actually, the source says: "P_l(n, l_bar) is the index of the (grand-)parent node of n in LOD l."
        # And "idx returns the operation stream index i... idx computes the negative offset...
        # because earlier nodes were removed by stop bits on coarser levels."
        
        # The term rank_1^S(P_l(n, l_bar)) counts stop bits before the parent's position in the stream.
        
        # Implementation detail:
        # We need the stream index of the parent of n at LOD l.
        # Let's denote stream_idx(m, l) as the stream index of node with Morton code m at LOD l.
        # stream_idx(m, l) = T[l] + m - sum_{k<l} rank_1^S(P_k(m, l)) * 2^{3(l-k)}
        
        # This is recursive. The source says it can be computed iteratively.
        
        # For the purpose of this article, we state the formula as given.
        # An implementer would compute P_l(n, l_bar) by traversing up the hierarchy.
        
        parent_morton = n // 8 # 3D parent
        # Get stream index of this parent at LOD l
        # This requires calling idx(parent_morton, l, brick)
        # But idx calls this formula.
        # The source says "P_l can be computed iteratively along with the inner sum".
        
        # Let's assume a function get_parent_stream_index(n, l_bar, l, brick) exists.
        p_idx = get_parent_stream_index(n, l_bar, l, brick)
        
        sum_offset += rank_1_S(brick.S, p_idx) * (2 ** (3 * (l_bar - l)))
        
    return i - sum_offset
```

**Parameters:**
*   Brick size $b = 64$ [S1, unofficial].
*   Back reference distance $\delta=1$ only [S1, unofficial].
*   Alphabet size $|\Sigma| = 6$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The $idx$ function involves sums of rank queries and powers of 2. Ensure integer arithmetic is used to avoid precision loss [S1, unofficial].
*   **Boundaries:** The Morton code of the parent of node $n$ is $\lfloor n/2^d \rfloor$ where $d=3$ in 3D (source mentions $d=2$ in 2D context, but CSGV is for 3D volumes) [S1, unofficial].
*   **Stop Bits:** Nodes marked with stop bits in $S$ are removed from the operation stream on finer levels. The $idx$ function correctly accounts for this by subtracting the offset caused by these removed nodes [S1, unofficial].

## Subtopics

### [Random Access Decompression (CSGV-R)](10178-random-access-decompression-csgv-r.md)
A rendering mode for CSGV-R that leverages true random access to decode voxel labels on demand. It offers three access strategies: direct "no cache", "voxel cache" with hash-based storage, and "voxel cache (es)" with an empty space bit vector. It solves memory overhead and latency issues of standard CSGV brick caching.

## Operating characteristics
*   **Time Complexity:** The $idx$ function runs in $O(L) = O(\log_b n)$ time, where $L$ is the number of LOD levels and $b$ is the branching factor [S1, unofficial]. Access and rank queries on the wavelet matrix are constant time given the constant alphabet size [S1, unofficial].
*   **Memory Complexity:** The compressed brick stores bit vectors $W$ and $S$ with a flat rank structure. The palette is stored in reverse order [S1, unofficial].
*   **Performance:** CSGV-R outperforms Neuroglancer with 2× to 4× smaller data sets at rates between 0.648% and 4.411% of the original volume sizes [S1, unofficial]. It has slightly lower compression rates than CSGV but faster render times [S1, unofficial].
*   **Strengths:** True per-voxel random access without brick-level caching; GPU-friendly due to elimination of serial ANS decoding; no decompression artifacts from caching [S1, unofficial].
*   **Limitations:** Slightly lower compression rates compared to serial CSGV; requires implementation of Huffman-shaped wavelet trees [S1, unofficial].
*   **When to use:** Real-time visualization of large segmentation volumes on GPUs where random access to individual voxels is required without high memory overhead for brick caching [S1, unofficial].
*   **When not to use:** When maximum compression ratio is the primary goal and serial access patterns are dominant (use standard CSGV) [S1, unofficial].

## Background
CSGV-R was introduced by the authors of the source paper [S1, unofficial] as an extension of Fast Compressed Segmentation Volumes (CSGV) [PD24]. CSGV provided strong brick-wise compression and brick-level random access but required serial decoding within bricks. CSGV-R adapts Huffman-shaped wavelet trees, previously used in text indexing, to the rendering context for the first time, enabling true multi-resolution random access on a per-voxel level [S1, unofficial]. It is used in domains such as connectomics, machine learning, and natural sciences for visualizing large-scale segmentation volumes [S1, unofficial].

## Sources
[S1] Random Access Segmentation Volume Compression for Interactive Volume Rendering, https://doi.org/10.1111/cgf.70116, fetched 2026-10-28 (unofficial)
