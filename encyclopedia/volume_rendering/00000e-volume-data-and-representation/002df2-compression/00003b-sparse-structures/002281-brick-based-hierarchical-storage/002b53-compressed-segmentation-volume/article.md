# Compressed Segmentation Volume

Also called: CSVol, CSVE, CSVD

## Overview
Compressed Segmentation Volume (CSV) is a lossless compression technique for segmentation volumes that encodes individual bricks using a multi-resolution label pyramid and entropy coding, enabling fast decompression and GPU-based rendering [S1][S2, unofficial]. It builds on [Brick-based Hierarchical Storage](2793-brick-based-hierarchical-storage.md) by replacing static label storage with an operation stream (Parent Reuse, Neighbor Reuse, Palette Advance, Palette Back Reference) compressed via rANS [S1]. Unlike [ExaBricks Hierarchy](2182-exabricks-hierarchy.md) which focuses on sparse page tables, CSV optimizes for high compression ratios (1–3% of original size) and parallel brick-wise decompression [S1].

## How it works
**Inputs:** A 3D segmentation volume (array of `uint32_t` labels) and a brick size $b$ (power of two, typically 16, 32, or 64) [S1][S2, unofficial].
**Outputs:** A compressed encoding array `m_encoding` (sequence of `uint32_t`), a brick start index array `m_brick_starts`, and global rANS frequency tables [S2, unofficial].

**Data Structures:**
*   `MultiGridNode`: Struct with `uint32_t label` and `bool constant_subregion` [S2, unofficial].
*   `m_encoding`: `std::vector<uint32_t>` storing compressed operation streams for all bricks and LODs [S2, unofficial].
*   `m_brick_starts`: `std::vector<uint32_t>` storing the start index in `m_encoding` for each brick [S2, unofficial].
*   `ReadState`: Struct with `uint32_t idxE` (read offset), `uint32_t rans_state` (decoder state), `bool in_detail_lod` (flag for finest LOD) [S2, unofficial].

**Algorithm Steps:**

1.  **Multi-Resolution Pyramid Construction:** For each brick, build a resolution pyramid. Level $L_N$ (coarsest) has $1^3$ node; Level $L_0$ (finest) has $b^3$ nodes. Level $L_l$ stores $(b/2^l)^3$ labels. Each voxel in $L_l$ overlaps 8 voxels in $L_{l-1}$. The label of a node in $L_l$ is the most frequent label among its 8 children in $L_{l-1}$. If ambiguous, choose the first occurring label [S1][S2, unofficial]. Voxels outside the volume boundary are flagged with label `0xFFFFFFFF` and skipped during frequency calculation [S2, unofficial].
2.  **Operation Stream Generation:** Iterate levels $l$ from $N$ down to 1. For each node $i$ at level $l$ in Morton Z-curve order:
    *   If `pyramid[i].constantChildren` is true, skip (stop bit set at parent).
    *   Set `parent = pyramid[i].label`.
    *   Iterate child nodes $j$ at level $l-1$ in Z-order.
    *   Determine operation `op` for child label $L = \text{pyramid}[j].\text{label}$ using `bestOperation(parent, pyramid, palette, L)` [S1].
    *   `bestOperation` checks in fixed order: `Rp`, `Rx`, `Ry`, `Rz`, `P*0`, `P*δ`, `Pa` [S1].
    *   If `op` is `Pa`, push $L$ to palette. If `op` is `P*δ`, output $\delta$ (4-bit) [S1].
    *   Output tuple `(op, stop)` where `stop = pyramid[j].constantChildren` [S1].
3.  **Entropy Coding (rANS):** Compute two global frequency tables per dataset: one for interior nodes (levels $L_N..L_1$) and one for leaf nodes ($L_0$, where stop bits are always 0). Use a subsampling prepass (e.g., every 512th brick) to count operation frequencies. Normalize zero frequencies to 1 to prevent rANS errors [S1][S2, unofficial]. Encode the operation stream using rANS [S1].
4.  **Storage:** Store encoded bricks sequentially in `m_encoding`. Record start indices in `m_brick_starts` using Morton order mapping [S2, unofficial].

**Decoding:**
1.  Initialize `ip = 0` (palette index), `out` array for target LOD $t$. Set `out[0] = palette[0]` [S1].
2.  Iterate levels $l$ from $N$ down to $t+1$. For each decoded node $i$ at level $l$ (spacing $2^{l-t}$) in Z-order:
    *   If `out[lastChildOf(i)]` is not empty, continue (constant area) [S1].
    *   Set `parent = out[i]` [S1].
    *   Iterate child nodes $j$ at level $l-1$ (spacing $2^{l-t-1}$) in Z-order.
    *   Read `(op, stop)` from rANS stream [S1].
    *   Apply operation:
        *   `Rp`: `out[j] = parent`
        *   `R{x,y,z}`: `out[j] = neighbor value` (if neighbor not yet decoded, reference its parent) [S1][S2, unofficial].
        *   `Pa`: `out[j] = palette[++ip]`
        *   `P*0`: `out[j] = palette[ip]`
        *   `P*δ`: `out[j] = palette[ip - δ]`
    *   If `stop` is true, fill $j$'s entire sub-block with `out[j]` [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_brick(brick_voxels, brick_size):
    # 1. Build Pyramid
    levels = int(log2(brick_size)) + 1
    pyramid = []
    # Level 0 is original brick
    pyramid.append(brick_voxels) 
    
    for l in range(1, levels):
        prev_level = pyramid[-1]
        curr_size = brick_size // (2**l)
        curr_level = []
        for z in range(curr_size):
            for y in range(curr_size):
                for x in range(curr_size):
                    # Gather 8 children from prev_level
                    children = []
                    for dz in range(2):
                        for dy in range(2):
                            for dx in range(2):
                                cx = x * 2 + dx
                                cy = y * 2 + dy
                                cz = z * 2 + dz
                                if cz < len(prev_level): # Boundary check
                                    children.append(prev_level[cz][cy][cx])
                    
                    # Most frequent label
                    if not children:
                        label = 0xFFFFFFFF
                    else:
                        label = max(set(children), key=children.count)
                    
                    # Constant check
                    is_constant = all(c == label for c in children) if children else False
                    curr_level.append((label, is_constant))
        pyramid.append(curr_level)

    # 2. Generate Operation Stream
    palette = [pyramid[levels-1][0][0]] # Root label
    operations = []
    
    for l in range(levels-1, 0, -1):
        level_nodes = pyramid[l]
        child_size = brick_size // (2**(l-1))
        
        for idx, (label, is_const) in enumerate(level_nodes):
            if is_const:
                continue
            
            parent_label = label
            # Iterate children in Z-order
            for child_idx in range(len(level_nodes) * 8): # Simplified iteration
                # Get child label and constant status from finer level
                # Note: Actual implementation maps 3D coords to 1D index
                child_label, child_const = get_child_label_and_const(pyramid[l-1], child_idx)
                
                op = best_operation(parent_label, child_label, palette)
                
                if op == 'Pa':
                    palette.append(child_label)
                elif op == 'P_delta':
                    # delta is 4-bit value
                    pass 
                
                operations.append((op, child_const))
                
    return operations, palette

def best_operation(parent, child_label, palette):
    if child_label == parent:
        return 'Rp'
    # Check neighbors (simplified)
    # ...
    if child_label == palette[-1]:
        return 'P_0'
    # Check P_delta
    # ...
    return 'Pa'

def decompress_brick(operations, palette, target_lod, brick_size):
    levels = int(log2(brick_size)) + 1
    target_level = levels - 1 - target_lod
    out_size = brick_size // (2**target_lod)
    out = [[None for _ in range(out_size)] for _ in range(out_size * out_size)] # Flattened 3D
    
    ip = 0
    out[0] = palette[0]
    
    for l in range(levels-1, target_lod, -1):
        spacing = 2**(l - target_lod)
        # Iterate decoded nodes
        for i in range(len(out)):
            if out[i] is not None and is_constant_area(out, i, l, target_lod):
                continue
            
            parent = out[i]
            # Iterate children
            for j in range(8):
                child_idx = get_child_index(i, j, l, target_lod)
                op, stop = read_next_operation(operations)
                
                if op == 'Rp':
                    out[child_idx] = parent
                elif op == 'Pa':
                    ip += 1
                    out[child_idx] = palette[ip]
                # ... other ops
                
                if stop:
                    fill_subblock(out, child_idx, l, target_lod, out[child_idx])
                    
    return out
```

## Subtopics

### [Multi-resolution Brick Encoding](5775-multi-resolution-brick-encoding.md)
Multi-resolution Brick Encoding is a lossless compression technique for segmentation volumes that exploits homogeneous label neighborhoods to compactly encode label data and boundaries. It builds a per-brick resolution pyramid to generate a sequence of operations and a label palette, forming the foundational representation for Compressed Segmentation Volume (CSV) compression. Unlike flat storage, it separates the label values from the structural assignment logic, enabling efficient entropy coding.

### Segmented Dataset Quantization
A method for segmented datasets where small blocks are quantized to have only one or two different components inside. Encoded data consists of the block type and the splitting plane which subdivides the block [S3].

## Operating characteristics
**Time and Memory Complexity:**
*   Compression: $O(N)$ where $N$ is the number of voxels. Parallelization uses $T$ threads for $T$ bricks [S1].
*   Decompression: $O(N)$ per brick. GPU-based decompression allows on-the-fly rendering [S1].
*   Memory: Temporary memory for pyramid generation is ~14% of brick size [S1]. Output array size depends on target LOD [S1].

**Performance:**
*   Compression Ratio (CR): Achieves 1% to 3% of original data size [S1].
    *   CELLS (1M labels): 2.805% (b=64, rANS) [S1].
    *   FIBER (32K labels): 0.899% (b=64, rANS) [S1].
    *   CORTEX (15M labels): 1.590% (b=64, rANS) [S1] (on 1024^3 subvolume).
*   Throughput: 1.5 to 3 GB/s for compression and decompression [S1].
*   rANS improves CR by ~40% over 4-bit nibble storage, with ~2.5% throughput impact [S1].

**Strengths and Limitations:**
*   Strengths: High compression, fast parallel (de)compression, supports LOD, GPU-friendly [S1].
*   Limitations: Neighbor references cannot cross brick borders, reducing efficiency for large homogeneous regions spanning bricks [S1]. Naive parallelization leads to load imbalance for large bricks [S1].

## Background
Originated in "Fast Compressed Segmentation Volumes for Scientific Visualization" (2023) [S1]. Developed to address storage and transfer challenges in voxel-based segmentation volumes (e.g., connectomics, materials science) [S1]. Uses rANS entropy coding as a compromise between Huffman and arithmetic coding [S1].

## Sources
[S1] Fast Compressed Segmentation Volumes for Scientific Visualization., https://doi.org/10.1109/tvcg.2023.3326573, fetched 2026-10-26
[S2] max-pio/compressed-segmentation-volumes, https://github.com/max-pio/compressed-segmentation-volumes, fetched 2026-10-26 (unofficial)
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
