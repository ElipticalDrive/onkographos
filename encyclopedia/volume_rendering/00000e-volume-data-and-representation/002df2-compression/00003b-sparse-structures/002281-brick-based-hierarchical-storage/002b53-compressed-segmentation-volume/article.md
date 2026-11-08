# Compressed Segmentation Volume

Also called: Brick-based Iterative Refinement, CSVD, CSVE, CSVol, Fast Compressed Segmentation Volumes, Iterative Refinement Compression for Segmentation Volumes

## Overview
Compressed Segmentation Volume (CSV) is a lossless compression technique for voxel-based segmentation volumes that encodes individual bricks using a multi-resolution label pyramid and an iterative refinement scheme [S1]. It solves the storage and transfer bottlenecks of high-label-count segmentation data by compactly encoding label regions and boundaries via a sequence of operations (Parent Reuse, Neighbor Reuse, Palette Advance, Palette Back Reference) and stop bits, which are then entropy-coded using range Asymmetric Numeral Systems (rANS) [S1][S2, unofficial]. Unlike flat storage or general-purpose compressors, CSV separates label values from structural assignment logic, enabling efficient entropy coding with global frequency tables and supporting adaptive level-of-detail (LOD) rendering [S1]. It achieves compression ratios of 1% to 3% of the original data size with high throughput (up to gigabytes per second) [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A segmentation volume (array of `uint32_t` labels) partitioned into cubic bricks of size $b \times b \times b$ (where $b$ is a power of two, typically 16, 32, or 64) [S1][S2, unofficial].
*   **Output:** For each brick, a palette of labels and a compressed bitstream of operations and stop bits. The global output consists of a concatenation of these streams (`m_encoding`) and a start-index array (`m_brick_starts`) [S2, unofficial].

**Data Structures:**
*   **MultiGrid:** A per-brick resolution pyramid. Level $L_l$ stores $(b/2^l)^3$ labels. Each voxel in $L_l$ overlaps $2^3$ voxels in $L_{l-1}$. The label of a voxel in $L_l$ is the most frequent label among its 8 children in $L_{l-1}$ [S1]. If a voxel lies outside the volume, its label is set to `0xFFFFFFFF` [S2, unofficial].
*   **MultiGridNode:** Contains `uint32_t label` and `bool constant_subregion` (stop bit) [S2, unofficial].
*   **Palette:** A list of unique labels encountered during encoding. Index $i_p$ tracks the last used entry [S1].
*   **Encoding Stream:** A sequence of 4-bit tuples `(op, stop)`. `op` is one of 7 operations; `stop` indicates if the current node's subtree is homogeneous [S1]. `P*δ` requires an additional 4-bit value for $\delta$ [S1].
*   **Frequency Tables:** Two global tables per dataset: one for interior nodes ($L_N \dots L_1$) and one for leaf nodes ($L_0$, where stop bits are always 0) [S1].

**Parameters:**
*   `brick_dim` ($b$): Side length of a brick. Default: 32 [S2, unofficial].
*   `use_rANS`: Boolean to enable rANS entropy coding. Default: `true` [S2, unofficial].
*   `use_detail_freq`: Boolean to use a separate frequency table for the finest LOD. Default: `true` [S2, unofficial].
*   `subsampling_factor`: Factor for computing frequency tables (e.g., every 512th brick). Default: 8 [S2, unofficial].

**Algorithm Steps:**

1.  **Brick Partitioning:** Divide the volume into $b \times b \times b$ bricks. Handle out-of-core processing if the volume exceeds RAM [S1].
2.  **Pyramid Construction (Per Brick):**
    *   Initialize $L_0$ with the brick's voxel labels.
    *   Iterate $l$ from 1 to $N = \log_2(b)$:
        *   For each node in $L_l$, gather its 8 children from $L_{l-1}$.
        *   Skip children with label `0xFFFFFFFF`.
        *   Find the most frequent label among valid children. If ambiguous, choose the one occurring first in the child array [S1].
        *   Set `constant_subregion` to `true` if all valid children have the same label and are constant [S2, unofficial].
3.  **Frequency Table Prepass (Global):**
    *   Process a subset of bricks (subsampling) to count operation frequencies [S1].
    *   Normalize frequencies: set any zero frequency to 1 to prevent rANS division by zero [S2, unofficial].
    *   Generate two tables: `freq_interior` and `freq_leaf` [S1].
4.  **Encoding (Per Brick):**
    *   Initialize palette with the root label ($L_N$). Set $i_p = 0$ [S1].
    *   Iterate levels $l$ from $N$ down to 1:
        *   Traverse nodes in Morton Z-curve order.
        *   If `pyramid[i].constantChildren` is true, skip [S1].
        *   Set `parent = pyramid[i].label` [S1].
        *   Iterate child nodes $j$ in Z-curve order:
            *   Get label $L = \text{pyramid}[j].\text{label}$ and stop bit $stop = \text{pyramid}[j].\text{constantChildren}$ [S1].
            *   Determine `op` using `bestOperation(parent, pyramid, palette, L)` with fixed priority: $R_p, R_x, R_y, R_z, P*0, P*\delta, P_a$ [S1].
            *   If `op` is $P_a$, push $L$ to palette and increment $i_p$ [S1].
            *   If `op` is $P*\delta$, output $\delta$ (4 bits) [S1].
            *   Output tuple `(op, stop)` (4 bits) [S1].
5.  **Entropy Coding:**
    *   Encode the operation stream using rANS with the precomputed frequency tables [S1].
    *   Store the resulting bitstream in `m_encoding` and record the start index in `m_brick_starts` [S2, unofficial].

**Decoding (Per Brick, Target LOD $t$):**
1.  Initialize palette read index $i_p = 0$. Initialize output array `out` of size $2^{3(N-t)}$ with empty markers [S1].
2.  Set `out[0] = palette[i_p]` [S1].
3.  Iterate levels $l$ from $N$ down to $t+1$:
    *   Traverse already decoded nodes in Morton Z-curve order with spacing $2^{l-t}$ [S1].
    *   If `out[lastChildOf(i)]` is not empty, continue (subtree already filled by stop bit) [S1].
    *   Set `parent = out[i]` [S1].
    *   Iterate child nodes $j$ with spacing $2^{l-t-1}$:
        *   Read `(op, stop)` from rANS stream [S1].
        *   Apply operation:
            *   $R_p$: `out[j] = parent`
            *   $R_{x,y,z}$: `out[j] = neighbor value` (if neighbor not yet decoded, reference its parent) [S1].
            *   $P_a$: `out[j] = palette[++i_p]`
            *   $P*0$: `out[j] = palette[i_p]`
            *   $P*\delta$: `out[j] = palette[i_p - \delta]`
        *   If `stop` is true, fill `out[j]`'s entire sub-block of size $2^{3(l-t-1)}$ with `out[j]`'s label [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_brick(brick_voxels, brick_dim):
    """
    Compress a single brick into a palette and operation stream.
    brick_voxels: list of uint32_t labels in Morton order.
    brick_dim: side length b (power of 2).
    Returns: (palette, operation_stream)
    """
    N = int(math.log2(brick_dim))
    # 1. Build Multi-Resolution Pyramid
    # pyramid[l][i] = (label, constant_subregion)
    pyramid = []
    
    # Level 0: Original voxels
    level_0 = []
    for label in brick_voxels:
        level_0.append((label, False)) # constant_subregion initially false
    pyramid.append(level_0)
    
    # Coarsen levels
    for l in range(1, N + 1):
        prev_level = pyramid[l-1]
        curr_level = []
        num_nodes = (brick_dim // (2**l)) ** 3
        children_per_node = 8
        stride = (brick_dim // (2**(l-1))) ** 3 // num_nodes
        
        for i in range(num_nodes):
            children = prev_level[i*children_per_node : (i+1)*children_per_node]
            valid_children = [c for c in children if c[0] != 0xFFFFFFFF]
            
            if not valid_children:
                label = 0xFFFFFFFF
                constant = True
            else:
                # Find most frequent label
                counts = {}
                for c in valid_children:
                    counts[c[0]] = counts.get(c[0], 0) + 1
                max_count = -1
                label = valid_children[0][0]
                for c in valid_children:
                    if counts[c[0]] > max_count:
                        max_count = counts[c[0]]
                        label = c[0]
                    elif counts[c[0]] == max_count:
                        # Ambiguity: choose first in array
                        pass 
                
                # Constant if all valid children have same label
                constant = all(c[0] == label for c in valid_children)
            
            curr_level.append((label, constant))
        pyramid.append(curr_level)

    # 2. Encode
    palette = [pyramid[N][0][0]] # Root label
    i_p = 0
    op_stream = []

    for l in range(N, 0, -1):
        level = pyramid[l]
        prev_level = pyramid[l-1]
        num_nodes = len(level)
        stride = (brick_dim // (2**(l-1))) ** 3 // num_nodes
        
        for i in range(num_nodes):
            label, constant = level[i]
            if constant:
                continue
            
            parent_label = label
            # Iterate children in Z-order
            for k in range(8):
                child_idx = i * 8 + k
                child_label, child_constant = prev_level[child_idx]
                
                # Determine best operation
                op = best_operation(parent_label, palette, i_p, child_label)
                
                if op == 'Pa':
                    palette.append(child_label)
                    i_p += 1
                    op_stream.append(('Pa', child_constant))
                elif op == 'P*delta':
                    delta = i_p - 1 - palette.index(child_label) # Simplified lookup
                    op_stream.append(('P*delta', child_constant, delta))
                else:
                    op_stream.append((op, child_constant))
                    
    return palette, op_stream

def best_operation(parent, palette, i_p, child_label):
    """
    Fixed priority: Rp, Rx, Ry, Rz, P*0, P*delta, Pa
    """
    if child_label == parent:
        return 'Rp'
    
    # Neighbor checks would require spatial context; simplified here
    # Assuming neighbor logic is handled by spatial traversal order
    # For this pseudocode, we assume neighbor reuse is detected by 
    # checking adjacent nodes in the output buffer during decoding,
    # but during encoding, we check if the child label matches a neighbor's 
    # known label. Since we build the pyramid, we can check neighbors in 
    # the same level or coarser levels.
    
    # Simplified: Check Palette Back References
    if child_label == palette[i_p]:
        return 'P*0'
    
    # Check P*delta (last 16 used entries)
    for delta in range(1, 16):
        idx = i_p - delta
        if idx >= 0 and palette[idx] == child_label:
            return 'P*delta'
            
    # Fallback to Palette Advance
    return 'Pa'

def decompress_brick(palette, op_stream, target_lod, brick_dim):
    """
    Decompress a brick to target LOD.
    """
    N = int(math.log2(brick_dim))
    t = N - target_lod # target_lod 0 is finest, N is coarsest
    # Output size for LOD t
    out_size = (brick_dim // (2**t)) ** 3
    out = [None] * out_size
    
    i_p = 0
    out[0] = palette[i_p]
    
    # Decode levels from coarsest (N) to target (t+1)
    for l in range(N, t, -1):
        # Nodes at level l have spacing 2^(l-t) in output array?
        # Actually, output array is sized for LOD t.
        # Level l nodes are spaced by 2^(l-t) in the final array?
        # No, the decoding is in-place.
        # Level N has 1 node. Level N-1 has 8 nodes.
        # We fill the array for LOD t.
        
        # Number of nodes at level l
        num_nodes_l = (brick_dim // (2**l)) ** 3
        # Number of nodes at level l-1
        num_nodes_l_minus_1 = (brick_dim // (2**(l-1))) ** 3
        
        # Stride in output array for level l nodes
        # Level t has stride 1. Level l has stride 2^(l-t)
        stride_l = 2**(l-t)
        stride_l_minus_1 = 2**(l-1-t)
        
        for i in range(num_nodes_l):
            # Check if subtree already filled
            last_child_idx = i * 8 + 7
            if out[last_child_idx * stride_l_minus_1] is not None:
                continue
                
            parent_val = out[i * stride_l]
            
            for k in range(8):
                child_idx = i * 8 + k
                child_pos = child_idx * stride_l_minus_1
                
                op, stop, *args = op_stream.pop(0)
                
                if op == 'Rp':
                    out[child_pos] = parent_val
                elif op == 'Pa':
                    i_p += 1
                    out[child_pos] = palette[i_p]
                elif op == 'P*0':
                    out[child_pos] = palette[i_p]
                elif op == 'P*delta':
                    delta = args[0]
                    out[child_pos] = palette[i_p - delta]
                elif op.startswith('R'):
                    # Neighbor lookup logic
                    out[child_pos] = get_neighbor_value(...) 
                
                if stop:
                    # Fill sub-block
                    sub_block_size = 2**(3*(l-1-t))
                    for s in range(sub_block_size):
                        out[child_pos + s] = out[child_pos]
                        
    return out
```

## Subtopics

### [Multi-resolution Brick Encoding](5775-multi-resolution-brick-encoding.md)
Multi-resolution Brick Encoding is a lossless compression technique for segmentation volumes that exploits homogeneous label neighborhoods to compactly encode label data and boundaries. It builds a per-brick resolution pyramid to generate a sequence of operations and a label palette, forming the foundational representation for Compressed Segmentation Volume (CSV) compression. Unlike flat storage, it separates the label values from the structural assignment logic, enabling efficient entropy coding.

### [Random Access CSGV](10177-random-access-csgv.md)
Random Access Compressed Segmentation Volumes (CSGV-R) is a lossless compression method for segmentation volumes that provides true per-voxel random access without brick-level caching. It solves the memory overhead and latency issues inherent in standard Compressed Segmentation Volume (CSGV) decoding, which requires serial decoding of entire bricks to access individual voxels. Unlike CSGV, which uses serial Asymmetric Numerical Systems (ANS) encoding, CSGV-R leverages Huffman-shaped wavelet trees to allow direct access to any node's operation and label lookup in constant time relative to the alphabet size, enabling rendering in the compression domain with optional voxel caching.

### Segmented Dataset Quantization
A method for segmented datasets where small blocks are quantized to have only one or two different components inside. Encoded data consists of the block type and the splitting plane which subdivides the block [S4].

## Operating characteristics
**Time and Memory Complexity:**
*   **Compression:** $O(V)$ where $V$ is the number of voxels. Parallelization is naive: $T$ threads compress $T$ bricks in parallel, followed by synchronization [S1].
*   **Decompression:** $O(V)$ per brick. In-place decoding overwrites coarser LODs until the target LOD is reached, increasing memory consumption but not strictly required for correctness [S3, unofficial].
*   **Memory Overhead:** Pyramid generation requires approximately 14% additional temporary memory per brick [S1].

**Measured Performance:**
*   **Throughput:** 1.5 to 3 GB/s for compression and decompression [S1][S3, unofficial].
*   **Compression Ratio (CR):** 1% to 3% of original data size [S1][S3, unofficial].
    *   CELLS (4.0GB): 2.805% (b=64, rANS) [S1].
    *   FIBER (5.7GB): 0.899% (b=64, rANS) [S1].
    *   CORTEX (927.7GB): 1.590% (b=64, rANS) [S1].
*   **Scaling:** Larger brick sizes (e.g., 64) yield better compression rates because neighbor references cannot be used across brick borders, and large homogeneous regions are more compactly encoded [S1]. However, larger bricks lead to less optimal CPU utilization due to divergent processing times in naive parallelization [S1].

**Strengths and Limitations:**
*   **Strengths:** High compression ratios for segmentation volumes; fast throughput; supports LOD rendering; out-of-core capability [S1][S3, unofficial].
*   **Limitations:** Neighbor references are limited to within bricks; naive parallelization scales poorly for large bricks; rANS requires global frequency tables which may be suboptimal for highly heterogeneous datasets [S1].

**When to Use:**
*   Use for GPU-based volume visualization with on-the-fly decompression and LOD rendering [S3, unofficial].
*   Use when storage/transfer of large segmentation volumes is a bottleneck [S1].
*   Do not use when per-voxel random access is required without caching (use CSGV-R instead) [10177-random-access-csgv.md].

## Background
Compressed Segmentation Volume (CSV) was introduced in 2023 by Pio et al. in "Fast Compressed Segmentation Volumes for Scientific Visualization" [S1]. It addresses the challenges of storing and visualizing voxel-based segmentation volumes with large numbers of labels, such as those from connectomics and computational biology [S1]. The method builds on the principles of brick-based hierarchical storage, using a multi-resolution pyramid and entropy coding to achieve high compression rates while maintaining rendering performance [S1][S2, unofficial]. It is used in GPU-based volume visualization with on-the-fly decompression [S3, unofficial].

## Sources
[S1] Fast Compressed Segmentation Volumes for Scientific Visualization., https://doi.org/10.1109/tvcg.2023.3326573, fetched 2026-10-26
[S2] max-pio/compressed-segmentation-volumes, https://github.com/max-pio/compressed-segmentation-volumes, fetched 2026-10-26 (unofficial)
[S3] (PDF) Lossless Compression of Volume Data - ResearchGate, https://www.researchgate.net/publication/221252510_Lossless_Compression_of_Volume_Data, fetched 2026-11-03 (unofficial)
[S4] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
