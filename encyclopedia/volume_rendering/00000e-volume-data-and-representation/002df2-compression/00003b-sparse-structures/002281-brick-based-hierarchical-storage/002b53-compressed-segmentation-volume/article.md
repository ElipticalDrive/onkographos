# Compressed Segmentation Volume

Also called: Brick-based Iterative Refinement, CSVD, CSVE, CSVol, Fast Compressed Segmentation Volumes, Iterative Refinement Compression for Segmentation Volumes

## Overview
Compressed Segmentation Volume (CSV) is a lossless compression technique for voxel-based segmentation volumes that encodes individual bricks using a multi-resolution label pyramid and entropy-coded operation sequences [S1]. It solves the problem of high storage and memory bandwidth requirements for large-scale segmentation data by exploiting local label homogeneity through parent, neighbor, and palette references [S1]. Unlike general-purpose compressors (e.g., HDF5, PNG) or other segmentation-specific methods (e.g., Compresso), CSV achieves significantly better compression ratios (1–3% of original size) while supporting efficient GPU-based on-the-fly decompression and adaptive level-of-detail (LOD) rendering [S1].

## How it works
**Inputs:** A segmentation volume (array of `uint32_t` labels) and a brick size $b$ (power of two, typically 16, 32, or 64) [S1][S2, unofficial].
**Outputs:** A compressed encoding array `m_encoding`, a brick start index array `m_brick_starts`, and global frequency tables for rANS decoding [S2, unofficial].

**Data Structures:**
*   `MultiGridNode`: Struct with `uint32_t label` and `bool constant_subregion` [S2, unofficial].
*   `m_encoding`: `std::vector<uint32_t>` storing compressed operation streams for all bricks and LODs [S2, unofficial].
*   `m_brick_starts`: `std::vector<uint32_t>` storing the start index in `m_encoding` for each brick [S2, unofficial].
*   `ReadState`: Struct with `uint32_t idxE` (read offset), `uint32_t rans_state` (rANS decoder state), and `bool in_detail_lod` [S2, unofficial].

**Algorithm Steps:**

1.  **Multi-Resolution Pyramid Construction:**
    For each brick, construct a resolution pyramid (octree) with $N = \log_2(b)$ levels. Level $L_l$ contains $(b/2^l)^3$ nodes. Level $L_0$ contains the original $b^3$ voxels [S1][S2, unofficial].
    *   For each node in $L_l$, gather its 8 children from $L_{l-1}$ [S1].
    *   Skip children with label `0xFFFFFFFF` (out-of-bounds) [S2, unofficial].
    *   Determine the node's label as the most frequent label among valid children. If ambiguous, choose the one occurring first in the child array [S1].
    *   Set `constant_subregion = true` if all valid children have the same label and are constant [S2, unofficial].
    *   This step requires $\approx 14\%$ additional temporary memory per brick [S1].

2.  **Operation Encoding (Per Brick):**
    Initialize palette with the root node's label ($L_N$). Set palette index $i_p = 0$.
    Iterate levels $l$ from $N$ down to 1. For each node $i$ at level $l$ in Morton Z-curve order:
    *   If `pyramid[i].constantChildren` is true, skip (stop bit handled implicitly or explicitly for children) [S1].
    *   Store `parent = pyramid[i].label`.
    *   Iterate child nodes $j$ of $i$ in Morton Z-curve order:
        *   Let $L = \text{pyramid}[j].\text{label}$ and $\text{stop} = \text{pyramid}[j].\text{constantChildren}$.
        *   Select operation `op` using `bestOperation(parent, pyramid, palette, L)` in fixed priority order:
            1.  **Parent Reuse ($R_p$):** If $L == \text{parent}$.
            2.  **Neighbor Reuse ($R_x, R_y, R_z$):** If $L$ matches a neighbor node adjacent to $j$ along axis $x, y,$ or $z$ pointing outside the current $2^3$ sibling block. If the referenced neighbor has a later Z-index (not yet decoded), reinterpret as a reference to the neighbor's parent [S1].
            3.  **Palette Back Reference $P^*0$:** If $L == \text{palette}[i_p]$.
            4.  **Palette Back Reference $P^*\delta$:** If $L == \text{palette}[i_p - \delta - 1]$ for some $\delta \in [0, 15]$.
            5.  **Palette Advance ($P_a$):** Otherwise.
        *   If `op` is $P_a$, append $L$ to palette and increment $i_p$.
        *   If `op` is $P^*\delta$, output $\delta$ as a 4-bit value.
        *   Output the tuple `(op, stop)` where `op` is encoded in 4 bits (7 operations + stop bit) [S1].

3.  **Entropy Coding (rANS):**
    *   Compute two global frequency tables per dataset via a prepass over a subsampled subset of bricks (e.g., every 512th or 4096th brick) [S1].
    *   Table 1: For interior nodes (levels $L_N \dots L_1$).
    *   Table 2: For leaf nodes (level $L_0$), where stop bits are always 0 [S1].
    *   Normalize frequencies: Set any zero frequency to 1 to prevent rANS division by zero [S2, unofficial].
    *   Encode the operation stream using range Asymmetric Numeral Systems (rANS) [S1].

4.  **Decoding (In-Place):**
    To decode a brick to target LOD $t$:
    *   Initialize output array `out` of size $2^{3(N-t)}$ with empty values. Set $i_p = 0$.
    *   Set `out[0] = palette[0]` [S1].
    *   Iterate levels $l$ from $N$ down to $t+1$:
        *   Iterate already decoded nodes $i$ at level $l$ (spacing $2^{l-t}$) in Z-order.
        *   If `out[lastChildOf(i)]` is not empty, continue (constant area) [S1].
        *   Set `parent = out[i]`.
        *   Iterate child nodes $j$ at level $l$ (spacing $2^{l-t-1}$) in Z-order:
            *   Read `(op, stop)` from rANS stream.
            *   Apply operation:
                *   $R_p$: `out[j] = parent`.
                *   $R_{x,y,z}$: `out[j] = neighbor value`. Handle boundary by rounding down to parent if neighbor index is out of bounds [S2, unofficial].
                *   $P_a$: `out[j] = palette[++i_p]`.
                *   $P^*0$: `out[j] = palette[i_p]`.
                *   $P^*\delta$: `out[j] = palette[i_p - \delta]`.
            *   If `stop` is true, fill the entire sub-block of $2^{3(l-t-1)}$ labels in `out` with `out[j]` [S1].
    *   Coarser LODs are overwritten in-place until target LOD $t$ is reached [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_brick(brick_voxels, brick_size):
    levels = int(log2(brick_size)) + 1
    pyramid = []
    pyramid.append(brick_voxels)  # L0
    
    for l in range(1, levels):
        prev_level = pyramid[l-1]
        curr_size = brick_size // (2**l)
        curr_level = []
        stride = 2**(l-1)
        for z in range(curr_size):
            for y in range(curr_size):
                for x in range(curr_size):
                    children = []
                    for dz in range(2):
                        for dy in range(2):
                            for dx in range(2):
                                cz = z*2 + dz
                                cy = y*2 + dy
                                cx = x*2 + dx
                                label = prev_level[cz * brick_size**(l-1) + cy * brick_size**(l-1) + cx]
                                if label != 0xFFFFFFFF:
                                    children.append(label)
                    
                    if not children:
                        label = 0xFFFFFFFF
                        constant = True
                    else:
                        counts = {}
                        for c in children:
                            counts[c] = counts.get(c, 0) + 1
                        max_count = -1
                        label = children[0]
                        for c in children:
                            if counts[c] > max_count:
                                max_count = counts[c]
                                label = c
                            elif counts[c] == max_count and c < label:
                                label = c
                        constant = all(c == label for c in children)
                    curr_level.append({'label': label, 'constant': constant})
        pyramid.append(curr_level)

    palette = [pyramid[levels-1][0]['label']]
    ip = 0
    operations = []
    
    for l in range(levels-1, 0, -1):
        level_nodes = pyramid[l]
        for i in range(len(level_nodes)):
            node = level_nodes[i]
            if node['constant']:
                continue
            parent_label = node['label']
            for j in range(len(pyramid[l-1])):
                child_node = pyramid[l-1][j]
                L = child_node['label']
                stop = child_node['constant']
                op = best_operation(parent_label, palette, ip, L)
                if op == 'Pa':
                    palette.append(L)
                    ip += 1
                elif op == 'P_delta':
                    delta = ip - 1 - palette.index(L)
                    operations.append(('P_delta', delta, stop))
                else:
                    operations.append((op, stop))
    return operations, palette

def best_operation(parent, palette, ip, L):
    if L == parent:
        return 'Rp'
    if L == palette[ip]:
        return 'P_0'
    for delta in range(16):
        idx = ip - delta - 1
        if idx >= 0 and L == palette[idx]:
            return 'P_delta'
    return 'Pa'

def decode_brick(operations, palette, target_lod, brick_size):
    levels = int(log2(brick_size)) + 1
    N = levels - 1
    out_size = 2**(3*(N - target_lod))
    out = [None] * out_size
    ip = 0
    out[0] = palette[0]
    
    for l in range(N, target_lod, -1):
        node_count = 2**(3*l)
        spacing = 2**(l - target_lod)
        for i in range(node_count):
            last_child_idx = (i * 8 + 7) * spacing
            if out[last_child_idx] is not None:
                continue
            parent_val = out[i * spacing]
            for j in range(8):
                child_idx = (i * 8 + j) * spacing
                op, stop = operations.pop(0)
                if op == 'Rp':
                    out[child_idx] = parent_val
                elif op.startswith('R_'):
                    out[child_idx] = parent_val
                elif op == 'Pa':
                    ip += 1
                    out[child_idx] = palette[ip]
                elif op == 'P_0':
                    out[child_idx] = palette[ip]
                elif op == 'P_delta':
                    delta = operations.pop(0)
                    out[child_idx] = palette[ip - delta]
                if stop:
                    sub_size = 2**(3*(l - target_lod - 1))
                    for k in range(sub_size):
                        out[child_idx + k] = out[child_idx]
    return out
```

**Edge Cases and Pitfalls:**
*   **Neighbor Ambiguity:** If a neighbor reference points to a node with a later Z-index (not yet decoded), it is reinterpreted as a reference to the neighbor's parent [S1].
*   **Zero Frequencies:** If subsampling misses operations, frequencies are clamped to 1 to prevent rANS undefined behavior [S2, unofficial].
*   **Boundary Handling:** Voxels outside the volume are labeled `0xFFFFFFFF` and skipped during pyramid coarsening [S2, unofficial]. Neighbor access at boundaries is rounded down to the parent element [S2, unofficial].
*   **Parallelization:** Naive parallelization (T threads for T bricks) leads to suboptimal CPU utilization for larger bricks due to divergent processing times [S1].

## Subtopics

### [Segmented Dataset Quantization](5376-segmented-dataset-quantization.md)
A compression technique for volumetric segmentation data that reduces storage by restricting small spatial blocks to contain only one or two distinct label values. It solves the problem of high memory bandwidth and storage requirements for large-scale segmented volumes by encoding the block type and the geometric splitting plane that separates the two components.

### [Multi-resolution Brick Encoding](5775-multi-resolution-brick-encoding.md)
Multi-resolution Brick Encoding is a lossless compression technique for segmentation volumes that exploits homogeneous label neighborhoods to compactly encode label data and boundaries. It builds a per-brick resolution pyramid to generate a sequence of operations and a label palette, forming the foundational representation for Compressed Segmentation Volume (CSV) compression. Unlike flat storage, it separates the label values from the structural assignment logic, enabling efficient entropy coding.

### [Random Access CSGV](10177-random-access-csgv.md)
Random Access Compressed Segmentation Volumes (CSGV-R) is a lossless compression method for segmentation volumes that provides true per-voxel random access without brick-level caching. It solves the memory overhead and latency issues inherent in standard Compressed Segmentation Volume (CSGV) decoding, which requires serial decoding of entire bricks to access individual voxels. Unlike CSGV, which uses serial Asymmetric Numerical Systems (ANS) encoding, CSGV-R leverages Huffman-shaped wavelet trees to allow direct access to any node's operation and label lookup in constant time relative to the alphabet size, enabling rendering in the compression domain with optional voxel caching.

## Operating characteristics
*   **Compression Ratio:** Achieves 1% to 3% of the original data set size [S1][S3, unofficial]. For example, CELLS (4.0GB) compresses to 2.805% with rANS (b=64) [S1].
*   **Throughput:** High throughput of 1.5 to 3 GB/s for both compression and decompression [S1][S3, unofficial].
*   **Memory:** Temporary memory overhead for pyramid generation is $\approx 14\%$ of brick size [S1]. In-place decoding overwrites coarser LODs, increasing memory consumption if not managed, but is not strictly mandatory for correctness [S3, unofficial].
*   **Scalability:** Supports out-of-core processing for datasets larger than RAM by processing bricks individually [S1].
*   **Strengths:** Superior compression ratios compared to HDF5, PNG, Compresso, and Neuroglancer [S1]. Supports GPU-based on-the-fly decompression and LOD rendering [S1].
*   **Limitations:** Neighbor references cannot cross brick borders, reducing efficiency for large homogeneous regions spanning bricks [S1]. Naive parallelization leads to load imbalance for large bricks [S1].

## Background
Compressed Segmentation Volume was introduced in "Fast Compressed Segmentation Volumes for Scientific Visualization" (2023) [S1]. It addresses the limitations of general-purpose compressors and earlier segmentation-specific methods like Compresso, which either lack LOD support or have inferior compression ratios for certain data types (e.g., highly anisotropic fibers) [S1]. The method builds on the principles of multi-resolution brick encoding and iterative refinement [S1].

## Sources
[S1] Fast Compressed Segmentation Volumes for Scientific Visualization., https://doi.org/10.1109/tvcg.2023.3326573, fetched 2026-10-26
[S2] max-pio/compressed-segmentation-volumes, https://github.com/max-pio/compressed-segmentation-volumes, fetched 2026-10-26 (unofficial)
[S3] (PDF) Lossless Compression of Volume Data - ResearchGate, https://www.researchgate.net/publication/221252510_Lossless_Compression_of_Volume_Data, fetched 2026-11-03 (unofficial)
